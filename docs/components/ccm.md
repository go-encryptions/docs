# ccm — AES-CCM AEAD

`github.com/go-encryptions/ccm`

```go
import "github.com/go-encryptions/ccm"
```

Package `ccm` implements the **CCM (Counter with CBC-MAC)** AEAD mode specified
by [RFC 3610](https://datatracker.ietf.org/doc/html/rfc3610) and
[NIST SP 800-38C](https://csrc.nist.gov/publications/detail/sp/800-38c/final),
suitable for use with AES (or any 128-bit block cipher).

CCM is **not provided by the Go standard library** — stdlib's `crypto/cipher`
only exposes GCM. This package fills that gap so callers (in particular the
[`zfscrypt`](zfscrypt.md) OpenZFS-encryption parser) don't have to depend on
`golang.org/x/crypto` for AES-CCM.

The implementation is pure Go, uses only `crypto/aes` plus the caller-supplied
`cipher.Block`, and exposes the standard `cipher.AEAD` interface, so it composes
with any code that already understands GCM-style sealed/open.

## Parameter ranges

| Parameter | Symbol | Allowed values |
|-----------|--------|----------------|
| nonce size | N | `[7, 13]` (RFC 3610 §3 — encoded as L = 15 − N) |
| tag size | M | `{4, 6, 8, 10, 12, 14, 16}` |
| plaintext length | P | `len(P) ≤ 2^(8L) − 1` |

Typical OpenZFS usage is **N = 12, M = 16** (L = 3, so the encoded-length field
is 3 bytes — sufficient for any ZFS block, which is at most 2²⁴ bytes).

## API

```go
// NewCCM returns a cipher.AEAD using b in CCM mode with the given
// tag and nonce sizes (bytes). b must have a 16-byte block size;
// tagSize ∈ {4,6,8,10,12,14,16}; nonceSize ∈ [7,13].
func NewCCM(b cipher.Block, tagSize, nonceSize int) (cipher.AEAD, error)
```

The returned value is a standard [`cipher.AEAD`](https://pkg.go.dev/crypto/cipher#AEAD):

- `NonceSize() int` — the configured nonce length.
- `Overhead() int` — bytes `Seal` appends after the ciphertext (the tag).
- `Seal(dst, nonce, plaintext, additionalData []byte) []byte` — encrypts and
  authenticates `plaintext`, authenticates `additionalData`, appends the result
  to `dst`. The nonce must be `NonceSize()` bytes and unique per key.
- `Open(dst, nonce, ciphertext, additionalData []byte) ([]byte, error)` —
  verifies and decrypts. On a tag mismatch it returns an error and leaves `dst`
  unchanged.

## Example

```go
block, _ := aes.NewCipher(key)          // 16/24/32-byte AES key
aead, err := ccm.NewCCM(block, 16, 12)  // tag=16, nonce=12 (the ZFS profile)
if err != nil {
    return err
}

ct := aead.Seal(nil, nonce, plaintext, aad)
pt, err := aead.Open(nil, nonce, ct, aad)
if err != nil {
    // authentication failed
}
```

## How it works

CCM combines two AES passes over the same key:

1. **CBC-MAC** over `B_0 || AAD-blocks || plaintext-blocks` produces the
   authentication tag (`computeMAC`). `B_0` encodes the flags, nonce and
   plaintext length per RFC 3610 §2.2; AAD is prefixed with its length and each
   region is zero-padded to a 16-byte boundary.
2. **CTR mode** encrypts the plaintext starting at counter 1, and the tag is
   masked with the counter-0 keystream block (`ctrXOR` / `formatCounter`).

`Open` decrypts into a scratch buffer, recomputes the CBC-MAC, and compares
against the recovered tag using `crypto/subtle.ConstantTimeCompare`. On failure
the scratch plaintext is zeroed before returning.

## Guarantees

- **Standard library only** — `crypto/aes`, `crypto/cipher`, `crypto/subtle`,
  `encoding/binary`, `errors`. No third-party dependencies.
- **Constant-time tag comparison** to avoid timing oracles.
- **AEAD-conformant** `Seal`/`Open` semantics, including the `Open`-leaves-`dst`-
  unchanged-on-failure contract.

Requires **Go 1.25** or newer.
