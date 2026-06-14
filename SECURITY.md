# Security Policy

## Reporting Vulnerabilities

If you discover a security vulnerability in the QCRA protocol (beyond the scope of the open challenge), please report it responsibly.

### Contact
- **Email:** adityakachhdiya@gmail.com
- **GitHub Issues:** For non-sensitive findings, open an issue with the `[SECURITY]` tag
- **PGP:** Available on request

### Scope
| In Scope | Out of Scope |
|----------|-------------|
| Protocol-level cryptographic flaws | Social engineering |
| Key derivation weaknesses | Phishing attacks |
| Statistical bias in ciphertext | Physical access attacks (already addressed in threat model) |
| P-Adic/SO(3) mathematical flaws | Denial of service |
| Side-channel vulnerabilities | Attacks requiring the shared secret |

### Response Timeline
- **Acknowledgment:** Within 48 hours
- **Initial Assessment:** Within 7 days
- **Fix/Disclosure:** Within 30 days (coordinated)

### Recognition
Valid vulnerability reports will receive:
- Public credit in the QCRA repository
- Inclusion in the project's security hall of fame
- Discussion of appropriate compensation for critical findings

---

## Challenge vs. Vulnerability Disclosure

- **Challenge submissions** (recovering Message #5 plaintext): Open a GitHub Issue titled `[CHALLENGE] Solution Submission`
- **Vulnerability reports** (protocol flaws that don't require the challenge data): Email or `[SECURITY]` tagged issue

---

*We follow [responsible disclosure](https://en.wikipedia.org/wiki/Responsible_disclosure) principles.*
