# zfscrypt — OpenZFS encryption core

`github.com/go-encryptions/zfscrypt`

```go
import "github.com/go-encryptions/zfscrypt"
```

Package `zfscrypt` implements the cryptographic primitives used by **OpenZFS
native (dataset-level) encryption**, as added in OpenZFS 0.8 (2019) and stable
since.

This package is the **"math" half** of ZFS encryption: KDF, MEK unwrap,
per-block key derivation, and AES-CCM/GCM block decryption. The **"format"
half** — parsing the `DSL_CRYPTO_KEY` bonus area and extracting per-block IV/MAC
from `blkptr_t` — lives in the consuming filesystem driver, which already
understands those on-disk structures.

Only the **read path** is currently implemented (the motivating use case,
cloud-boot, just needs to pull `/boot/vmlinuz` and `/boot/initrd` out of an
encrypted root), so `Wrap`/`EncryptBlock` are not exposed. Adding them later is
straightforward — `Seal` is already present in the underlying AEADs.

### References (OpenZFS source tree)

`module/zfs/zio_crypt.c` · `module/zfs/dsl_crypt.c` ·
`include/sys/zio_crypt.h` · `include/sys/dsl_crypt.h`

## Algorithms & key schedule

| Stage | Algorithm |
|-------|-----------|
| Wrapping-key derivation | PBKDF2-HMAC-**SHA1**(passphrase, salt, iters, 32) |
| MEK unwrap | AES-128/192/256-**CCM** or **GCM** AEAD `Open` |
| Per-block key derivation | HKDF-**SHA512**(mek, salt=blockSalt, info=algName, klen) |
| Block decryption | AES-128/192/256-CCM (via [`ccm`](ccm.md)) or AES-GCM (stdlib) |
| Metadata auth | HMAC-SHA512, truncated to 32 bytes |

!!! note "SHA1 is intentional"
    The wrapping-key KDF uses **PBKDF2-HMAC-SHA1**, matching what OpenZFS
    shipped in 0.8. It is part of the on-disk format, not a security choice —
    do not "upgrade" it without a corresponding format change.

## Suites

`Suite` is a `uint8` matching the on-disk `crypt_algorithm` field of a
`DSL_CRYPTO_KEY` object:

| Constant | Value | Key length |
|----------|-------|------------|
| `SuiteInherit` | 0 | — (resolved by the format layer) |
| `AES128CCM` | 1 | 16 |
| `AES192CCM` | 2 | 24 |
| `AES256CCM` | 3 | 32 |
| `AES128GCM` | 4 | 16 |
| `AES192GCM` | 5 | 24 |
| `AES256GCM` | 6 | 32 |

Helpers: `Suite.KeyLen() int`, `Suite.IsCCM() bool`, `Suite.String() string`.

Fixed sizes (ZFS uses 12-byte IVs and 16-byte MACs uniformly):

```go
const (
    IVSize         = 12 // per-block / unwrap IV
    MACSize        = 16 // authentication tag
    WrappingKeyLen = 32 // always AES-256 wraps the MEK
    WrappedKeySize = 64 // 32-byte MEK || 32-byte HMAC key
)
```

## API

```go
// DeriveWrappingKey derives the 32-byte wrapping key from the user
// passphrase via PBKDF2-HMAC-SHA1. iters and salt come from the
// DSL_CRYPTO_KEY object.
func DeriveWrappingKey(passphrase, salt []byte, iters int) ([]byte, error)

// Unwrap decrypts the wrapped (MEK || HMAC-key) blob with the
// wrapping key, returning the 32-byte MEK and 32-byte HMAC key.
// suite selects CCM vs GCM; ad must equal the AD the pool used at
// wrap time, or Open fails the tag check.
func Unwrap(suite Suite, wrappingKey, iv, mac, wrapped, ad []byte) (mek, hmacKey []byte, err error)

// DeriveBlockKey derives the per-block data key from the MEK via
// HKDF-SHA512 (salt = per-block randomiser, info = suite.String()).
func DeriveBlockKey(suite Suite, mek, salt []byte) ([]byte, error)

// DecryptBlock authenticates and decrypts one ZFS block with a
// previously-derived per-block key.
func DecryptBlock(suite Suite, key, iv, mac, ciphertext, ad []byte) ([]byte, error)

// HMAC returns HMAC-SHA512 over data, truncated to 32 bytes, using
// the dataset's HMAC key — for metadata that bypasses the AEAD
// layer (e.g. the per-block salt).
func HMAC(hmacKey, data []byte) []byte
```

## Read path

```go
// 1. Derive the wrapping key from the passphrase.
wk, err := zfscrypt.DeriveWrappingKey(passphrase, kdfSalt, iters)

// 2. Unwrap the master encryption key (and its HMAC sibling).
mek, hmacKey, err := zfscrypt.Unwrap(suite, wk, wrapIV, wrapMAC, wrapped, wrapAD)

// 3. Per encrypted block: derive its key, then decrypt.
blockKey, err := zfscrypt.DeriveBlockKey(suite, mek, blockSalt)
pt, err := zfscrypt.DecryptBlock(suite, blockKey, blockIV, blockMAC, ciphertext, blockAD)
```

The IV, MAC, salt and AAD inputs are supplied by the format driver after it
parses the `DSL_CRYPTO_KEY` object and the encrypted block's `blkptr_t`. Passing
the wrong AAD surfaces as a tag-verification failure, not as garbage plaintext.

## Internals

`decryptAEAD` builds the right AEAD for the suite — `ccm.NewCCM(block, 16, 12)`
for CCM suites or `cipher.NewGCMWithNonceSize(block, 12)` for GCM — then splices
`ciphertext || mac` back together (ZFS stores the tag separately in the
`blkptr`) and calls `Open` with the IV as the nonce.

## Dependencies

Standard library (`crypto/aes`, `crypto/cipher`, `crypto/hkdf`, `crypto/hmac`,
`crypto/pbkdf2`, `crypto/sha1`, `crypto/sha512`) plus this org's own
[`ccm`](ccm.md) for the CCM suites. No `golang.org/x/crypto`.

Requires **Go 1.25** or newer.
