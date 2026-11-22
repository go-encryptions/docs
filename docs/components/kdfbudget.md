# kdfbudget — a bound on the total key-derivation work

`github.com/go-encryptions/kdfbudget` (v0.1.0)

```go
import "github.com/go-encryptions/kdfbudget"
```

A header read from an image is untrusted. Limiting each KDF run is not enough:
a hostile header can hold many runs (key slots times digests, keybag lockers,
every KDF a format tries in turn), each one within its limit, and the sum is
unbounded: 3155 runs for any password in LUKS
([go-fde/luks#32](https://github.com/go-fde/luks/issues/32)), 888 with no
passphrase in APFS ([go-fde/apfs#35](https://github.com/go-fde/apfs/issues/35)).
A `Budget` is made for one unlock and every run is charged **before** it is
spent.

```go
b := kdfbudget.Default() // or Unlimited() for a header you trust

if err := b.ChargePBKDF2(kdfbudget.CostSHA512, slot.Iterations); err != nil {
	return err // errors.Is(err, kdfbudget.ErrLimit); nothing was spent
}
key := pbkdf2(slot) // only now
```

| | |
|---|---|
| `ChargePBKDF2(hashBlockCost, iters)` | one output block |
| `ChargeArgon2(memKiB, passes, threads)` | also refuses 0 passes, 0 or more than 255 threads, under 8 KiB per thread |
| `ChargeCompressions(hashBlockCost, n)` | any other hash chain (BitLocker's stretch) |
| `Budget{MaxMemory, MaxThreads, MaxWork, Deadline}` | safe for concurrent use; the zero value and nil refuse everything |

## Cost model

One unit is the compression of one 64-byte block by a plain hash. PBKDF2 costs
`2 * iterations * hashBlockCost` (SHA-1, SHA-256, RIPEMD-160 and BLAKE2s 1,
SHA-512 2, Whirlpool and Streebog 4); Argon2 costs `passes * memKiB * 8`;
memory is a peak, checked per run. The constants are conservative round
estimates: the budget is a denial-of-service bound accurate to a small factor,
not a timer.

## Defaults

4 GiB of memory (cryptsetup's Argon2 ceiling), 255 threads, 2^32 units (about
three minutes of one core). Tests assert that they accept a 4 GiB Argon2, 32
LUKS2 slots of 1 GiB x 4 passes, OpenZFS's minimum `pbkdf2iters` (100000, no
maximum) and 2^31, BitLocker's fixed stretch and every VeraCrypt variant at PIM
2000, and refuse the 17th LUKS slot of 2^26 SHA-512 iterations and the 129th
4 GiB Argon2 locker.

**Tested.** 100% statement coverage, a fuzz target on the charge arithmetic
(overflow, thread truncation, exact cost) and a 32-bit lane.
