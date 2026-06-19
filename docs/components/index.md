# Components

`go-encryptions` is a set of dependency-free Go modules (standard library only,
`CGO_ENABLED=0`). `zfscrypt` builds directly on `ccm`; nothing else is required.

| Module | Import path | Layer | What it does |
|--------|-------------|-------|--------------|
| [`ccm`](ccm.md) | `github.com/go-encryptions/ccm` | primitive | AES-CCM AEAD (RFC 3610 / NIST SP 800-38C). A `cipher.AEAD` wrapper around any 128-bit block cipher, with configurable nonce (7–13 bytes) and tag (4–16 bytes) sizes. Fills stdlib's CCM gap. |
| [`zfscrypt`](zfscrypt.md) | `github.com/go-encryptions/zfscrypt` | format core | OpenZFS native-encryption primitives: KDF, MEK unwrap, per-block key derivation, AES-CCM/GCM block decrypt. Read path only. Depends on `ccm`. |

## Dependency graph

```
zfscrypt ──► ccm ──► crypto/aes, crypto/cipher (stdlib)
   │
   └──► crypto/pbkdf2, crypto/hkdf, crypto/hmac, crypto/sha1, crypto/sha512 (stdlib)
```

`ccm` has zero non-stdlib dependencies. `zfscrypt` depends on exactly one
external module — this org's own `ccm` — because the Go standard library does
not provide CCM.
