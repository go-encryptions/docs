# streebog — Streebog hash (GOST R 34.11-2012)

`github.com/go-encryptions/streebog`

```go
import "github.com/go-encryptions/streebog"
```

Package `streebog` implements the **Streebog** hash function of GOST R
34.11-2012 (RFC 6986) with 256-bit and 512-bit digests, as standard
[`hash.Hash`](https://pkg.go.dev/hash#Hash) values. It is pure Go, with no cgo
and no dependency beyond the standard library.

Its first consumer is VeraCrypt-volume support in `go-fde/veracrypt`, which
needs Streebog-512 as one of the PRFs of PBKDF2-HMAC when it derives header
keys.

## Byte order

RFC 6986 prints every vector as an integer, most significant byte first: its
Example 1 message `3231...3130` is the string `0123...012` read from the end,
and the digest it prints is the byte reversal of the digest bytes. The package
takes and returns byte strings with the first byte least significant, which is
the order of Botan's `streebog.vec`. The tests assert both views: they reverse
the RFC's strings and check the four examples, and Botan's vectors pass
unchanged. That VeraCrypt uses the same order is to be pinned by
`go-fde/veracrypt`'s known-answer test.

## Not constant-time

The round function is eight 256-entry tables of 64-bit words indexed by state
bytes. Table look-ups use secret-dependent addresses, so it is **not**
constant-time.

## API

```go
func New512() hash.Hash                // Size() == 64, BlockSize() == 64
func New256() hash.Hash                // Size() == 32, BlockSize() == 64
func Sum512(data []byte) [64]byte
func Sum256(data []byte) [32]byte
```

Both also implement `encoding.BinaryMarshaler`, `encoding.BinaryUnmarshaler`,
`encoding.BinaryAppender` and `hash.Cloner`, so `crypto/hmac` and
`crypto/pbkdf2` save the keyed states instead of rehashing the key block on
every call. `UnmarshalBinary` refuses a wrong length, a foreign magic, the other
digest size's state and an inconsistent buffer.

```go
dk, err := pbkdf2.Key(streebog.New512, password, salt, 500000, 64)
```

## Verification

- RFC 6986 section 10, Examples 1 and 2, for 512 and 256 bits.
- Botan's `streebog.vec`.
- A differential corpus of 1000 digests (both sizes) hashed by the Botan
  command line, built by `internal/genbotan`.
- The table-driven round is checked against a direct implementation of S, P and
  L from the RFC.
- Fuzz tests for chunked writes and for `UnmarshalBinary`.
- 100 % statement coverage, with the race detector, on amd64 and arm64, and
  under QEMU on big-endian s390x and on riscv64.

See the [README](https://github.com/go-encryptions/streebog#readme) for the
benchmark against Botan.
