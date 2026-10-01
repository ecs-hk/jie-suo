# jie-suo

Public key file encryption using `age(1)`

## Setup

### Install `age`

`jie-suo` shells out to `age(1)`, which must already be installed.

`jie-suo` does **not** use your `PATH` — that is deliberate, so that a
trojan `age` earlier in your `PATH` can never be executed. It looks in these
four directories only:

```
/usr/bin  /bin  /usr/local/bin  /opt/homebrew/bin
```

The official `age` release installs to `/usr/local/bin` and Homebrew installs
to `/opt/homebrew/bin`, so both work out of the box. If you install `age`
anywhere else, move or symlink it into one of the four.

### Install script

After cloning the repository:

```bash
sudo cp jie-suo /usr/local/bin && \
sudo chown root /usr/local/bin/jie-suo && \
sudo chmod 555 /usr/local/bin/jie-suo
```

### Generate keypair

```bash
age-keygen
```

Permanently store your private key in a reputable password safe (e.g. KeePassXC).

******

## Encrypt

Export public key to environment, e.g. in `~/.bash_profile`:

```bash
export JIE_SUO_PUBKEY='age1w2fhwhdncy93pru7axc3wp74yfqmdv5wc0a6707my2f8zmat0pmq9xpxmy'
```

Replace that with the output of `age-keygen -y` for your own keypair. An `age`
public key is `age1` followed by 58 bech32 characters; anything else is
rejected before `age` is ever called. Surrounding whitespace and newlines are
stripped, so this also works:

```bash
export JIE_SUO_PUBKEY="$(age-keygen -y key.txt)"
```

Encrypt and save `foo.json.age` to cwd:

```bash
jie-suo lock /tmp/foo.json
```

Note that:

- the output is always written to your **current directory**, not next to the
  input file
- the output is created with `600` permissions
- an existing output file is never overwritten — delete it first
- the ciphertext is staged via `mktemp` as `<name>.age.tmp.XXXXXX` and
  published with `mv -n` only when `age` succeeds, so `foo.json.age` either
  does not exist or is complete — an existing output (including a dangling
  symlink) is never overwritten, and a leftover `.tmp.` file only happens
  if `jie-suo` is killed with `SIGKILL`

******

## Decrypt

Your private key is permanently stored in a password safe. For script
execution, copy it to a temporary location and point the environment variable
at that copy.

The copy **must** have exactly `600` permissions, otherwise `jie-suo` refuses
to use it. Note that `cp` on its own leaves the copy at `644` under a default
`umask`, so the `chmod` is not optional:

```bash
cp ~/secrets/age.key /tmp/jie-suo.key
chmod 600 /tmp/jie-suo.key
export JIE_SUO_PRIVATE_KEY_FILE='/tmp/jie-suo.key'
```

Decrypt and save `foo.json` to cwd:

```bash
jie-suo unlock foo.json.age
```

As with `lock`, the plaintext is written to your **current directory**, is
created with `600` permissions, and is never written over an existing file.
The same staging applies: the plaintext is assembled via `mktemp` under a
temporary name and published with `mv -n` only when decryption succeeds,
so a failed or interrupted run never leaves a partial `foo.json`.

When you are done, remove the temporary key:

```bash
shred -u /tmp/jie-suo.key     # GNU/Linux
rm -P /tmp/jie-suo.key        # BSD/macOS
```

If decryption fails, the incomplete output file is removed for you, so no
partial plaintext is left behind.
