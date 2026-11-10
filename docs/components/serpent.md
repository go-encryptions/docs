# serpent — Serpent block cipher

`github.com/go-encryptions/serpent`

```go
import "github.com/go-encryptions/serpent"
```

Package `serpent` implements the **Serpent** block cipher (Anderson, Biham and
Knudsen, AES finalist) as a standard
[`cipher.Block`](https://pkg.go.dev/crypto/cipher#Block): 16-byte blocks and
16-, 24- or 32-byte keys. It is pure Go, bitsliced, with no cgo and no
dependency beyond the standard library.

Its first consumer is VeraCrypt-volume support in `go-fde/veracrypt`, which
needs Serpent under [`go-encryptions/xts`](https://github.com/go-encryptions/xts) with 256-bit keys.

## Byte order

There are two conventions for the byte order of Serpent's key, plaintext and
ciphertext. This package uses the **standard** one, the same as Botan,
VeraCrypt and the Linux `serpent` algorithm: block and key are read as
little-endian 32-bit words. It does not use the reversed order of Linux's
`tnepres` nor of the original submission's test files.

## API

```go
func NewCipher(key []byte) (cipher.Block, error)

type KeySizeError int // returned for any key length other than 16, 24, 32
```

`Encrypt` and `Decrypt` follow the `cipher.Block` contract: `dst` and `src` may
be the same slice (in-place), and a buffer shorter than 16 bytes panics.

```go
block, err := serpent.NewCipher(key) // serpent.KeySizeError if len(key) is not 16, 24 or 32
if err != nil {
    return err
}
block.Encrypt(ct[:16], pt[:16])
```

## Verification

- All 1047 vectors of Botan's `src/tests/data/block/serpent.vec` (all three key
  sizes), which fixes the byte order above.
- A differential corpus of 750 random vectors generated from the Botan command
  line by `internal/genbotan`.
- A fuzz test cross-checking `Encrypt`/`Decrypt` round trips.
- 100 % statement coverage, with the race detector, on amd64 and arm64, and
  under QEMU on big-endian s390x.

See the [README](https://github.com/go-encryptions/serpent#readme) for the
benchmark against Botan.
