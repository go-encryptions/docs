# camellia — Camellia block cipher

`github.com/go-encryptions/camellia`

```go
import "github.com/go-encryptions/camellia"
```

Package `camellia` implements the **Camellia** block cipher (RFC 3713; Aoki,
Ichikawa, Kanda, Matsui, Moriai, Nakajima and Tokita) as a standard
[`cipher.Block`](https://pkg.go.dev/crypto/cipher#Block): 16-byte blocks and
16-, 24- or 32-byte keys. It is pure Go, with no cgo and no dependency beyond
the standard library.

Its first consumer is VeraCrypt-volume support in `go-fde/veracrypt`, which
needs Camellia under [`go-encryptions/xts`](https://github.com/go-encryptions/xts) with 256-bit keys.

## Byte order

Camellia is big-endian by specification: block and key are read as big-endian
64-bit words, so the ciphertext is the same on every architecture.

## API

```go
func NewCipher(key []byte) (cipher.Block, error)

type KeySizeError int // returned for any key length other than 16, 24, 32
```

`Encrypt` and `Decrypt` follow the `cipher.Block` contract: `dst` and `src` may
be the same slice (in-place), and a buffer shorter than 16 bytes panics.

```go
block, err := camellia.NewCipher(key) // camellia.KeySizeError if len(key) is not 16, 24 or 32
if err != nil {
    return err
}
block.Encrypt(ct[:16], pt[:16])
```

## Verification

- The three vectors of RFC 3713 Appendix A (128-, 192- and 256-bit keys).
- The 17 vectors of Botan's `src/tests/data/block/camellia.vec`.
- A differential corpus of 750 random vectors generated from the Botan command
  line by `internal/genbotan`.
- A fuzz test cross-checking `Encrypt`/`Decrypt` round trips.
- 100 % statement coverage, with the race detector, on amd64 and arm64, and
  under QEMU on big-endian s390x and on riscv64.

See the [README](https://github.com/go-encryptions/camellia#readme) for the
benchmark against Botan.
