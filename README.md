# Quantum-Secure Patient Profile Architecture
## "OTP-Locked Key Recovery" System

This document details the secure flow for generating patient profiles where the **Private Key Seed** is never stored in plain text on the server. Access requires both the **QR Code** (something you have) and a **Time-Sensitive OTP** sent to the registered mobile (something you are/have).

---

## 🚀 Core Security Principles

1.  **Zero-Knowledge Server**: The server stores only encrypted blobs. It never sees the raw Private Key Seed.
2.  **Delayed Claiming**: Patients can claim their profile days/weeks after registration. OTP is triggered ONLY at claim time.
3.  **Quantum Resistance**: Uses ML-KEM-768 (Kyber) for encapsulation and AES-256-GCM for payload encryption.
4.  **Brute-Force Protection**: OTP attempts are rate-limited; decryption keys are derived using slow KDF (Argon2/PBKDF2).

---

## 🔄 The Workflow

### Phase 1: Doctor Registration (Day 0)
*No OTP sent yet. Patient is not present.*

1.  **Doctor App** generates a random `recovery_secret` (256-bit).
2.  **Doctor App** generates the Patient's `private_key_seed` using Quantum SDK (`generate_seed()`).
3.  **Encryption**:
    *   Encrypt `private_key_seed` using `recovery_secret` as the key (AES-256-GCM).
    *   Result: `encrypted_payload`.
4.  **Storage**:
    *   Server stores: `{ request_id, patient_phone_hash, encrypted_payload, salt }`.
    *   `recovery_secret` is **NOT** stored on the server yet (or stored wrapped by a master key if immediate persistence is needed, but logically separated).
    *   *Refined Approach*: To allow delayed claiming without doctor holding state, the server stores `encrypted_payload` (locked by `recovery_secret`). The `recovery_secret` itself is encrypted with a temporary key derived from the future OTP logic, OR simply:
    *   **Correct Flow for Delayed Claim**:
        1. Doctor generates `private_key_seed`.
        2. Doctor generates random `lock_key`.
        3. Encrypt `private_key_seed` with `lock_key` → `encrypted_payload`.
        4. Encrypt `lock_key` with `Master_Public_Key` (Server Side) → `wrapped_lock_key`.
        5. Store `{ request_id, phone_hash, encrypted_payload, wrapped_lock_key }`.
        6. Generate QR containing `request_id`.
5.  **Result**: Doctor hands QR to patient. Doctor's device wipes memory.

### Phase 2: Patient Claiming (Day 3+ / Login Time)
*OTP is triggered HERE.*

1.  **Patient** installs app and scans QR (`request_id`).
2.  **Patient** enters their registered mobile number.
3.  **Server Validation**:
    *   Hashes entered number → matches `patient_phone_hash`.
    *   **Trigger OTP**: Generates 6-digit random OTP, saves hash+expiry (5 mins), sends SMS.
4.  **Verification**:
    *   Patient enters OTP.
    *   Server verifies OTP hash.
5.  **Key Release**:
    *   If OTP valid: Server uses internal `Master_Private_Key` (in HSM/KMS) to unwrap `lock_key`.
    *   Server sends `lock_key` to Patient's app over TLS 1.3.
6.  **Local Decryption**:
    *   Patient's app uses `lock_key` to decrypt `encrypted_payload`.
    *   Result: `private_key_seed`.
7.  **Finalization**:
    *   App imports seed into Quantum SDK (`start_session`).
    *   App saves seed to Device Secure Enclave / Keychain.
    *   App deletes `lock_key` from memory.

---

## 🛡️ Threat Model & Mitigation

| Threat Scenario | Attacker Gets | Can they access Private Key? | Why? |
| :--- | :--- | :--- | :--- |
| **Database Theft** | `encrypted_payload`, `wrapped_lock_key`, `phone_hash` | ❌ **NO** | Cannot decrypt payload without `lock_key`. Cannot unwrap `lock_key` without Master Key (in HSM). |
| **Stolen QR Code** | `request_id` | ❌ **NO** | Needs registered phone number + OTP to trigger release. |
| **SIM Swap** | Receives OTP SMS | ❌ **NO** | (Optional) Requires Device Fingerprint matching original registration context. |
| **Server Compromise** | Running code, DB access | ❌ **NO** | Master Private Key is in HSM/KMS (AWS KMS, Azure Key Vault) requiring dual-control/approval. |
| **Brute Force OTP** | Infinite attempts | ❌ **NO** | Rate limited (5 attempts/hour). OTP expires in 5 mins. |

---

## 🧩 SDK Integration Points

### 1. Quantum Key Generation SDK

```typescript
// Doctor App - Phase 1
import { QuantumSDK } from '@quantum-health/sdk';

const quantum = new QuantumSDK({
  algorithm: 'ML-KEM-768',  // Kyber-768 for quantum resistance
  mode: 'key-generation'
});

// Generate cryptographically secure seed
const keyMaterial = await quantum.generateSeed({
  entropy: 256,  // bits
  format: 'raw'
});

// Returns: { seed: Uint8Array, seedId: string }
```

### 2. Encryption Module (AES-256-GCM)

```typescript
// Doctor App - Encrypt private_key_seed with lock_key
import { CryptoModule } from '@quantum-health/crypto';

const crypto = new CryptoModule();

const encryptedPayload = await crypto.encrypt({
  algorithm: 'AES-256-GCM',
  key: lockKey,           // Random 256-bit lock_key
  plaintext: keyMaterial.seed,
  options: {
    generateIV: true,     // Random 96-bit IV
    includeAuthTag: true  // 128-bit authentication tag
  }
});

// Returns: { ciphertext: Uint8Array, iv: Uint8Array, authTag: Uint8Array }
```

### 3. Key Encapsulation (ML-KEM-768)

```typescript
// Server - Wrap lock_key with Master Public Key
import { KemModule } from '@quantum-health/kem';

const kem = new KemModule({
  variant: 'ML-KEM-768',
  role: 'encapsulator'
});

// Load server's master public key (from HSM/KMS)
const masterPublicKey = await kem.loadPublicKey(masterPublicKeyPem);

const encapsulated = await kem.encapsulate({
  publicKey: masterPublicKey,
  plaintext: lockKey  // 256-bit lock_key
});

// Returns: { ciphertext: Uint8Array, sharedSecret: Uint8Array }
// Store ciphertext as wrapped_lock_key
```

### 4. Key Decapsulation (Patient Claiming)

```typescript
// Server - Unwrap lock_key using Master Private Key (in HSM)
import { KemModule } from '@quantum-health/kem';

const kem = new KemModule({
  variant: 'ML-KEM-768',
  role: 'decapsulator'
});

// HSM performs decapsulation - key never leaves HSM boundary
const unwrappedKey = await kem.decapsulate({
  privateKeyHandle: hsmMasterKeyHandle,  // Reference to key in HSM
  ciphertext: wrappedLockKey
});

// Returns: lockKey (Uint8Array) - transmitted over TLS 1.3 to patient app
```

### 5. Client-Side Decryption

```typescript
// Patient App - Decrypt encrypted_payload with received lock_key
import { CryptoModule } from '@quantum-health/crypto';

const crypto = new CryptoModule();

const decryptedSeed = await crypto.decrypt({
  algorithm: 'AES-256-GCM',
  key: lockKey,  // Received from server after OTP verification
  ciphertext: encryptedPayload.ciphertext,
  iv: encryptedPayload.iv,
  authTag: encryptedPayload.authTag
});

// Returns: private_key_seed (Uint8Array)
```

### 6. Secure Storage Integration

```typescript
// Patient App - Store seed in device secure enclave
import { SecureStorage } from '@quantum-health/storage';

const storage = new SecureStorage({
  platform: 'auto',  // Uses iOS Keychain / Android Keystore
  biometricRequired: true,
  invalidationPolicy: 'biometry_changed'
});

await storage.setItem({
  key: 'quantum_private_seed',
  value: decryptedSeed,
  metadata: {
    createdAt: Date.now(),
    algorithm: 'ML-KEM-768',
    version: '1.0'
  }
});

// Immediately clear from memory
crypto.zeroize(decryptedSeed);
crypto.zeroize(lockKey);
```

### 7. Session Initialization

```typescript
// Patient App - Start quantum-secure session
import { QuantumSDK } from '@quantum-health/sdk';

const quantum = new QuantumSDK({
  algorithm: 'ML-KEM-768',
  mode: 'session'
});

const session = await quantum.startSession({
  seed: await storage.getItem('quantum_private_seed'),
  peerPublicKey: serverPublicKey,
  options: {
    keyDerivation: 'HKDF-SHA256',
    sessionTimeout: 3600,  // seconds
    rotationPolicy: 'per-request'
  }
});

// Use session for all subsequent cryptographic operations
const encryptedRequest = await session.encrypt(patientData);
```

### 8. OTP Service Integration

```typescript
// Server - OTP Generation and Verification
import { OtpService } from '@quantum-health/auth';

const otpService = new OtpService({
  length: 6,
  expirySeconds: 300,  // 5 minutes
  maxAttempts: 5,
  lockoutDuration: 3600  // 1 hour after max attempts
});

// Trigger OTP
await otpService.sendOtp({
  requestId: patientRequestId,
  phoneHash: hashedPhoneNumber,
  channel: 'sms',
  provider: 'twilio'  // or other SMS provider
});

// Verify OTP
const isValid = await otpService.verify({
  requestId: patientRequestId,
  otp: userProvidedOtp
});

if (isValid) {
  // Proceed with key release
  const lockKey = await unwrapLockKey(requestId);
  return { success: true, lockKey };
}
```

### 9. Rate Limiting & Brute-Force Protection

```typescript
// Server - Rate limiter middleware
import { RateLimiter } from '@quantum-health/security';

const otpLimiter = new RateLimiter({
  windowMs: 3600000,  // 1 hour
  maxRequests: 5,     // 5 OTP attempts per hour
  keyGenerator: (req) => req.body.request_id
});

const kdfConfig = {
  algorithm: 'argon2id',
  memoryCost: 65536,   // 64 MB
  timeCost: 3,         // 3 iterations
  parallelism: 4,      // 4 threads
  outputLength: 32     // 256 bits
};
```

---

## 📋 Implementation Plan

### Phase 1: Foundation (Weeks 1-2)
- [ ] Set up Quantum SDK dependencies
- [ ] Implement ML-KEM-768 key generation
- [ ] Create AES-256-GCM encryption module
- [ ] Build secure random number generator

### Phase 2: Server Infrastructure (Weeks 3-4)
- [ ] Set up HSM/KMS integration (AWS KMS or Azure Key Vault)
- [ ] Implement master key pair generation
- [ ] Build database schema for encrypted patient records
- [ ] Create OTP service with rate limiting

### Phase 3: Doctor App Features (Weeks 5-6)
- [ ] Implement patient registration flow
- [ ] Integrate QR code generation
- [ ] Add lock_key encapsulation
- [ ] Build memory zeroization routines

### Phase 4: Patient App Features (Weeks 7-8)
- [ ] Implement QR code scanning
- [ ] Build phone number verification UI
- [ ] Integrate OTP input and validation
- [ ] Add secure storage (Keychain/Keystore)
- [ ] Implement session initialization

### Phase 5: Security Hardening (Weeks 9-10)
- [ ] Conduct security audit
- [ ] Implement additional rate limiting
- [ ] Add device fingerprinting (optional)
- [ ] Penetration testing
- [ ] Compliance review (HIPAA, GDPR)

### Phase 6: Deployment & Monitoring (Week 11+)
- [ ] Deploy to staging environment
- [ ] Run integration tests
- [ ] Set up monitoring and alerting
- [ ] Production deployment
- [ ] Continuous security monitoring

---

## 🔒 Security Checklist

- [ ] All keys generated using CSPRNG
- [ ] Master Private Key stored exclusively in HSM/KMS
- [ ] TLS 1.3 enforced for all communications
- [ ] Memory zeroization after key usage
- [ ] OTP rate limiting implemented
- [ ] Database fields encrypted at rest
- [ ] Audit logging enabled for all key operations
- [ ] Regular key rotation policy defined
- [ ] Incident response plan documented

---

## 📚 References

- **NIST FIPS 203**: ML-KEM Specification
- **RFC 9180**: HPKE (Hybrid Public Key Encryption)
- **OWASP Mobile Security**: Secure Storage Guidelines
- **HIPAA Security Rule**: Technical Safeguards
- **NIST SP 800-132**: PBKDF Recommendations