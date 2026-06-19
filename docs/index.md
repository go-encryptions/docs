# go-encryptions

Pure-Go **encryption primitives**, standard library only — no cgo, no
`golang.org/x/crypto`, no third-party crypto dependencies.

The org fills gaps the Go standard library leaves open and assembles them into
the cryptographic core of real on-disk encryption formats. Today that means two
modules: a self-contained **AES-CCM** AEAD (which stdlib does not provide), and
**zfscrypt**, the "math" half of OpenZFS native dataset encryption built on top
of it.

Everything here is the *crypto* layer only. Parsing on-disk structures (the ZFS
`DSL_CRYPTO_KEY` bonus area, `blkptr_t` IV/MAC fields, …) is the job of the
format driver that calls into these packages — keeping the primitives small,
auditable, and independent of any particular disk layout.

## Components

| Module | Import path | What it does |
|--------|-------------|--------------|
| [`ccm`](components/ccm.md) | `github.com/go-encryptions/ccm` | AES-CCM (Counter with CBC-MAC) AEAD per RFC 3610 / NIST SP 800-38C. Exposes the standard `cipher.AEAD` interface; fills the gap left by stdlib's GCM-only `crypto/cipher`. |
| [`zfscrypt`](components/zfscrypt.md) | `github.com/go-encryptions/zfscrypt` | The cryptographic core of OpenZFS native dataset encryption — PBKDF2 wrapping-key derivation, MEK unwrap, HKDF-SHA512 per-block keys, and AES-CCM/GCM block decryption (read path). Depends on `ccm`. |

## Design principles

- **Standard library only.** The sole exception is `zfscrypt` depending on this
  org's own `ccm` (because stdlib has no CCM). No `x/crypto`, no SDKs.
- **Standard interfaces.** `ccm` is a drop-in `cipher.AEAD`, so it composes with
  any code that already understands GCM-style `Seal`/`Open`.
- **Constant-time verification.** Tag comparison uses `crypto/subtle`; failed
  authentication wipes scratch plaintext and returns an error with `dst`
  unchanged.
- **Crypto, not format.** These packages take already-parsed keys, IVs, MACs and
  AAD and return plaintext. They never touch the disk.
