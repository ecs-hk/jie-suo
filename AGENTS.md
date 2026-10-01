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
- Output is staged via `mktemp` to `<name>.tmp.XXXXXX` (`O_EXCL`) in cwd and published with `mv -n`, so the final name only ever exists with complete contents and is never overwritten on a race; a leftover `*.tmp.*` can only come from SIGKILL and never blocks a rerun (random suffix)
- On `age` failure the staged temp file is removed unconditionally (not just when empty), so no truncated ciphertext or partial plaintext is left behind. The `INT`/`TERM`/`HUP` traps remove the same temp file and exit with 128+signo, so an interruption can never print a success message for missing or partial output
- Every path argument given to an external command (`age`, `stat`, `rm`, `mv`) is preceded by `--`, so dash-leading names are never parsed as options; the sole exception is BSD `stat`, which takes no `--`, so a leading dash is neutralised with a `./` prefix instead (GNU `stat` keeps `--`)
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
# round trip (move the plaintext aside first: unlock refuses to overwrite it)
export JIE_SUO_PUBKEY="$(age-keygen -y key.txt)"
./jie-suo lock foo.json && mv foo.json foo.orig && \
        ./jie-suo unlock foo.json.age && cmp foo.orig foo.json

# private key permission matrix: 400, 644 and 4600 must all be rejected
chmod 400 key.txt ; ./jie-suo unlock foo.json.age

# output must be 600, and failures must leave no file behind
stat -c '%a' foo.json.age

# TERM while 'age' runs: exit 143, no success message, no output, no temp
truncate -s 1G big.bin
./jie-suo lock big.bin & pid=$!
for _ in $(seq 1 300); do ls big.bin.age* >/dev/null 2>&1 && break; sleep 0.01; done
kill -TERM "$pid" ; wait "$pid" ; echo "$?"       # 143
ls big.bin.age big.bin.age.tmp.*                  # neither exists

# INT to the whole process group (Ctrl-C equivalent) must exit 130:
# run the lock under 'set -m', then kill -INT -"$(ps -o pgid= -p "$pid")"

# dash-leading names, a file literally named 'age', and a dangling symlink
# at the output name must all be handled
./jie-suo lock ./-weird
JIE_SUO_PRIVATE_KEY_FILE=-key ./jie-suo unlock ./-weird.age
./jie-suo unlock age                              # I only decrypt .age files
ln -s /tmp/target-x protected.txt
./jie-suo unlock protected.txt.age                # /tmp/target-x untouched
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
