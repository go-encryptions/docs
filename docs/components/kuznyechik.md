# kuznyechik — Kuznyechik block cipher

`github.com/go-encryptions/kuznyechik`

```go
import "github.com/go-encryptions/kuznyechik"
```

Package `kuznyechik` implements the **Kuznyechik** block cipher of GOST R
34.12-2015 (RFC 7801) as a standard
[`cipher.Block`](https://pkg.go.dev/crypto/cipher#Block): 16-byte blocks and
32-byte keys. It is pure Go, with no cgo and no dependency beyond the standard
library.

Its first consumer is VeraCrypt-volume support in `go-fde/veracrypt`, which
needs Kuznyechik under [`go-encryptions/xts`](https://github.com/go-encryptions/xts)
with 256-bit keys.

## Byte order

Blocks and keys are the byte strings of RFC 7801 section 5 as printed there:
the first byte is the leftmost one. VeraCrypt's `kuznyechik.c` uses the same
arrays (its tables are built from the same `l` and `R`, and its first round
constant is the RFC's `C_1`), so the two agree byte for byte. The package loads
words explicitly little-endian, so the result is the same on every
architecture.

## Not constant-time

The implementation is table-driven (the S-box and the linear transformation
are folded into sixteen 256-entry tables). Table look-ups use secret-dependent
addresses, so it is **not** constant-time.

## API

```go
func NewCipher(key []byte) (cipher.Block, error)

type KeySizeError int // returned for any key length other than 32
```

`Encrypt` and `Decrypt` follow the `cipher.Block` contract: `dst` and `src` may
be the same slice (in-place), and a buffer shorter than 16 bytes panics.

```go
block, err := kuznyechik.NewCipher(key) // kuznyechik.KeySizeError if len(key) != 32
if err != nil {
    return err
}
block.Encrypt(ct[:16], pt[:16])
```

## Verification

- RFC 7801 section 5: the S, R and L examples, the round constants and ten round
  keys, and the encryption and decryption example.
- The 65 vectors of Botan's `src/tests/data/block/kuznyechik.vec`.
- A differential corpus of 500 random vectors generated from the Botan command
  line by `internal/genbotan`.
- A fuzz test cross-checking `Encrypt`/`Decrypt` round trips.
- 100 % statement coverage, with the race detector, on amd64 and arm64, and
  under QEMU on big-endian s390x and on riscv64.

See the [README](https://github.com/go-encryptions/kuznyechik#readme) for the
benchmark against Botan.
