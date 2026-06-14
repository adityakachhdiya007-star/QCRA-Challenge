# QCRA Cryptographic Open Challenge 🔐

**Protocol:** QCRA v18.0 — Sovereign Post-Quantum Defense Protocol  
**Language:** Rust 🦀  
**Challenge Version:** 1.0  
**Status:** 🟢 ACTIVE  

---

## 📖 What is QCRA?

QCRA (Quantum-Chess Routing Architecture) is an experimental post-quantum VPN protocol that goes beyond standard ML-KEM implementations by adding **three novel cryptographic layers** to defeat AI-driven traffic analysis:

| Layer | Technology | What It Does |
|-------|-----------|--------------|
| **Post-Quantum KEM** | ML-KEM-1024 + X25519 Hybrid | Resists quantum key recovery (Shor's algorithm) |
| **Double Ratchet** | 3-Phase HKDF-SHA256 + BLAKE3 | Per-message forward secrecy |
| **P-Adic KDF** | Non-Archimedean ℚₚ arithmetic | Defeats gradient-based ML attacks (ultrametric space) |
| **SO(3) Key Evolution** | Quaternion SLERP + 6D Continuous | Eliminates statistical key-evolution fingerprints |
| **Lorenz Chaotic Mixer** | RK4-integrated Lorenz attractor | Adds chaotic entropy mixing to KDF chains |
| **P-Adic Encryption** | Modular arithmetic in ℤ/p^k | Second-pass encryption in non-Archimedean metric |
| **AEAD** | ChaCha20-Poly1305 | Authenticated encryption of final payload |

The full protocol is described in the [Whitepaper (LaTeX source)](whitepaper/main.tex).

---

## 🎯 The Challenge

### Goal
**Recover the plaintext of Message #5** from the encrypted session transcript.

### What You're Given

| File | Description |
|------|-------------|
| [`session_transcript.json`](challenge/session_transcript.json) | Full session data: public keys, KEM ciphertext, all 5 encrypted messages, session parameters |
| [`ciphertexts.hex`](challenge/ciphertexts.hex) | Raw hex ciphertexts (one per line) |
| [`verification.sha256`](challenge/verification.sha256) | SHA-256 hash of the correct plaintext for Message #5 |
| [`whitepaper/main.tex`](whitepaper/main.tex) | Full mathematical specification of all cryptographic layers |
| [`protocol_spec.md`](protocol_spec.md) | Simplified protocol specification with algorithm pseudocode |

### What's NOT Given (The Secrets)
- ❌ Alice or Bob's X25519 private keys
- ❌ Alice's ML-KEM-1024 decapsulation key
- ❌ The 32-byte shared secret derived from the handshake
- ❌ Any intermediate ratchet chain keys
- ❌ Source code implementation (proprietary — only algorithms are published)

### Hints
- All 5 messages are English sentences
- Message #5 is **69 characters** long
- Messages were encrypted sequentially (Alice → Bob, messages 1 through 5)

### How to Verify Your Answer
```bash
echo -n 'YOUR_RECOVERED_PLAINTEXT' | sha256sum
# Should match: 2f8597bccf9f10ad43ab38ce5c0f69748c72c221c318f7626a1cb179cc88de12
```

---

## 🏆 Rules & Reward

| Rule | Detail |
|------|--------|
| **Goal** | Recover the plaintext of Message #5 |
| **Valid Attack** | Any — break the KEM, the ratchet, the P-Adic layer, find bias in ciphertext, side-channel analysis, anything goes |
| **Deadline** | 30 days from post date |
| **Reward** | Public credit in the QCRA repository + discussion of bug bounty for critical findings |
| **Verification** | SHA-256 hash match (see above) |
| **Submission** | Open a GitHub Issue titled `[CHALLENGE] Solution Submission` with your methodology |

### Partial Credit
Even if you can't recover the plaintext, we value:
- Statistical bias analysis of the ciphertexts
- Formal critique of the P-Adic AI-resistance claim (Theorem 3.1 in whitepaper)
- Protocol-level attacks (e.g., showing the ratchet can be broken without brute force)
- Side-channel considerations in the Lorenz chaotic mixer

---

## 🔬 Technical Details

### Encryption Pipeline (per message)

```
Plaintext
  ↓
[1] Lorenz Chaotic XOR (send chain key mixing)
  ↓
[2] SO(3) Rotation Key XOR (geodesic evolution on S³)
  ↓
[3] HKDF-SHA256 → BLAKE3 MAC → XOR Interleave (3-phase KDF)
  ↓
[4] ChaCha20-Poly1305 AEAD Seal
  ↓
[5] P-Adic Encryption: E(m) = (m × key + noise) mod p^20
  ↓
Ciphertext (output)
```

### Session Parameters
- **ML-KEM:** ML-KEM-1024 (FIPS 203)
- **Classical DH:** X25519
- **P-Adic Prime:** p = 104729
- **P-Adic Precision:** 20 digits (modulus ≈ 334 bits)
- **Lorenz:** σ=10, ρ=28, β=8/3, dt=0.005, 1000 warmup steps, 500 steps/key
- **SO(3):** Quaternion SLERP, step size 0.01, BLAKE3 extraction
- **AEAD:** ChaCha20-Poly1305 with sequence-derived nonces
- **KDF Chain:** HKDF-SHA256 + BLAKE3 keyed MAC + XOR interleave

---

## 📁 Repository Structure

```
QCRA-Challenge/
├── README.md                          ← This file
├── protocol_spec.md                   ← Protocol specification (algorithms only)
├── SECURITY.md                        ← Responsible disclosure
├── challenge/
│   ├── session_transcript.json        ← Full session data
│   ├── ciphertexts.hex                ← Raw ciphertexts
│   └── verification.sha256           ← Answer verification hash
├── whitepaper/
│   └── main.tex                       ← Full academic whitepaper
└── posts/
    ├── reddit_post.md                 ← Reddit draft
    └── hackernews_post.md             ← HN draft
```

---

## ⚠️ Disclaimer

QCRA is an **experimental research protocol** at TRL-4 (Technology Readiness Level 4 — validated in laboratory). It has NOT undergone formal security audit or third-party review. This challenge is an invitation for community scrutiny.

The P-Adic AI resistance claims (Section 3 of whitepaper) are theoretical arguments, not formal security proofs with reductions to known hard problems. We explicitly invite critique of these claims.

---

## 📜 License

Challenge data is released under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).  
QCRA protocol implementation is proprietary. The whitepaper and protocol specification are published for research purposes.

---

*Built with Rust 🦀 | Architecture by QCRA Team | June 2026*
