# ctr — AES-CTR

`github.com/go-encryptions/ctr` (v0.1.1)

```go
import "github.com/go-encryptions/ctr"
```

AES in counter mode for a stream of messages under one key, written from the
[README](https://github.com/go-encryptions/ctr). `crypto/cipher.NewCTR` builds its
state for every message and, on arm64 and amd64, reloads the round keys into
vector registers for every 128 bytes. `ctr` expands the key once, loads it into
registers once per call, keeps eight blocks in flight, and allocates nothing.

```go
c, err := ctr.New(key) // 16, 24 or 32 bytes; expands the key once
if err != nil {
	return err
}
defer c.Zeroize() // the round keys live in c, so they can be wiped

var iv [16]byte // the counter block; XORKeyStream does not change it
c.XORKeyStream(dst, src, &iv) // dst == src, or no overlap at all
```

The counter is the whole 128-bit block, incremented big-endian, as in
`crypto/cipher.NewCTR` and NIST SP 800-38A. The output is byte-for-byte what the
standard library produces, on every architecture. A `Cipher` is safe for
concurrent use.

| Platform | Kernel |
|---|---|
| arm64 | ARMv8 `AESE`/`AESMC`, eight blocks, keys resident in V0-V14 |
| amd64 | AES-NI (VEX encodings), eight blocks; needs AES and AVX |
| anything else | `crypto/aes` + `crypto/cipher` |

**Correctness.** NIST SP 800-38A known-answer vectors, differential tests against
`crypto/cipher` for every length from 0 to 530 bytes, all three key sizes, a fuzz
target, and 100% statement coverage on every architecture CI runs.

**Measured, and where.** Apple M4 Max only, one core, 4 KiB message: `ctr` 306 ns,
`crypto/cipher.NewCTR` (key schedule reused) 444 ns. The README gives no amd64
(Zen 4) figure yet; none is stated here.

Used by [theca](https://go-theca.github.io/docs/) v0.7.0 for sector encryption.
