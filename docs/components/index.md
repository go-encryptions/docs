# Components

`go-encryptions` is a set of pure-Go modules (`CGO_ENABLED=0`). `zfscrypt`
builds directly on `ccm`. `xts` and `cbc` add assembly kernels for amd64 and
arm64, with a portable path everywhere else.

| Module | Import path | Layer | What it does |
|--------|-------------|-------|--------------|
| [`xts`](xts.md) | `github.com/go-encryptions/xts` | mode | XTS-AES (IEEE P1619, NIST SP 800-38E), a drop-in for `golang.org/x/crypto/xts` with byte-identical output. A fused 8-block arm64 kernel, plus `DecryptSectors`; ahead of OpenSSL on one core. |
| [`cbc`](cbc.md) | `github.com/go-encryptions/cbc` | mode | AES-CBC for disk encryption: an 8-block arm64 decryption kernel, plus per-sector IVs (eboiv, essiv) 8 at a time. Byte-identical to `crypto/cipher`; ahead of OpenSSL on one core. |
| [`ccm`](ccm.md) | `github.com/go-encryptions/ccm` | primitive | AES-CCM AEAD (RFC 3610 / NIST SP 800-38C). A `cipher.AEAD` wrapper around any 128-bit block cipher, with configurable nonce (7–13 bytes) and tag (4–16 bytes) sizes. Fills stdlib's CCM gap. |
| [`zfscrypt`](zfscrypt.md) | `github.com/go-encryptions/zfscrypt` | format core | OpenZFS native-encryption primitives: KDF, MEK unwrap, per-block key derivation, AES-CCM/GCM block decrypt. Read path only. Depends on `ccm`. |
| [`serpent`](serpent.md) | `github.com/go-encryptions/serpent` | primitive | The Serpent block cipher (16/24/32-byte keys) as a `cipher.Block`, in the standard byte order of Botan, VeraCrypt and Linux `serpent`. Pure Go, bitsliced. |
| [`camellia`](camellia.md) | `github.com/go-encryptions/camellia` | primitive | The Camellia block cipher (RFC 3713, 16/24/32-byte keys) as a `cipher.Block`. Pure Go, big-endian as specified. |

## Dependency graph

```
zfscrypt ──► ccm ──► crypto/aes, crypto/cipher (stdlib)
   │
   └──► crypto/pbkdf2, crypto/hkdf, crypto/hmac, crypto/sha1, crypto/sha512 (stdlib)
```

`ccm` has zero non-stdlib dependencies. `zfscrypt` depends on exactly one
external module — this org's own `ccm` — because the Go standard library does
not provide CCM.
