# hmac — HMAC whose keyed state can be wiped

`github.com/go-encryptions/hmac` (v0.1.0)

```go
import "github.com/go-encryptions/hmac"
```

HMAC (RFC 2104) and PBKDF2 (RFC 8018) for Go, written from the
[README](https://github.com/go-encryptions/hmac). `crypto/hmac` keeps its two
padded keys inside digests a program cannot reach, so they stay in memory until
the garbage collector reuses the page. `hmac` precomputes the same two keyed
states, so `Reset` does not hash the pads again, but keeps them in buffers it
owns and zeroes them on request. The gap is described in
[go-theca/theca#74](https://github.com/go-theca/theca/issues/74).

```go
m := hmac.New(sha256.New, key) // implements hash.Hash; key is not retained
defer m.Wipe()                 // zeroes every key-derived byte the package holds

for _, sector := range sectors {
	m.Reset()
	m.Write(sector)
	tag = m.Sum(tag[:0])
}

dk, err := hmac.PBKDF2(sha256.New, password, salt, 500000, 64)
```

The output is byte-for-byte what `crypto/hmac` produces, for every hash. A `MAC`
is not safe for concurrent use. After `Wipe`, `Write`, `Sum` and `Reset` panic.

## What Wipe does, and what it cannot do

| Wipe zeroes | |
|---|---|
| the marshaled inner and outer keyed states | always |
| the pads | when the hash cannot be marshaled |
| the scratch buffer holding the last inner digest | always |
| the two live digests | by restoring the public initial state, when the hash implements `BinaryMarshaler` and `BinaryUnmarshaler` (all of the standard library); `Wipeable` reports it |

Wipe cannot reach copies made by the Go runtime or the hash (`Sum` copies its
state, the garbage collector and stack growth move values), the slice holding
your key, the live digests of a hash that cannot be marshaled (they are only
reset), or swap and core dumps taken before `Wipe`. It removes the long-lived
copies the package owns; it is not a proof that no byte of the key is left in
the process.

**Correctness.** Known-answer vectors from RFC 4231, RFC 6070 and published
PBKDF2-HMAC-SHA-256 vectors; the HMAC-SHA-256 vectors of go-theca/theca
(`testdata/theca_vectors.json`); differential tests and fuzz targets against
`crypto/hmac` and `crypto/pbkdf2`; 100% statement coverage.

**Measured, and where.** Apple M4 Max only, one core, `Reset` + `Write` +
`Sum` with one keyed MAC: 85 ns for 64 B and 1434 to 1468 ns for 4 KiB, against
85 ns and 1461 to 1469 ns for `crypto/hmac`. Zero allocations per operation in
both. The 1 MiB and per-sector rows and other platforms are not yet measured.
