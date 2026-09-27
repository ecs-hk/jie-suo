# jie-suo

Public key file encryption using `age(1)`.

## Overview

- `jie-suo lock <file>` encrypts a file to a public key (`$JIE_SUO_PUBKEY`)
- `jie-suo unlock <file.age>` decrypts a file using a private key (`$JIE_SUO_PRIVATE_KEY_FILE`)
- Uses the `age` CLI tool for encryption/decryption

## Key Conventions

- `age` is looked up in a fixed `PATH` (`/usr/bin:/bin:/usr/local/bin:/opt/homebrew/bin`); the caller's `PATH` is deliberately not inherited, to avoid executing a trojan `age`
- `age` commands are invoked with `if !` to capture exit codes directly
- `umask 077` is set at startup, so every output file is `600` from creation — never `chmod` plaintext after the fact
- `stat -c '%a'` (GNU) and `stat -f '%Lp'` (BSD/macOS) **both** print permission bits in octal; no conversion is needed. A value like `81ed` is a full mode including file-type bits (`0x81ed` == `0100755`), not the output of `%Lp`
- Output files are always written to cwd, not the input file's directory, and are never overwritten
- On `age` failure the incomplete output is removed unconditionally (not just when empty), so no truncated ciphertext or partial plaintext is left behind. An `INT`/`TERM`/`HUP` trap covers interruption
- Private key file must have exactly `600` permissions; `400` is rejected by design
- Argument count is validated before anything else, and `PATH` for `age` is checked after

## Code Style

- Source code wraps at **80 columns maximum**; the current ceiling is 79
- **Never auto-wrap shell.** `fmt`, `fold` and friends corrupt backslash
  continuations (see the aligned `age` invocation) and quoting. Detect with
  the `awk` check below, then fix by hand
- Multi-word `errout` messages may be split across lines with `\` — `errout`
  joins its arguments with a space

## Testing

```bash
shellcheck jie-suo
bash -n jie-suo
awk 'length > 80 { print FILENAME ":" FNR " (" length " chars)"; n=1 } END { exit n }' jie-suo
```

All three must be clean. `shellcheck` has no line-length check of its own, so
the `awk` one-liner is what actually enforces the column limit; it reports
every offender and exits non-zero, so it composes with `&&`.

Functional checks are manual — there is no test harness in the repo:

```bash
# round trip
export JIE_SUO_PUBKEY="$(age-keygen -y key.txt)"
./jie-suo lock foo.json && ./jie-suo unlock foo.json.age

# private key permission matrix: 400, 644 and 4600 must all be rejected
chmod 400 key.txt ; ./jie-suo unlock foo.json.age

# output must be 600, and failures must leave no file behind
stat -c '%a' foo.json.age
```

## Setup

```bash
sudo cp jie-suo /usr/local/bin && \
sudo chown root /usr/local/bin/jie-suo && \
sudo chmod 555 /usr/local/bin/jie-suo
```

## Dependencies

- `age` must be installed in one of the four `PATH` directories above
- `bash` (the shebang is `#!/bin/bash`; POSIX `sh` is not sufficient)
