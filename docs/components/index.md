# Components

`go-encryptions` is a set of pure-Go modules (`CGO_ENABLED=0`). `zfscrypt`
builds directly on `ccm`. `xts` and `cbc` add assembly kernels for amd64 and
arm64, with a portable path everywhere else.

| Module | Import path | Layer | What it does |
|--------|-------------|-------|--------------|
| [`xts`](xts.md) | `github.com/go-encryptions/xts` | mode | XTS-AES (IEEE P1619, NIST SP 800-38E), a drop-in for `golang.org/x/crypto/xts` with byte-identical output. A fused 8-block arm64 kernel, plus `DecryptSectors`; ahead of OpenSSL on one core. |
| [`cbc`](cbc.md) | `github.com/go-encryptions/cbc` | mode | AES-CBC for disk encryption: an 8-block arm64 decryption kernel, plus per-sector IVs (eboiv, essiv) 8 at a time. Byte-identical to `crypto/cipher`; ahead of OpenSSL on one core. |
| [`ctr`](ctr.md) | `github.com/go-encryptions/ctr` | mode | AES-CTR for a stream of messages under one key: the key expanded once, eight blocks in flight, no allocation, `Zeroize`. Byte-identical to `crypto/cipher`; 1.03 of OpenSSL on an Apple M4 Max and 0.97 on a Zen 4 core, per 4 KiB sector (v0.1.2). |
| [`hmac`](hmac.md) | `github.com/go-encryptions/hmac` | primitive | HMAC whose keyed state can be wiped, plus PBKDF2 (RFC 8018). `Wipe` zeroes the precomputed pads and overwrites the live digests, and the page says what it cannot reach. Byte-identical to `crypto/hmac`; level with it at 64 B and 4 KiB on an Apple M4 Max (v0.1.0). |
| [`kdfbudget`](kdfbudget.md) | `github.com/go-encryptions/kdfbudget` | primitive | One shared bound on the total key-derivation work of an unlock: every PBKDF2, Argon2 or hash-chain run is charged before it is spent, in comparable work units, and refused with `ErrLimit` when the total does not fit (v0.1.0). |
| [`argon2`](argon2.md) | `github.com/go-encryptions/argon2` | primitive | Argon2 (RFC 9106), the API and code of `x/crypto/argon2` plus a NEON BLAMKA kernel on arm64: 0.73 of libsodium on an Apple M4 Max (48.9 against 66.5 ms, t=3, m=64 MiB); unchanged on amd64, where it measures 0.25 of libsodium on Zen 4 (v0.1.1). |
| [`ccm`](ccm.md) | `github.com/go-encryptions/ccm` | primitive | AES-CCM AEAD (RFC 3610 / NIST SP 800-38C). A `cipher.AEAD` wrapper around any 128-bit block cipher, with configurable nonce (7–13 bytes) and tag (4–16 bytes) sizes. Fills stdlib's CCM gap. |
| [`zfscrypt`](zfscrypt.md) | `github.com/go-encryptions/zfscrypt` | format core | OpenZFS native-encryption primitives: KDF, MEK unwrap, per-block key derivation, AES-CCM/GCM block decrypt. Read path only. Depends on `ccm`. |
| [`serpent`](serpent.md) | `github.com/go-encryptions/serpent` | primitive | The Serpent block cipher (16/24/32-byte keys) as a `cipher.Block`, in the standard byte order of Botan, VeraCrypt and Linux `serpent`. Bitsliced, with AVX2 kernels on amd64. |
| [`camellia`](camellia.md) | `github.com/go-encryptions/camellia` | primitive | The Camellia block cipher (RFC 3713, 16/24/32-byte keys) as a `cipher.Block`. Pure Go, big-endian as specified. |
| [`kuznyechik`](kuznyechik.md) | `github.com/go-encryptions/kuznyechik` | primitive | The Kuznyechik block cipher (GOST R 34.12-2015 / RFC 7801, 32-byte keys) as a `cipher.Block`, in the byte order of the RFC and VeraCrypt. Pure Go, table-driven. |
| [`twofish`](twofish.md) | `github.com/go-encryptions/twofish` | primitive | The Twofish block cipher (16/24/32-byte keys) as a `cipher.Block`, with optional `EncryptBlocks`/`DecryptBlocks`. Pure Go, table-driven. |
| [`whirlpool`](whirlpool.md) | `github.com/go-encryptions/whirlpool` | primitive | The Whirlpool-512 hash (ISO/IEC 10118-3) as a `hash.Hash` that marshals its state. Table-driven Go with arm64 and amd64 kernels (AVX-512 on amd64, 1.27x faster than Botan on one core). |
| [`streebog`](streebog.md) | `github.com/go-encryptions/streebog` | primitive | The Streebog hash (GOST R 34.11-2012 / RFC 6986), 256 and 512 bits, as a `hash.Hash` that marshals its state. Pure Go, table-driven. |
| [`unixcrypt`](unixcrypt.md) | `github.com/go-encryptions/unixcrypt` | primitive | The crypt(3) password hashes MD5-crypt, SHA-256/512-crypt and bcrypt (`$2a$`/`$2b$`/`$2x$`/`$2y$`), checked against `openssl passwd` and the OpenBSD and crypt_blowfish vectors. Pure Go. |

## Dependency graph

```
zfscrypt ──► ccm ──► crypto/aes, crypto/cipher (stdlib)
   │
   └──► crypto/pbkdf2, crypto/hkdf, crypto/hmac, crypto/sha1, crypto/sha512 (stdlib)
```

`ccm` has zero non-stdlib dependencies. `zfscrypt` depends on exactly one
external module — this org's own `ccm` — because the Go standard library does
not provide CCM.
