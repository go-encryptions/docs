# whirlpool — Whirlpool-512 hash

`github.com/go-encryptions/whirlpool`

```go
import "github.com/go-encryptions/whirlpool"
```

Package `whirlpool` implements the **Whirlpool** hash function (ISO/IEC
10118-3, 512-bit digest) as a standard
[`hash.Hash`](https://pkg.go.dev/hash#Hash). It is the final 2003 version of
Barreto and Rijmen, with the revised S-box and the diffusion matrix
`cir(1, 1, 4, 1, 8, 5, 2, 9)`. It is Go with no cgo and no dependency beyond
the standard library; on arm64 the compression function is an assembly kernel
(generated with go-asmgen) that keeps the whole round in registers, and every
other architecture runs the portable Go form.

Its first consumer is VeraCrypt-volume support in `go-fde/veracrypt`, which
needs Whirlpool-512 as one of the PRFs of PBKDF2-HMAC when it derives header
keys.

## Not constant-time

The round function is eight 256-entry tables of 64-bit words indexed by state
bytes. Table look-ups use secret-dependent addresses, so it is **not**
constant-time.

## API

```go
func New() hash.Hash              // Size() == 64, BlockSize() == 64
func Sum(data []byte) [64]byte    // one shot
```

The value returned by `New` also implements `encoding.BinaryMarshaler`,
`encoding.BinaryUnmarshaler`, `encoding.BinaryAppender` and `hash.Cloner`.
`crypto/hmac` and `crypto/pbkdf2` use that to save the keyed inner and outer
states instead of rehashing the key block on every call. `UnmarshalBinary`
refuses a state of the wrong length, a foreign magic or an inconsistent buffer.

```go
h := hmac.New(whirlpool.New, key)
dk, err := pbkdf2.Key(whirlpool.New, password, salt, 500000, 64)
```

## Verification

- The ISO/IEC 10118-3 examples of the reference implementation, including the
  one-million-`a` vector.
- Botan's `whirlpool.vec`.
- A differential corpus of 500 messages hashed by the Botan command line, built
  by `internal/genbotan`; `openssl dgst -whirlpool` agrees on `abc`.
- Fuzz tests for chunked writes and for `UnmarshalBinary`.
- 100 % statement coverage, with the race detector, on amd64 and arm64, and
  under QEMU on big-endian s390x and on riscv64.

## Performance

Against Botan 3.14.1 on one core of an Apple M4 Max (arm64), interleaved with
`abbench --max-load 8`, since v0.3.0:

| per call | this package | Botan 3.14.1 | ratio |
|---|---|---|---|
| 1 KiB | 2.355 µs ± 1 % | 2.507 µs ± 5 % | 1.06x faster |
| 64 KiB | 140.1 µs ± 1 % | 149.6 µs ± 24 % | 1.07x faster |

amd64 and the other targets run the portable Go form, which has not been
compared with Botan there. See the
[README](https://github.com/go-encryptions/whirlpool#readme) for the method.
