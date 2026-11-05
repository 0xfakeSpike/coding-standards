# Cryptography and secrets

This document covers algorithm selection, key management, TLS, and field encryption. Password storage is a separate problem. See [Authentication](authentication.md). For secure random APIs, see `languages/<language>/security.md`. For how configuration is loaded, see [Configuration](../architecture/configuration.md).

## Algorithm selection

### Use mature implementations and algorithms [Required]

Use a maintained cryptography library. Do not implement algorithms or protocols yourself. Choose from the table below.

| Purpose | Use | Forbidden |
|---|---|---|
| Symmetric encryption | AES-256-GCM, ChaCha20-Poly1305 | ECB, DES, RC4 |
| Hashing | SHA-256 and above, BLAKE2 | MD5, SHA-1 |
| Message authentication | HMAC-SHA-256 | Hand-assembled hash constructions |
| Signatures | Ed25519, ECDSA P-256 and above, RSA 3072-bit and above | RSA keys shorter than 2048 bits |
| Public-key encryption | HPKE, or RSA-OAEP | RSA PKCS#1 v1.5 encryption |
| Key derivation | HKDF, and a password hashing algorithm for deriving from passwords | KDFs based on MD5 or SHA-1 |

**Why:** A secure algorithm does not mean a secure implementation. Homegrown implementations rarely avoid side channels and misuse.

**Source:** OWASP Cryptographic Storage Cheat Sheet, OWASP ASVS

### Use authenticated encryption [Required]

Prefer authenticated modes such as GCM. When only CTR or CBC is available, you must add a MAC, using encrypt-then-MAC. Do not reveal the reason for a decryption failure externally.

**Why:** Data that is encrypted but not authenticated can be tampered with, and differences in errors enable padding oracle attacks.

**Source:** OWASP Cryptographic Storage Cheat Sheet, OWASP ASVS

### Never repeat a nonce [Required]

Do not reuse an IV or nonce under the same key. A repeated GCM nonce breaks both confidentiality and integrity. Rotate the key before reaching the usage limit of the chosen mode.

**Source:** OWASP ASVS, OWASP Cryptographic Storage Cheat Sheet

### Use a CSPRNG for secure random values [Required]

Generate unguessable values such as tokens, keys, IVs, and session IDs with a CSPRNG, with at least 128 bits of entropy. Do not use ordinary random numbers, and do not treat UUIDs as security tokens.

**Source:** OWASP Cryptographic Storage Cheat Sheet, OWASP ASVS

### Compare secret values in constant time [Required]

**Applies to:** high risk

Compare signatures, MACs, and tokens in constant time, not with ordinary string comparison that returns early.

**Source:** OWASP ASVS

## Key management

### Keep secrets out of code, images, and logs [Required]

Put keys, tokens, and database passwords in a secrets management service, or have the platform inject them securely. Do not write them into code, image layers, build arguments, logs, error messages, or URLs. Environment variables can be used for injection, but note they are easily exposed through process information and crash dumps.

Avoid:

```yaml
env:
  DB_PASSWORD: s3cr3t-value
```

Prefer:

```yaml
env:
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: orders-db
        key: password
```

**Tooling:** gitleaks, trufflehog, run in commit hooks and CI.

**Source:** OWASP Secrets Management Cheat Sheet, OWASP Cryptographic Storage Cheat Sheet

### Revoke leaked secrets first [Required]

Treat any secret that has entered Git history as leaked. Revoke or rotate it first, then clean the history. Deleting the commit alone does not solve it.

### Store keys separately from data [Required]

**Applies to:** high risk

Do not keep keys in the same store as the data they protect. With envelope encryption, the DEK encrypts the data and the KEK encrypts the DEK. Keep the KEK in a KMS or HSM. The encrypted DEK can sit with the data.

**Source:** OWASP Cryptographic Storage Cheat Sheet

### Use each key for one purpose [Required]

**Applies to:** high risk

Use different keys for encryption, signing, and authentication, use different keys per environment, and separate by tenant when the risk model requires it. Generate keys randomly, not from words or phrases.

**Source:** OWASP Key Management Cheat Sheet, OWASP Cryptographic Storage Cheat Sheet

### Prepare the rotation process in advance [Required]

**Applies to:** high risk

Rotate in these cases: a suspected leak, departure of someone with access, reaching the key's validity period, nearing the mode's usage limit, or a new attack on the algorithm. Have the rotation method ready before you need it. Prefer dynamic, short-lived credentials to reduce manual rotation. User passwords are not included here and change only when there are signs of a leak.

**Why:** If you start writing the rotation process only when a leak happens, the exposure window gets longer.

**Source:** OWASP Cryptographic Storage Cheat Sheet, OWASP Secrets Management Cheat Sheet

### Record the key used for each ciphertext [Recommended]

Store a key ID with each ciphertext. After rotation, new data uses the new key. When rotating the KEK, rewrap the DEKs with the new KEK, then destroy the old KEK.

**Source:** OWASP Cryptographic Storage Cheat Sheet, OWASP Key Management Cheat Sheet

## TLS

### Use TLS on every link [Required]

**Applies to:** production service

Encrypt public endpoints and internal links across hosts, including databases, caches, messaging, and monitoring, with no fallback to plaintext. Browser-facing entry points redirect HTTP to HTTPS and configure HSTS. See [Web risks](web-risks.md). APIs do not redirect transparently. They reject plaintext directly.

**Source:** OWASP ASVS, OWASP Transport Layer Security Cheat Sheet

### Enable only strong protocols and cipher suites [Required]

Default to TLS 1.3, with TLS 1.2 as the minimum. Disable TLS 1.0, 1.1, and SSL. For TLS 1.2, prefer ECDHE with AEAD cipher suites.

```nginx
ssl_protocols TLSv1.2 TLSv1.3;
```

**Tooling:** testssl.sh

**Source:** OWASP Transport Layer Security Cheat Sheet, OWASP ASVS

### Clients do not disable certificate validation [Required]

Do not disable certificate or hostname validation in any environment. Public services use publicly trusted certificates. For internal CAs or self-signed certificates, trust only the specified CA.

**Why:** With validation off, TLS cannot stop man-in-the-middle attacks.

**Source:** OWASP ASVS

### Validate the certificate before trusting an mTLS identity [Required]

Before using the identity in a client certificate for authentication or authorization, verify that the certificate is trusted.

**Source:** OWASP ASVS

## Field encryption

### Choose the encryption layer by threat model [Recommended]

| Layer | Protects against | Does not protect against |
|---|---|---|
| Disk or file system | Theft of physical media | Compromise of the application or database |
| Transparent database encryption | Copying of data files and backups | Accounts that can query the database |
| Application-level field encryption | Leaks through database accounts, DBAs, or backups | Compromise of the application itself |

The best protection is not storing the data. Do not store sensitive fields you can do without. For example, keep only a token's hash or the last four digits of a bank card.

**Source:** OWASP Cryptographic Storage Cheat Sheet

### Encrypt sensitive fields at the application layer [Recommended]

For highly sensitive fields such as ID numbers and bank accounts, use authenticated encryption and envelope encryption, and include the key ID with the ciphertext. You can use the record ID as additional authenticated data to prevent ciphertext from being moved to other rows.

### Use blind indexes to query encrypted fields [Recommended]

For equality queries, generate a separate index column with a keyed HMAC. Do not use deterministic encryption or unkeyed hashes. A blind index still reveals that "two rows have the same value", so it does not suit low-cardinality fields.

**Why:** Values such as phone numbers have a small value space, so unkeyed hashes can be enumerated and reversed.
