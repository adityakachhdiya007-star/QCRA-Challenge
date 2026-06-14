# QCRA Protocol Specification

> **Version:** 18.0 | **Classification:** Public (Algorithms Only)  
> **Note:** This document describes the cryptographic algorithms. Source code is proprietary.

---

## 1. Handshake (Initial Key Exchange)

### 1.1 ML-KEM-1024 (Post-Quantum)

Alice generates an ML-KEM-1024 keypair `(dk, ek)`. She sends `ek` to Bob.

Bob encapsulates:
```
(ciphertext, shared_secret) ← ML-KEM-1024.Encapsulate(ek)
```

Alice decapsulates:
```
shared_secret ← ML-KEM-1024.Decapsulate(dk, ciphertext)
```

Both now hold the same 32-byte `shared_secret`.

### 1.2 X25519 (Classical DH)

Both parties generate ephemeral X25519 keypairs and perform Diffie-Hellman:
```
dh_secret = X25519_DH(our_secret, their_public)
```

### 1.3 Root Key Derivation

The ML-KEM shared secret is fed into HKDF-SHA256:
```
root_key = HKDF-SHA256(salt=None, ikm=mlkem_ss, info="QCRA_ƒ∂_INIT_RK_v13")
```

Then the DH secret is mixed in via the 3-Phase KDF (§2) twice:
```
(root_key_1, chain_key_1) = KDF_3Phase(root_key, dh_secret)
(root_key_2, chain_key_2) = KDF_3Phase(root_key_1, dh_secret)
```

Initiator: `send_chain = chain_key_1`, `recv_chain = chain_key_2`  
Responder: `send_chain = chain_key_2`, `recv_chain = chain_key_1`

---

## 2. Three-Phase KDF Chain

Every chain key advancement uses this interlocked derivation:

### Phase 1: HKDF Extraction
```
raw_64_bytes = HKDF-SHA256.Expand(salt=chain_key, ikm=dh_input, info="QCRA_ƒ∂_RK_STEP_v13", len=64)
```

### Phase 2: BLAKE3 Integrity Tag
```
tag = BLAKE3_KeyedHash(key=chain_key, data=raw_64_bytes)
```

### Phase 3: XOR Interleave
```
for i in 0..32:
    new_root_key[i] = raw_64_bytes[i] XOR tag[i % 32]
    new_chain_key[i] = raw_64_bytes[32+i] XOR tag[(i+16) % 32]
```

### Message Key Derivation
```
next_chain_key = HKDF-SHA256.Expand(salt=None, ikm=chain_key, info="QCRA_ƒ∂_CK_NEXT_v13")
msg_key_raw = HKDF-SHA256.Expand(salt=None, ikm=chain_key, info="QCRA_ƒ∂_MSG_KEY_v13")
ghost_tag = BLAKE3_KeyedHash(key=chain_key, data=msg_key_raw)
msg_key = msg_key_raw XOR ghost_tag  // byte-by-byte
```

---

## 3. Quantum Layer (Three Sub-Layers)

All three sub-layers are activated with the same `shared_secret`. Send and receive chains use **separate** engine instances derived via stream IDs as domain separators.

### 3.1 Lorenz Chaotic Mixer

**Initialization:**
```
(x0, y0, z0) = BLAKE3_KeyedHash(shared_secret, "QCRA_v6_LORENZ_KDF_Λ")
              → mapped to Lorenz basin: x∈[-20,20], y∈[-30,30], z∈[0,50]
Warm up: 1000 RK4 integration steps (dt=0.005)
```

**Key Derivation (every 500 steps):**
```
Advance Lorenz system 500 steps via RK4:
  k1x = σ(y - x),  k1y = x(ρ - z) - y,  k1z = xy - βz
  ... (standard RK4 with σ=10, ρ=28, β=8/3)

state_bytes = x.to_le_bytes() || y.to_le_bytes() || z.to_le_bytes() || step.to_le_bytes()
key = BLAKE3_KeyedHash(key=state_bytes, data="QCRA_v6_LORENZ_KDF_Λ")
```

**Mixing into chain key:**
```
chain_key = chain_key XOR lorenz_key  // byte-by-byte, 32 bytes
```

### 3.2 SO(3) Rotation Key Evolution

**Initialization:**
```
init_quat = Quaternion.from_key_bytes(BLAKE3_KeyedHash(shared_secret, "QCRA_SO3_INIT_STATE_v18"))
target_quat = Quaternion.from_key_bytes(BLAKE3_KeyedHash(shared_secret, "QCRA_SO3_TARGET_STATE_v18"))
t = 0.0, step_size = 0.01
```

**Key Evolution:**
```
t += step_size  // advance along geodesic

if t >= 1.0:
    t = 0.0, epoch += 1
    current = target
    target = Quaternion.from_key_bytes(BLAKE3_KeyedHash(old_target, epoch))

interpolated = SLERP(current, target, t)
rotation_matrix = interpolated.to_rotation_matrix()  // 3×3
continuous_6d = first_two_columns(rotation_matrix)    // 6 floats
key = BLAKE3_Hash(continuous_6d.to_bytes())           // 32 bytes
```

**Mixing:**
```
chain_key = chain_key XOR so3_key  // byte-by-byte
```

### 3.3 P-Adic Encryption (Second Pass)

**Key Derivation:**
```
key_value = BLAKE3_KeyedHash(shared_secret, "QCRA_v6_PADIC_KDF_π") → BigUint
key_value = key_value mod p^precision  // p=104729, precision=20
Ensure key_value is coprime to p
key_inverse = modular_inverse(key_value, p^precision)  // Extended GCD
```

**Encryption (block-by-block):**
```
block_size = ceil(log2(p^precision) / 8)  // ≈42 bytes
chunk_size = block_size - 1

For each chunk of plaintext:
    m = BigUint.from_bytes_le(chunk)
    noise = deterministic_noise(shared_secret, block_index)
    ciphertext_block = (m × key + noise) mod p^precision
```

**Deterministic Noise:**
```
noise_raw = BLAKE3_KeyedHash(key=shared_secret, data=block_index || "QCRA_v6_PADIC_NOISE_ε")
noise = (noise_raw × p^(precision/2)) mod p^precision  // High p-adic valuation
```

**Output Format:**
```
[block_count: u32_le] [last_chunk_size: u32_le] [ciphertext_blocks...]
```

---

## 4. Full Encryption Pipeline (Per Message)

```
1. Mix Lorenz chaotic key into send chain key (XOR)
2. Mix SO(3) rotation key into send chain key (XOR)
3. Derive message key via 3-Phase KDF (§2)
4. Advance send chain key
5. Seal plaintext with ChaCha20-Poly1305:
     nonce = sequence_number as 12-byte LE (padded with zeros)
     ciphertext = ChaCha20Poly1305.Seal(msg_key, nonce, plaintext)
6. P-Adic second-pass encryption (§3.3):
     final_ct = PAdicEngine.Encrypt(ciphertext, padic_key)
```

Output: `(final_ct, sender_dh_public_key, sequence_number)`

---

## 5. Decryption Pipeline (Reverse)

```
1. P-Adic first-pass decryption:
     inner_ct = PAdicEngine.Decrypt(final_ct, padic_key)
2. Mix Lorenz chaotic key into recv chain key (XOR)
3. Mix SO(3) rotation key into recv chain key (XOR)
4. Derive message key via 3-Phase KDF
5. Advance recv chain key
6. Open with ChaCha20-Poly1305:
     plaintext = ChaCha20Poly1305.Open(msg_key, nonce, inner_ct)
```

---

## 6. Constants and Domain Separators

| Constant | Value |
|----------|-------|
| `_Ω` (Root KDF) | `QCRA_ƒ∂_RK_STEP_v13` |
| `_Δ` (Chain advance) | `QCRA_ƒ∂_CK_NEXT_v13` |
| `_Ψ` (Message key) | `QCRA_ƒ∂_MSG_KEY_v13` |
| `_Ξ` (Init root) | `QCRA_ƒ∂_INIT_RK_v13` |
| Lorenz KDF | `QCRA_v6_LORENZ_KDF_Λ` |
| P-Adic KDF | `QCRA_v6_PADIC_KDF_π` |
| P-Adic Noise | `QCRA_v6_PADIC_NOISE_ε` |
| SO(3) Init | `QCRA_SO3_INIT_STATE_v18` |
| SO(3) Target | `QCRA_SO3_TARGET_STATE_v18` |
| Lorenz σ, ρ, β | 10.0, 28.0, 8/3 |
| Lorenz dt | 0.005 |
| Lorenz warmup | 1000 steps |
| Lorenz steps/key | 500 steps |
| P-Adic prime (p) | 104729 |
| P-Adic precision (k) | 20 |
| SO(3) step size | 0.01 |
| AEAD | ChaCha20-Poly1305 |
| Nonce | sequence_number as 12-byte LE |

---

*This specification is published for cryptographic review purposes. Implementation is proprietary.*
