# jie-suo

Public key file encryption using `age(1)`.

## Overview

- `jie-suo lock <file>` encrypts a file to a public key (`$JIE_SUO_PUBKEY`)
- `jie-suo unlock <file.age>` decrypts a file using a private key (`$JIE_SUO_PRIVATE_KEY_FILE`)
- Uses the `age` CLI tool for encryption/decryption

## Key Conventions

- `age` commands are invoked with `if !` to capture exit codes directly
- `stat -f '%Lp'` (BSD/macOS) returns hex; converted to octal for comparison
- Output files are always written to cwd, not the input file's directory
- Private key file must have `600` permissions

## Testing

```bash
shellcheck jie-suo
bash -n jie-suo
```

## Setup

```bash
sudo cp jie-suo /usr/local/bin && \
sudo chown root /usr/local/bin/jie-suo && \
sudo chmod 555 /usr/local/bin/jie-suo
```

## Dependencies

- `age` must be installed and in PATH
- Bash 4+ (for `[[ =~ ]]` regex matching)
