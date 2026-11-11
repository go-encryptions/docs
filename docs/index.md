# go-encryptions

Pure-Go **encryption primitives**, with no cgo.

The org fills gaps that the Go standard library and `golang.org/x/crypto`
leave open, and assembles them into the cryptographic core of real on-disk
encryption formats. It has three kinds of module:

- **disk-encryption modes** (`xts`, `cbc`), with assembly kernels ahead of
  OpenSSL on one core;
- **primitives** the standard library lacks (`ccm`, `serpent`, `camellia`);
- **format cores** built on them (`zfscrypt`).

Everything here is the *crypto* layer only. Parsing on-disk structures (the ZFS
`DSL_CRYPTO_KEY` bonus area, `blkptr_t` IV/MAC fields, a LUKS or BitLocker
header…) is the job of the format driver that calls into these packages, such
as [go-fde](https://go-fde.github.io/) or go-filesystems.

## Components

| Module | Import path | What it does |
|--------|-------------|--------------|
| [`xts`](components/xts.md) | `github.com/go-encryptions/xts` | XTS-AES, a drop-in for `golang.org/x/crypto/xts` with byte-identical output and a fused amd64/arm64 kernel. |
| [`cbc`](components/cbc.md) | `github.com/go-encryptions/cbc` | AES-CBC for disk encryption: an 8-block arm64 decryption kernel, and per-sector IVs (eboiv, essiv). |
| [`ccm`](components/ccm.md) | `github.com/go-encryptions/ccm` | AES-CCM (Counter with CBC-MAC) AEAD per RFC 3610 / NIST SP 800-38C. Exposes the standard `cipher.AEAD` interface; fills the gap left by stdlib's GCM-only `crypto/cipher`. |
| [`serpent`](components/serpent.md) | `github.com/go-encryptions/serpent` | The Serpent block cipher as a `cipher.Block`, bitsliced, in the byte order VeraCrypt uses. |
| [`camellia`](components/camellia.md) | `github.com/go-encryptions/camellia` | The Camellia block cipher (RFC 3713) as a `cipher.Block`, pure Go, big-endian as specified. |
| [`zfscrypt`](components/zfscrypt.md) | `github.com/go-encryptions/zfscrypt` | The cryptographic core of OpenZFS native dataset encryption — PBKDF2 wrapping-key derivation, MEK unwrap, HKDF-SHA512 per-block keys, and AES-CCM/GCM block decryption (read path). Depends on `ccm`. |

## Design principles

- **Few dependencies.** The `ccm`, `serpent`, `cbc` and `zfscrypt` packages
  import the standard library only, apart from `zfscrypt` importing this
  org's `ccm`. `xts` adds `golang.org/x/sys/cpu` for feature detection.
- **Standard interfaces.** `ccm` is a drop-in `cipher.AEAD`, `serpent` a
  `cipher.Block`, and `xts` has `x/crypto/xts`'s API.
- **Byte-identical to a reference.** Each module is checked against an
  oracle: `crypto/cipher`, `x/crypto/xts`, Botan's Serpent vectors.
- **Measured.** The kernels are benchmarked against OpenSSL or Botan on one
  core, and a gap is filed as an issue.
- **Constant-time verification.** Tag comparison uses `crypto/subtle`; failed
  authentication wipes scratch plaintext and returns an error with `dst`
  unchanged.
- **Crypto, not format.** These packages take already-parsed keys, IVs, MACs and
  AAD and return plaintext. They never touch the disk.
