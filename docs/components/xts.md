# xts — XTS-AES for disk encryption

`github.com/go-encryptions/xts`

```go
import "github.com/go-encryptions/xts"
```

Package `xts` implements the **XTS** tweakable block-cipher mode (IEEE P1619,
[NIST SP 800-38E](https://csrc.nist.gov/pubs/sp/800/38/e/final)). It is the
sector-level mode that LUKS, APFS FileVault 2, BitLocker, VeraCrypt and
dm-crypt use.

It is a drop-in replacement for `golang.org/x/crypto/xts`: same `NewCipher` /
`Encrypt` / `Decrypt` API, byte-identical output. The difference is speed.
`x/crypto/xts` drives the block cipher one 16-byte block at a time through
`cipher.Block`. This package has a fused AES-XTS kernel that pipelines several
blocks and folds the tweak XOR into the rounds.

## API

```go
// key is two keys of the block cipher, concatenated (32, 48 or 64 bytes for AES).
c, err := xts.NewCipher(aes.NewCipher, key)

c.Encrypt(ciphertext, plaintext, sectorNum) // one sector; sectorNum is the tweak
c.Decrypt(plaintext, ciphertext, sectorNum)

// Many consecutive sectors in one call: sector firstSector+i is
// ciphertext[i*512:(i+1)*512].
c.DecryptSectors(plaintext, ciphertext, 512, firstSector)
c.EncryptSectors(ciphertext, plaintext, 512, firstSector)
```

- Buffers must be a whole number of 16-byte blocks, at least one block long:
  the `x/crypto/xts` contract.
- `DecryptSectors` and `EncryptSectors` give exactly what `Decrypt` and
  `Encrypt` give sector by sector. On arm64 they encrypt the tweaks of 8
  sectors in one pass, instead of making each sector wait for its own tweak.
- None of these allocate.
- Any `cipher.Block` constructor works. Non-AES ciphers (Twofish, Serpent)
  take the portable per-block path.

## Performance

- On amd64 (AES-NI) and arm64 (ARMv8 AES + NEON), the kernel is Plan 9
  assembly. On arm64 it takes 8 blocks at a time, with every round key in a
  register.
- Measured on one Apple M4 Max core against OpenSSL 3.6.5
  (`openssl speed -evp aes-*-xts`), with 6 interleaved rounds of
  `abbench --max-load 8`. Each figure is the share of OpenSSL's decryption
  throughput:

| | `Decrypt` (one sector per call) | `DecryptSectors` (1 MiB per call) |
|---|---|---|
| AES-128, 512 B | 86 % | **112 %** |
| AES-128, 4 KiB | 99 % | **102 %** |
| AES-256, 512 B | 92 % | **117 %** |
| AES-256, 4 KiB | **103 %** | **107 %** |

- riscv64, loong64, ppc64le and s390x use the same per-block algorithm as
  `x/crypto/xts`. The output is identical; only the speed differs.

## Guarantees

- Byte-identical to `golang.org/x/crypto/xts` on every architecture. CI
  checks this, with x/crypto as the reference.
- No cgo, and no dependencies beyond `golang.org/x/sys/cpu` (feature
  detection) and `golang.org/x/crypto` (tests only).
- Built and tested on all six 64-bit Go targets, natively or under QEMU, with
  100 % statement coverage.

Consumers include [`go-fde/luks`](https://github.com/go-fde/luks),
[`go-fde/apfs`](https://github.com/go-fde/apfs),
[`go-fde/bitlocker`](https://github.com/go-fde/bitlocker) and
[`go-fde/veracrypt`](https://github.com/go-fde/veracrypt).
