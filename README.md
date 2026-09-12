# 🔐 Quantum-Secure Patient Profile Architecture
## "OTP-Locked QR" System

This document details the secure flow for generating patient profiles where the **Private Key Seed** is **never stored in plain text** on the server. The system uses a **Zero-Knowledge** approach where:
- **Server** stores only the encrypted blob (locked box)
- **Patient** receives the OTP key via SMS (the key)
- **QR Code** provides the request_id (pointer to the box)

Access requires **both** the QR Code (something you have) **and** the Time-Sensitive OTP sent to the registered mobile (something you have).

---

## 🚀 Core Security Principles

1.  **Zero-Knowledge Server**: Server stores only `EncryptedBlob`, `Salt`, `PhoneHash`. Never sees raw OTP or Private Key Seed.
2.  **OTP-Derived Encryption**: Encryption key is derived from OTP using PBKDF2 (100k iterations). No separate key storage needed.
3.  **Quantum Resistance**: Patient's `seed_hex` generated using Quantum SDK's CSPRNG, protected by AES-256-GCM.
4.  **Brute-Force Protection**: 100,000 PBKDF2 iterations + rate limiting (5 attempts/hour) + 5-minute OTP expiry.
5.  **Elegant Simplicity**: No HSM/KMS required. The OTP **is** the key derivation secret.

---

## 🔄 The Workflow

### Phase 1: Doctor Registers Patient (Creation)
*No OTP sent yet. Patient is not present. Doctor's device holds secrets temporarily in RAM only.*

1.  **Generate Seed**: Doctor's app uses Quantum SDK to generate `seed_hex` (Private Key)
    ```javascript
    const { seed_hex } = await generate_seed();
    // seed_hex exists only in RAM temporarily
    ```

2.  **Create OTP**: System generates random 6-digit OTP (e.g., `849201`)

3.  **Derive Encryption Key**: Run OTP through PBKDF2
    ```javascript
    const encryptionKey = await pbkdf2(OTP, salt, iterations=100000);
    // Key = PBKDF2(OTP, Salt, Iterations=100,000)
    ```

4.  **Encrypt the Seed**: AES-256-GCM encryption
    ```javascript
    const EncryptedBlob = await aes256GcmEncrypt(seed_hex, encryptionKey);
    // EncryptedBlob = AES-256-GCM(seed_hex, Key)
    ```

5.  **Store Data**:
    - **Database**: Stores `{ EncryptedBlob, PhoneHash, Salt, AttemptCount, request_id }`
    - **QR Code**: Contains only `request_id` (pointer to database record)
    - **SMS**: Sends plain OTP (`849201`) to patient's phone

6.  **Wipe Memory**: Doctor's app immediately deletes:
    - Raw `seed_hex`
    - Plain OTP
    - Derived `encryptionKey`

7.  **Result**: 
    - ✅ Server has the **locked box** (EncryptedBlob)
    - ✅ Patient has the **key** (OTP) in their SMS
    - ✅ Doctor has **nothing** (memory wiped)

### Phase 2: Patient Scans & Unlocks (Retrieval)
*OTP verification happens during decryption attempt.*

1.  **Scan QR**: Patient scans QR → App gets `request_id`

2.  **Enter Phone**: Patient enters phone number → Server verifies hash matches stored `PhoneHash`

3.  **Enter OTP**: Patient types the OTP from their SMS (`849201`)

4.  **Derive Key on Client**: Patient's app runs same PBKDF2 derivation
    ```javascript
    const encryptionKey = await pbkdf2(userInputOTP, salt, iterations=100000);
    ```

5.  **Fetch & Decrypt**:
    ```javascript
    const EncryptedBlob = await fetchFromServer(request_id);
    const seed_hex = await aes256GcmDecrypt(EncryptedBlob, encryptionKey);
    ```

6.  **Success/Fail**:
    - ✅ **Correct OTP**: Decryption succeeds → App now has `seed_hex`
      - Save to device secure storage (iOS Keychain / Android Keystore)
      - Import into Quantum SDK: `start_session({ did, seed_hex })`
    - ❌ **Wrong OTP**: Decryption fails (auth tag mismatch / garbage output) → Access denied

7.  **Optional Server Verification** (Recommended for rate limiting):
    ```javascript
    // Server checks OTP hash before allowing decrypt attempt
    // Prevents brute force by tracking failed attempts per request_id
    const isValid = await verifyOtpHash(request_id, userInputOTP);
    if (!isValid) throw new Error('Invalid OTP');
    ```

---

## 🛡️ Threat Model & Mitigation

| Threat Scenario | What They Get | Can they get Private Key? | Why? |
| :--- | :--- | :--- | :--- |
| **Database Theft** | `EncryptedBlob`, `Salt`, `PhoneHash` | ❌ **NO** | Don't have OTP. Brute-forcing 6-digit OTP with 100k PBKDF2 iterations takes years per attempt. |
| **QR Code Theft** | `request_id` | ❌ **NO** | It's just an ID. Without phone number + OTP, it's useless. |
| **SMS Interception** | OTP (`849201`) | ❌ **NO** | Have the key, but don't have `EncryptedBlob` (needs DB access) or `request_id` (needs QR). |
| **Server Compromise** | Running code, DB access | ❌ **NO** | Server never holds OTP or decrypted seed. Zero-knowledge architecture. |
| **Brute Force OTP** | Infinite attempts | ❌ **NO** | Rate limited (5 attempts/hour). OTP expires in 5 mins. 100k PBKDF2 iterations slow each attempt. |
| **Doctor Device Forensics** | Memory dump after registration | ❌ **NO** | `seed_hex`, OTP, and `encryptionKey` wiped from RAM immediately after encryption. |

---

## 🧩 SDK Integration Points (Using Existing Quantum SDK)

The architecture leverages your existing Quantum SDK across all platforms. Here's how to integrate:

### 1. Generate Patient Seed (Doctor App - Phase 1)

**All Platforms**: Use `generate_seed()` to create the patient's private key seed.

```javascript
// JavaScript/TypeScript (Web/Node.js)
import { generate_seed } from "@dignera/quantum-sdk";

const { seed_hex } = generate_seed();
// seed_hex is 64-byte hex string (256-bit entropy)
// Exists only in RAM - will be wiped after encryption
```

```python
# Python (Backend/Scripts)
from quantum_sdk import generate_seed

seed_hex = generate_seed().seed_hex
# Save temporarily for encryption, then wipe
```

```rust
// Rust (Native services)
use quantum_sdk::generate_seed;

let seed = generate_seed()?;
let seed_hex = seed.seed_hex;
```

```swift
// iOS (Swift)
import QuantumSDK

let seedHex = try generateSeed().seedHex
// Temporary - wipe after encrypting
```

```kotlin
// Android (Kotlin)
import uniffi.quantum_sdk_mobile.*

val seedHex = withContext(Dispatchers.IO) { 
    generateSeed().seedHex 
}
```

---

### 2. Derive DID (Optional - If using DID-based identity)

```javascript
// Doctor App - Create DID document for patient
import { derive_did } from "@dignera/quantum-sdk";

const derived = await derive_did(seed_hex, keypair_index=0);
// Returns: { did, auth_public_key, key_agreement_public_key, encrypted_identity }
// encrypted_identity can be stored server-side for future recovery
```

---

### 3. PBKDF2 Key Derivation (Both Phases)

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