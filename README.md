# jie-suo

Public key file encryption using `age(1)`

## Setup

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
export JIE_SUO_PUBKEY='age1yyzyyz'
```

Encrypt and save `foo.json.age` to cwd:

```bash
jie-suo lock /tmp/foo.json
```

******

## Decrypt

Export private key file name to environment:

```bash
export JIE_SUO_PRIVATE_KEY_FILE='/home/someguy/secret.txt'
```

Decrypt and save `foo.json` to cwd:

```bash
jie-suo unlock foo.json.age
```

Your private key is permanently stored in a password safe. For script execution, copy it to a temporary location, set `JIE_SUO_PRIVATE_KEY_FILE` to that path, then shred it when finished.
