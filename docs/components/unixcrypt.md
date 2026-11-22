# unixcrypt — crypt(3) password hashes

`github.com/go-encryptions/unixcrypt`

```go
import "github.com/go-encryptions/unixcrypt"
```

Package `unixcrypt` implements the crypt(3) / Modular Crypt Format password
hashes in pure Go (`CGO_ENABLED=0`): **MD5-crypt** (`$1$`), **SHA-256-crypt**
and **SHA-512-crypt** (`$5$`, `$6$`, Drepper's specification) and **bcrypt**
(`$2a$`, `$2b$`, `$2x$`, `$2y$`). It never calls libc's `crypt(3)`, so the
output is identical on every Go target, big-endian ones included. Its
consumers are `go-puppet/puppet` (`pw_hash()`) and `go-ansible/template`
(`password_hash`).

## API

```go
func MD5Crypt(password, salt string) (string, error)
func SHA256Crypt(password, salt string, rounds int) (string, error)
func SHA512Crypt(password, salt string, rounds int) (string, error)
func Bcrypt(password, prefix string, cost int, salt []byte) (string, error)
func BcryptStrict(password, prefix string, cost int, salt []byte) (string, error)
func BcryptFromMCF(password, prefix, mcfSalt string) (string, error)
```

## Behaviour that matches crypt(3) and `openssl passwd`

- **Salts.** The salt ends at its first `$`, is truncated to 8 characters
  (MD5-crypt) or 16 (SHA-crypt), and the kept part must be `[./A-Za-z0-9]`;
  anything else returns `ErrBadSalt`. SHA-crypt also reads `rounds=N$salt`
  from the salt when the `rounds` argument is `0`.
- **SHA-crypt password length.** The cost is quadratic in the password length,
  so `SHA256Crypt` and `SHA512Crypt` refuse a password above `MaxPasswordLen`
  (511 bytes) with `ErrPasswordTooLong`, as glibc and libxcrypt do.
- **bcrypt.** Only the prefixes `2a`, `2b`, `2x`, `2y` and a cost of 4 to 31
  are accepted, and `BcryptFromMCF` needs exactly two digits for the cost.
  `$2x$` reproduces the pre-2011 crypt_blowfish sign-extension bug so that
  legacy hashes verify; use it for verification only.
- **bcrypt reads the first 72 bytes of the password.** `Bcrypt` ignores the
  rest, as bcrypt itself does; `BcryptStrict` returns `ErrPasswordTooLong`
  instead. A NUL byte inside a bcrypt password does not end it.

## Verification

Known-answer tests against `openssl passwd` (MD5-crypt, SHA-crypt), the
OpenBSD `bcrypt.c` vectors and openwall crypt_blowfish's `wrapper.c` vectors
(`$2x$`, `$2y$`). Every refusal above has a test that fails without it,
checked by mutation.

See the [README](https://github.com/go-encryptions/unixcrypt#readme).
