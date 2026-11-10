# cbc — AES-CBC for disk encryption

`github.com/go-encryptions/cbc`

```go
import "github.com/go-encryptions/cbc"
```

Package `cbc` implements AES-CBC for disk encryption, with an arm64 kernel
that decrypts 8 blocks at a time. Its output is byte-identical to
`crypto/cipher`.

## API

```go
c, err := cbc.NewCipher(key) // 16, 24 or 32 bytes

c.Decrypt(dst, src, iv)               // one CBC chain
c.DecryptSectors(dst, src, 512, ivs)  // one chain per sector, one IV each
c.EncryptECB(ivs, positions)          // per-sector IVs: eboiv, essiv
c.Encrypt(dst, src, iv)               // through crypto/cipher (serial)
```

## Why

CBC decryption is parallel. Each plaintext block is `D(C[i]) ^ C[i-1]`, and
every ciphertext block is known before the first one is decrypted. On arm64,
`crypto/cipher`'s CBC decrypter does not use that:
- it decrypts one block at a time through `cipher.Block`;
- it allocates a 512-byte buffer per call.

This package interleaves the AESD rounds of 8 blocks and keeps every round
key in a register, the same kernel shape as [`xts`](xts.md).

Disk formats give each sector its own IV by encrypting its position:
- BitLocker's eboiv is `AES_K(byte offset)`;
- LUKS's essiv is `AES_{SHA-256(K)}(sector)`.

`EncryptECB` computes those IVs 8 at a time, and `DecryptSectors` takes one
IV per sector.

Encryption cannot be parallelised within one chain, because each block needs
the previous ciphertext, so `Encrypt` uses `crypto/cipher`. On architectures
other than arm64, everything goes through `crypto/cipher`, with the same
output.

## Performance

Decryption on one Apple M4 Max core, from 6 interleaved rounds of
`abbench --max-load 8`. The reference is OpenSSL 3.6.5 on the same core
(`openssl speed -decrypt -evp aes-*-cbc -bytes N`):

| | `crypto/cipher` | `cbc` | OpenSSL |
|---|---|---|---|
| AES-128, 512 B | 2.03 GB/s | **18.3 GB/s** (104 %) | 17.6 GB/s |
| AES-128, 4 KiB | 2.59 GB/s | **20.7 GB/s** (101 %) | 20.6 GB/s |
| AES-256, 512 B | 1.92 GB/s | **14.8 GB/s** (107 %) | 13.8 GB/s |
| AES-256, 4 KiB | 2.48 GB/s | **15.3 GB/s** (118 %) | 13.0 GB/s |

`cbc` makes zero allocations per call; `crypto/cipher` makes one.

## Checked

- Every path is compared block for block with `crypto/cipher`: AES-128, -192
  and -256; the 8-block groups and the 1–7 block tail; in place or not; the
  kernel and the portable path. `FuzzDecrypt` does the same on arbitrary
  input.
- Each kernel was broken once on purpose, and the tests failed both times.
- CI runs on amd64, arm64, riscv64, loong64, ppc64le and big-endian s390x,
  with 100 % coverage on each.

Its consumer is [`go-fde/bitlocker`](https://github.com/go-fde/bitlocker)
(AES-CBC, with and without the Elephant diffuser).
