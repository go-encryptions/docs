# twofish — Twofish block cipher

`github.com/go-encryptions/twofish`

```go
import "github.com/go-encryptions/twofish"
```

Package `twofish` implements the **Twofish** block cipher (Schneier, Kelsey,
Whiting, Wagner, Hall and Ferguson, 1998) as a standard
[`cipher.Block`](https://pkg.go.dev/crypto/cipher#Block): 16-byte blocks and
16-, 24- or 32-byte keys. It is pure Go, with no cgo and no dependency beyond
the standard library, and replaces the deprecated `golang.org/x/crypto/twofish`.

Its first consumer is VeraCrypt-volume support in `go-fde/veracrypt`, which
needs Twofish under [`go-encryptions/xts`](https://github.com/go-encryptions/xts).

## Not constant-time

The key-dependent S-boxes and the MDS matrix are folded into four 256-entry
tables of 32-bit words per key. Table look-ups use secret-dependent addresses,
so it is **not** constant-time, in the cipher and in the key schedule.

## API

```go
func NewCipher(key []byte) (cipher.Block, error)

type KeySizeError int // returned for any key length other than 16, 24 or 32
```

`Encrypt` and `Decrypt` follow the `cipher.Block` contract: `dst` and `src` may
be the same slice (in-place), and a buffer shorter than 16 bytes panics.

The returned value also has `EncryptBlocks(dst, src []byte)` and
`DecryptBlocks(dst, src []byte)`, found with a type assertion: ECB over
`len(src)/16` blocks, with four blocks going through the rounds side by side.
The bytes are those of `Encrypt` on each block.

```go
block, err := twofish.NewCipher(key) // twofish.KeySizeError unless 16, 24 or 32 bytes
if err != nil {
    return err
}
block.Encrypt(ct[:16], pt[:16])
```

## Verification

- The designers' reference known-answer tests (tables, variable key, variable
  plaintext, 1107 vectors over the three key sizes) and the ECB Monte Carlo
  tests for encryption and decryption, replayed in full.
- The 723 vectors of Botan's `src/tests/data/block/twofish.vec`.
- A differential corpus of 600 random vectors generated from the Botan command
  line by `internal/genbotan`.
- A fuzz test cross-checking round trips and the batched path against
  block-by-block.
- 100 % statement coverage, with the race detector, on amd64 and arm64, and
  under QEMU on big-endian s390x and on riscv64.

## Performance

On an Apple M4 Max, one core, 1 KiB: the batched path is 1.32x faster than
Botan 3.14.1; the one-block path is 2.2x faster than `x/crypto/twofish` but
slower than Botan (a single block is a chain of dependent rounds). See the
[README](https://github.com/go-encryptions/twofish#readme) for the table.
