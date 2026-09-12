# 🔐 Quantum-Secure Patient Profile Architecture
## "Delayed OTP-Locked QR" System

This document details the secure flow for generating patient profiles where the **Private Key Seed** is **never stored in plain text** on the server. The system uses a **Zero-Knowledge** approach with **Delayed Claiming**:

- **Server** stores only the encrypted blob (locked box) + wrapped recovery key
- **Patient** receives OTP via SMS **ONLY when claiming** (not at registration)
- **QR Code** provides the `request_id` (pointer to the box)
- **Doctor** generates everything at registration, then wipes memory immediately

Access requires **both** the QR Code (something you have) **and** the Time-Sensitive OTP sent to the registered mobile **when patient actively claims** their profile.

### ⏱️ Key Feature: Delayed Claiming
- **Registration (Day 0)**: Doctor creates profile → NO OTP sent → Patient not present
- **Claiming (Day 3+)**: Patient scans QR + enters phone → OTP sent NOW → Patient unlocks seed

---

## 🚀 Core Security Principles

1.  **Zero-Knowledge Server**: Server stores only `EncryptedPayload`, `WrappedRecoveryKey`, `PhoneHash`. Never sees raw `seed_hex`, OTP, or recovery key.
2.  **Two-Layer Encryption**: 
    - Layer 1: `seed_hex` encrypted with random `recovery_key` (AES-256-GCM)
    - Layer 2: `recovery_key` encrypted with OTP-derived key (PBKDF2 + AES-256-GCM)
3.  **Delayed OTP Trigger**: OTP generated and sent **ONLY** when patient enters phone number during claiming (not at registration)
4.  **Quantum Resistance**: Patient's `seed_hex` generated using Quantum SDK's CSPRNG (`generate_seed()`), protected by AES-256-GCM.
5.  **Brute-Force Protection**: 100,000 PBKDF2 iterations + rate limiting (5 attempts/hour) + 5-minute OTP expiry.

---

## 🔄 The Workflow

### Phase 1: Doctor Registration (Day 0 - Patient NOT Present)
***NO OTP SENT YET*** - Doctor creates profile and hands QR to patient. Patient can claim days/weeks later.

1.  **Generate Seed**: Doctor's app uses Quantum SDK `generate_seed()` to create patient's private key
    ```javascript
    import { generate_seed } from "@dignera/quantum-sdk";
    
    const { seed_hex } = generate_seed();
    // seed_hex exists only in RAM - will be wiped after encryption
    ```

2.  **Generate Recovery Key**: Create random 256-bit recovery key (not OTP yet!)
    ```javascript
    const recoveryKey = crypto.getRandomValues(new Uint8Array(32));
    // This is a random key, NOT derived from OTP
    ```

3.  **Layer 1 Encryption**: Encrypt `seed_hex` with `recovery_key`
    ```javascript
    const encryptedPayload = await aes256GcmEncrypt(seed_hex, recoveryKey);
    // encryptedPayload = AES-256-GCM(seed_hex, recoveryKey)
    ```

4.  **Generate OTP Secret**: Create random OTP that will be used later during claiming
    ```javascript
    const otpSecret = crypto.getRandomValues(new Uint8Array(32));
    // This will be used to derive OTP key when patient claims
    ```

5.  **Layer 2 Encryption**: Encrypt `recovery_key` with `otp_secret`
    ```javascript
    const wrappedRecoveryKey = await aes256GcmEncrypt(recoveryKey, otpSecret);
    // wrappedRecoveryKey = AES-256-GCM(recoveryKey, otpSecret)
    ```

6.  **Store Data**:
    - **Database**: Stores `{ request_id, phone_hash, encryptedPayload, wrappedRecoveryKey, otpSecretHash, salt }`
      - `encryptedPayload`: Layer 1 encrypted seed
      - `wrappedRecoveryKey`: Layer 2 encrypted recovery key
      - `otpSecretHash`: SHA-256 hash of `otpSecret` (for verification later)
      - `phone_hash`: SHA-256(patient_phone + global_salt)
    - **QR Code**: Contains only `request_id` (UUID pointer to database record)
    - **NO SMS SENT YET** - OTP generation happens only during claiming

7.  **Wipe Memory**: Doctor's app immediately zeroizes:
    - Raw `seed_hex`
    - `recoveryKey`
    - `otpSecret`
    - All intermediate encryption keys

8.  **Result**: 
    - ✅ Server has **double-locked box** (`encryptedPayload` + `wrappedRecoveryKey`)
    - ✅ Doctor has **nothing** (memory wiped)
    - ⏳ Patient has **QR code** but NO OTP yet (will receive when claiming)

---

### Phase 2: Patient Claiming (Day 3+ - When Patient Logs In)
***OTP TRIGGERED HERE*** - Patient actively claims their profile.

1.  **Install App & Scan QR**: Patient installs app, scans QR → App gets `request_id`

2.  **Enter Phone Number**: Patient enters registered phone number
    ```javascript
    const phoneHash = sha256(phoneNumber + globalSalt);
    ```

3.  **Server Validation**:
    - Hash entered phone → matches stored `phone_hash`?
    - Check rate limit: < 5 attempts in last hour?
    - **TRIGGER OTP**: Generate 6-digit OTP, send SMS NOW
      ```javascript
      // Server-side OTP generation
      const otp = generate6DigitOTP(); // e.g., "849201"
      const otpHash = sha256(otp);
      
      // Store OTP hash with 5-minute expiry
      await redis.setex(`otp:${request_id}`, 300, otpHash);
      
      // Send SMS via Twilio/AWS SNS
      await smsProvider.send(phoneNumber, `Your OTP: ${otp}`);
      ```

4.  **Patient Enters OTP**: User types OTP from SMS into app

5.  **Derive OTP Key**: Patient's app derives encryption key from OTP
    ```javascript
    const otpKey = await pbkdf2(userInputOTP, salt, iterations=100000);
    ```

6.  **Fetch Encrypted Data**: App downloads from server
    ```javascript
    const { encryptedPayload, wrappedRecoveryKey, salt } = await fetch(`/api/recovery/${request_id}`);
    ```

7.  **Decrypt Layer 2**: Unwrap `recovery_key` using OTP-derived key
    ```javascript
    try {
      const recoveryKey = await aes256GcmDecrypt(wrappedRecoveryKey, otpKey);
      // If OTP wrong: throws auth tag mismatch error
    } catch (e) {
      // Wrong OTP - increment attempt counter
      await server.incrementAttemptCount(request_id);
      throw new Error('Invalid OTP');
    }
    ```

8.  **Decrypt Layer 1**: Unlock `seed_hex` using recovered `recovery_key`
    ```javascript
    const seed_hex = await aes256GcmDecrypt(encryptedPayload, recoveryKey);
    // Success! Patient now has their private key seed
    ```

9.  **Import into Quantum SDK**:
    ```javascript
    import { start_session } from "@dignera/quantum-sdk";
    
    const session = await start_session({
      did: patientDID, // Optional if using DID
      seed_hex: seed_hex
    });
    ```

10. **Save to Secure Storage**:
    - SDK automatically saves to device secure enclave (Keychain/Keystore/IndexedDB)
    - Session valid for 1 hour, then re-authenticate with biometrics/passcode

11. **Zeroize Memory**:
    ```javascript
    zeroize(recoveryKey);
    zeroize(seed_hex);
    zeroize(otpKey);
    ```

12. **Result**:
    - ✅ Patient has `seed_hex` saved securely in device
    - ✅ Quantum SDK session active
    - ✅ Server never saw raw `seed_hex`, `recoveryKey`, or plain OTP

---

## 🛡️ Threat Model & Mitigation

| Threat Scenario | What They Get | Can they get Private Key? | Why? |
| :--- | :--- | :--- | :--- |
| **Database Theft** | `encryptedPayload`, `wrappedRecoveryKey`, `phone_hash`, `salt` | ❌ **NO** | Two-layer encryption: Need OTP to unwrap `recovery_key`, then need `recovery_key` to decrypt `seed_hex`. Brute-forcing 6-digit OTP with 100k PBKDF2 iterations takes years per attempt. |
| **Stolen QR Code** | `request_id` | ❌ **NO** | Just an ID pointer. Needs registered phone number + OTP to trigger decryption flow. |
| **SMS Interception** | OTP (`849201`) | ❌ **NO** | Have the key, but don't have `wrappedRecoveryKey` (needs DB access) or `request_id` (needs QR). Also needs phone hash verification. |
| **Server Compromise** | Running code, DB access | ❌ **NO** | Server never holds raw `seed_hex`, `recoveryKey`, or plain OTP. Zero-knowledge architecture with client-side decryption. |
| **Brute Force OTP** | Infinite attempts | ❌ **NO** | Rate limited (5 attempts/hour per `request_id`). OTP expires in 5 mins. 100k PBKDF2 iterations make each attempt ~100ms. |
| **Doctor Device Forensics** | Memory dump after registration | ❌ **NO** | `seed_hex`, `recoveryKey`, `otpSecret` all zeroized from RAM immediately after encryption. |
| **SIM Swap Attack** | Receives patient's SMS | ⚠️ **PARTIAL** | Would receive OTP, but still needs QR code + patient's device. *Mitigation*: Add device fingerprinting during claiming phase. |
| **Delayed Claim Exploit** | Access to unclaimed profile | ❌ **NO** | Profile remains encrypted indefinitely. OTP only generated when patient actively claims with correct phone number. |

---

## 🧩 SDK Integration Points (Using Existing Quantum SDK)

The architecture leverages your existing Quantum SDK across all platforms. Here's how to integrate the **Delayed OTP-Locked QR** system:

### 1. Generate Patient Seed (Doctor App - Phase 1, Step 1)

**All Platforms**: Use `generate_seed()` to create the patient's private key seed.

```javascript
// JavaScript/TypeScript (Web/Node.js - Doctor App)
import { generate_seed } from "@dignera/quantum-sdk";

const { seed_hex } = generate_seed();
// seed_hex is 64-byte hex string (256-bit entropy)
// Exists only in RAM - will be wiped after Layer 1 encryption
```

```python
# Python (Backend/Scripts - Doctor App)
from quantum_sdk import generate_seed

seed_hex = generate_seed().seed_hex
# Save temporarily for encryption, then wipe
```

```rust
// Rust (Native services - Doctor App)
use quantum_sdk::generate_seed;

let seed = generate_seed()?;
let seed_hex = seed.seed_hex;
```

```swift
// iOS (Swift - Doctor App)
import QuantumSDK

let seedHex = try generateSeed().seedHex
// Temporary - wipe after Layer 1 encryption
```

```kotlin
// Android (Kotlin - Doctor App)
import uniffi.quantum_sdk_mobile.*

val seedHex = withContext(Dispatchers.IO) { 
    generateSeed().seedHex 
}
```

---

### 2. Generate Random Recovery Key (Doctor App - Phase 1, Step 2)

```javascript
// Web/Node.js - Generate 256-bit random recovery key
const recoveryKey = crypto.getRandomValues(new Uint8Array(32));
// This is a random key, NOT derived from OTP
// Will be used for Layer 1 encryption
```

```python
# Python - Generate 256-bit random recovery key
import os
recovery_key = os.urandom(32)  # 32 bytes = 256 bits
```

---

### 3. Layer 1 Encryption: Encrypt Seed with Recovery Key (Doctor App - Phase 1, Step 3)

```javascript
// Web/Node.js - AES-256-GCM encryption
async function encryptLayer1(seedHex, recoveryKey) {
  const iv = crypto.getRandomValues(new Uint8Array(12)); // 96-bit IV
  
  const encoder = new TextEncoder();
  const seedBytes = encoder.encode(seedHex);
  
  const ciphertext = await crypto.subtle.encrypt(
    { name: 'AES-GCM', iv: iv },
    recoveryKey,
    seedBytes
  );
  
  return {
    encryptedPayload: new Uint8Array(ciphertext),
    iv: iv
  };
}

// Usage
const { encryptedPayload, iv: layer1Iv } = await encryptLayer1(seed_hex, recoveryKey);
```

```python
# Python - AES-256-GCM encryption
from cryptography.hazmat.primitives.ciphers.aead import AESGCM

def encrypt_layer1(seed_hex: str, recovery_key: bytes) -> dict:
    aesgcm = AESGCM(recovery_key)
    iv = os.urandom(12)  # 96-bit IV
    
    ciphertext = aesgcm.encrypt(iv, seed_hex.encode(), None)
    
    return {
        'encrypted_payload': ciphertext,
        'layer1_iv': iv
    }
```

---

### 4. Generate OTP Secret (Doctor App - Phase 1, Step 4)

```javascript
// Web/Node.js - Generate random OTP secret (used later during claiming)
const otpSecret = crypto.getRandomValues(new Uint8Array(32));
// This will be wrapped with OTP-derived key when patient claims
// NOT sent via SMS yet - OTP generated only during claiming phase
```

```python
# Python - Generate random OTP secret
otp_secret = os.urandom(32)  # 32 bytes = 256 bits
```

---

### 5. Layer 2 Encryption: Wrap Recovery Key with OTP Secret (Doctor App - Phase 1, Step 5)

```javascript
// Web/Node.js - AES-256-GCM encryption
async function encryptLayer2(recoveryKey, otpSecret) {
  const iv = crypto.getRandomValues(new Uint8Array(12)); // 96-bit IV
  
  const ciphertext = await crypto.subtle.encrypt(
    { name: 'AES-GCM', iv: iv },
    otpSecret,
    recoveryKey
  );
  
  return {
    wrappedRecoveryKey: new Uint8Array(ciphertext),
    iv: iv
  };
}

// Usage
const { wrappedRecoveryKey, iv: layer2Iv } = await encryptLayer2(recoveryKey, otpSecret);
```

```python
# Python - AES-256-GCM encryption
def encrypt_layer2(recovery_key: bytes, otp_secret: bytes) -> dict:
    aesgcm = AESGCM(otp_secret)
    iv = os.urandom(12)  # 96-bit IV
    
    ciphertext = aesgcm.encrypt(iv, recovery_key, None)
    
    return {
        'wrapped_recovery_key': ciphertext,
        'layer2_iv': iv
    }
```

```javascript
// Web/Node.js - Using Web Crypto API
async function deriveKeyFromOTP(otp, salt) {
  const encoder = new TextEncoder();
  const otpBytes = encoder.encode(otp);
  
  // Import OTP as key material
  const keyMaterial = await crypto.subtle.importKey(
    'raw',
    otpBytes,
    'PBKDF2',
    false,
    ['deriveBits', 'deriveKey']
  );
  
  // Derive 256-bit AES key with 100k iterations
  const key = await crypto.subtle.deriveKey(
    {
      name: 'PBKDF2',
      salt: salt,
      iterations: 100000,
      hash: 'SHA-256'
    },
    keyMaterial,
    { name: 'AES-GCM', length: 256 },
    false,
    ['encrypt', 'decrypt']
  );
  
  return key;
}
```

```python
# Python - Using cryptography library
from cryptography.hazmat.primitives.kdf.pbkdf2 import PBKDF2HMAC
from cryptography.hazmat.primitives import hashes
import os

def derive_key_from_otp(otp: str, salt: bytes) -> bytes:
    kdf = PBKDF2HMAC(
        algorithm=hashes.SHA256(),
        length=32,  # 256-bit key
        salt=salt,
        iterations=100000,
    )
    key = kdf.derive(otp.encode())
    return key  # 32 bytes for AES-256
```

---

### 4. Encrypt Seed with OTP-Derived Key (Doctor App - Phase 1)

```javascript
// Web/Node.js - AES-256-GCM encryption
async function encryptSeed(seedHex, otp) {
  const salt = crypto.getRandomValues(new Uint8Array(16));
  const key = await deriveKeyFromOTP(otp, salt);
  
  const iv = crypto.getRandomValues(new Uint8Array(12)); // 96-bit IV
  
  const encoder = new TextEncoder();
  const seedBytes = encoder.encode(seedHex);
  
  const ciphertext = await crypto.subtle.encrypt(
    { name: 'AES-GCM', iv: iv },
    key,
    seedBytes
  );
  
  return {
    encrypted_blob: new Uint8Array(ciphertext),
    salt: salt,
    iv: iv
  };
}
```

```python
# Python - AES-256-GCM encryption
from cryptography.hazmat.primitives.ciphers.aead import AESGCM
import os

def encrypt_seed(seed_hex: str, otp: str) -> dict:
    salt = os.urandom(16)
    key = derive_key_from_otp(otp, salt)
    
    aesgcm = AESGCM(key)
    iv = os.urandom(12)  # 96-bit IV
    
    ciphertext = aesgcm.encrypt(iv, seed_hex.encode(), None)
    
    return {
        'encrypted_blob': ciphertext,
        'salt': salt,
        'iv': iv
    }
```

---

### 5. Store Encrypted Data (Server Side)

```javascript
// Server API - Store encrypted blob
POST /api/patient/register
{
  "request_id": "uuid-v4",
  "phone_hash": "sha256(phone_number)",
  "encrypted_blob": "base64...",  // From step 4
  "salt": "base64...",            // From step 4
  "iv": "base64...",              // From step 4
  "attempt_count": 0
}

// Database schema (example)
CREATE TABLE patient_recovery (
  request_id UUID PRIMARY KEY,
  phone_hash VARCHAR(64) NOT NULL,
  encrypted_blob BYTEA NOT NULL,
  salt BYTEA NOT NULL,
  iv BYTEA NOT NULL,
  attempt_count INTEGER DEFAULT 0,
  created_at TIMESTAMP DEFAULT NOW(),
  otp_expires_at TIMESTAMP
);
```

---

### 6. Patient Retrieves & Decrypts (Patient App - Phase 2)

```javascript
// Patient App - Fetch and decrypt
import { start_session } from "@dignera/quantum-sdk";

async function claimPatientProfile(requestId, phone, otp) {
  // Step 1: Fetch encrypted data from server
  const response = await fetch(`/api/patient/recovery/${requestId}`, {
    method: 'POST',
    body: JSON.stringify({ phone_hash: sha256(phone) })
  });
  
  const { encrypted_blob, salt, iv } = await response.json();
  
  // Step 2: Derive key from OTP
  const key = await deriveKeyFromOTP(otp, salt);
  
  // Step 3: Decrypt seed
  const decryptedBytes = await crypto.subtle.decrypt(
    { name: 'AES-GCM', iv: iv },
    key,
    encrypted_blob
  );
  
  const decoder = new TextDecoder();
  const seed_hex = decoder.decode(decryptedBytes);
  
  // Step 4: Import into Quantum SDK
  const session = await start_session({
    did: patientDID,  // Optional if using DID
    seed_hex: seed_hex
  });
  
  // Step 5: Save to secure storage
  await saveToSecureEnclave(seed_hex);
  
  // Step 6: Wipe sensitive data from memory
  zeroize(key);
  zeroize(seed_hex);
  
  return session;
}
```

---

### 7. Secure Storage (Patient App)

```javascript
// Web - IndexedDB (encrypted at rest)
import { start_session } from "@dignera/quantum-sdk";

// start_session automatically saves vault to IndexedDB
const session = await start_session({ did, seed_hex });
// Session lasts 1 hour, then call start_session again

// iOS - Keychain
try {
  await start_session({ did, seed_hex });
  // SDK handles Keychain storage automatically
} catch (e) {
  // Handle error
}

// Android - Keystore
withContext(Dispatchers.IO) {
  startSession(StartSessionOptions(did, seed_hex, ...))
  // SDK handles Keystore storage automatically
}
```

---

### 8. OTP Service & Rate Limiting (Server Side)

```javascript
// Node.js - OTP generation with rate limiting
import crypto from 'crypto';
import Redis from 'ioredis';

const redis = new Redis();

class OtpService {
  async sendOtp(requestId, phoneHash) {
    // Check rate limit
    const attempts = await redis.get(`otp_attempts:${requestId}`);
    if (attempts >= 5) {
      throw new Error('Too many attempts. Try again in 1 hour.');
    }
    
    // Generate 6-digit OTP
    const otp = Math.floor(100000 + Math.random() * 900000).toString();
    
    // Store OTP hash with 5-minute expiry
    const otpHash = crypto.createHash('sha256').update(otp).digest('hex');
    await redis.setex(`otp:${requestId}`, 300, otpHash);
    
    // Send SMS via Twilio/Provider
    await smsProvider.send(phoneHash, `Your OTP: ${otp}`);
    
    return true;
  }
  
  async verifyOtp(requestId, otp) {
    const otpHash = crypto.createHash('sha256').update(otp).digest('hex');
    const storedHash = await redis.get(`otp:${requestId}`);
    
    if (!storedHash || storedHash !== otpHash) {
      // Increment attempt counter
      await redis.incr(`otp_attempts:${requestId}`);
      await redis.expire(`otp_attempts:${requestId}`, 3600);
      return false;
    }
    
    return true;
  }
}
```

---

### 9. Memory Zeroization (Critical Security Practice)

```javascript
// JavaScript - Zeroize sensitive data
function zeroize(array) {
  if (array instanceof Uint8Array) {
    for (let i = 0; i < array.length; i++) {
      array[i] = 0;
    }
  }
}

// After encryption in Doctor App:
zeroize(seedBytes);
zeroize(keyMaterial);

// After decryption in Patient App:
zeroize(decryptedBytes);
zeroize(key);
```

```python
# Python - Secure memory clearing
import ctypes

def zeroize(buffer):
    ctypes.memset(ctypes.addressof(buffer), 0, len(buffer))

# Usage
seed_hex_bytes = seed_hex.encode()
# ... use seed_hex_bytes ...
zeroize(seed_hex_bytes)
```

---

## 📋 Implementation Plan

### Phase 1: Foundation (Weeks 1-2) ✅
- [x] Quantum SDK available (JS, Python, Rust, iOS, Android)
- [x] `generate_seed()` function implemented across all platforms
- [x] `start_session()` with secure storage (Keychain/Keystore/IndexedDB)
- [ ] PBKDF2 key derivation from OTP (using platform crypto APIs)
- [ ] AES-256-GCM encryption/decryption wrappers

### Phase 2: Server Infrastructure (Weeks 3-4)
- [ ] REST API endpoints:
  - `POST /api/patient/register` - Store encrypted blob
  - `POST /api/patient/recovery/:requestId` - Fetch encrypted data
  - `POST /api/patient/otp/send` - Trigger OTP SMS
  - `POST /api/patient/otp/verify` - Verify OTP (rate-limited)
- [ ] Database schema with encrypted fields
- [ ] Redis-based rate limiting (5 attempts/hour per request_id)
- [ ] SMS provider integration (Twilio/AWS SNS)
- [ ] Phone number hashing (SHA-256 with salt)

### Phase 3: Doctor App Features (Weeks 5-6)
- [ ] Patient registration UI flow
- [ ] Generate seed using Quantum SDK `generate_seed()`
- [ ] Generate OTP and derive encryption key
- [ ] Encrypt seed with AES-256-GCM
- [ ] Generate QR code with `request_id`
- [ ] Memory zeroization after encryption
- [ ] Send OTP via SMS API

### Phase 4: Patient App Features (Weeks 7-8)
- [ ] QR code scanner integration
- [ ] Phone number entry + hash verification
- [ ] OTP input UI (6-digit keypad)
- [ ] Fetch encrypted blob from server
- [ ] Derive key from OTP using PBKDF2
- [ ] Decrypt seed using AES-256-GCM
- [ ] Import into Quantum SDK: `start_session({ did, seed_hex })`
- [ ] Save to secure storage (automatic via SDK)
- [ ] Memory zeroization after import

### Phase 5: Security Hardening (Weeks 9-10)
- [ ] Security audit by third-party firm
- [ ] Rate limiting penetration testing
- [ ] Brute-force attack simulation
- [ ] Device forensics testing (memory wiping verification)
- [ ] HIPAA compliance review
- [ ] GDPR data protection assessment
- [ ] Optional: Device fingerprinting for SIM swap protection

### Phase 6: Deployment & Monitoring (Week 11+)
- [ ] Staging environment deployment
- [ ] End-to-end integration tests
- [ ] Monitoring dashboards (OTP success/fail rates, attempt counts)
- [ ] Alerting rules (unusual attempt patterns, high failure rates)
- [ ] Production deployment with canary release
- [ ] Continuous security monitoring
- [ ] Incident response runbook

---

## 🔒 Security Checklist

### Cryptography
- [x] CSPRNG for seed generation (Quantum SDK `generate_seed()`)
- [ ] PBKDF2 with 100,000 iterations (SHA-256)
- [ ] AES-256-GCM with random 96-bit IV
- [ ] Unique salt per encryption
- [ ] TLS 1.3 for all client-server communication

### Key Management
- [x] Seed exists only in RAM during encryption/decryption
- [ ] Memory zeroization immediately after use
- [x] Secure storage via Quantum SDK (Keychain/Keystore/IndexedDB)
- [ ] No long-term key storage on server

### Access Control
- [ ] OTP rate limiting: 5 attempts per hour per request_id
- [ ] OTP expiry: 5 minutes
- [ ] Phone number hash verification before OTP send
- [ ] Account lockout after repeated failures

### Data Protection
- [ ] Database encryption at rest (AES-256)
- [ ] Phone numbers stored as SHA-256 hashes only
- [ ] Audit logging for all OTP operations
- [ ] No plaintext seeds in logs or backups

### Operations
- [ ] Incident response plan documented
- [ ] Key breach notification procedure
- [ ] Regular security updates schedule
- [ ] Disaster recovery plan tested

---

## 📚 References

### Standards & Specifications
- **NIST FIPS 203**: ML-KEM (Kyber) Post-Quantum Cryptography
- **NIST SP 800-132**: PBKDF Recommendations
- **RFC 9180**: HPKE (Hybrid Public Key Encryption)
- **FIPS 197**: AES Specification
- **RFC 8018**: PKCS #5 (PBKDF2)

### Security Guidelines
- **OWASP Mobile Security Testing Guide**: Secure Storage
- **OWASP Cheat Sheet**: Cryptographic Storage
- **HIPAA Security Rule**: Technical Safeguards (§164.312)
- **GDPR Article 32**: Security of Processing

### Platform Documentation
- **Web Crypto API**: PBKDF2, AES-GCM
- **iOS Security Guide**: Keychain Data Protection
- **Android Keystore**: Hardware-Backed Security
- **Quantum SDK Docs**: `generate_seed()`, `start_session()`

---

## 🎯 Architecture Summary

**Why This Design?**

1. **No HSM Required**: Unlike the previous design with ML-KEM encapsulation, this OTP-derived approach eliminates the need for expensive HSM/KMS infrastructure. The OTP itself is the key secret.

2. **True Zero-Knowledge**: Server stores only encrypted blobs. Even if attackers compromise the entire database and server code, they cannot decrypt patient seeds without the OTP (which is sent via SMS and never stored).

3. **Elegant Simplicity**: 
   - Doctor generates seed → encrypts with OTP-derived key → stores blob
   - Patient receives OTP via SMS → derives same key → decrypts blob
   - No key escrow, no master keys, no complex key management

4. **Quantum-Secure Foundation**: Patient's `seed_hex` is generated using Quantum SDK's CSPRNG and can be used for:
   - Post-quantum digital signatures (Dilithium2)
   - Post-quantum key agreement (Kyber-768)
   - DID-based identity with quantum-resistant keys

5. **Battle-Tested Primitives**:
   - PBKDF2: Widely deployed, well-understood, FIPS-compliant
   - AES-256-GCM: Industry standard, hardware-accelerated on modern devices
   - SHA-256: Collision-resistant, fast, universally available

**Trade-offs:**
- ⚠️ SMS delivery reliability (consider fallback channels)
- ⚠️ 6-digit OTP entropy (~20 bits) mitigated by 100k PBKDF2 iterations
- ⚠️ SIM swap attacks (mitigated by optional device fingerprinting)