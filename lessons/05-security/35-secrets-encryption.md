# Lesson 35. Secrets and encryption

## Goal
Store and pass passwords, tokens and keys safely, and encrypt data at rest and in transit.

## Where secrets leak

| Leak | Example | Fix |
|---|---|---|
| Git history | `.env` committed once | `.gitignore`, pre-commit scanners, rotate if leaked |
| Command line | `mysql -pSecret` visible in `ps`, shell history | read from file/env, `read -s` |
| Shell history | `export TOKEN=...` | leading space with `HISTCONTROL=ignorespace`, or files |
| Logs | app logs full request headers | redact in code and log pipeline |
| Images | `COPY .env` into Docker image | runtime secrets, build secrets |
| CI output | `echo $TOKEN` in a pipeline | masked variables, never print |
| World-readable files | `/etc/app/config.yml` with `644` | `600` / `640` with a service group |

## Basic rules

1. **Never in Git.** Not even in a private repo.
2. **Least scope.** One token per app and environment, minimum permissions, expiry.
3. **Rotate** regularly and immediately after any leak or staff change.
4. **Centralize** in a secret manager when you have more than a handful.
5. **Audit** who read what.

## Passing secrets to apps

```bash
# file only the service can read
sudo install -m 600 -o app -g app /dev/null /etc/myapp/env
echo 'DB_PASSWORD=...' | sudo tee /etc/myapp/env > /dev/null
# systemd: EnvironmentFile=/etc/myapp/env

# systemd credentials (newer systemd): not visible in the environment of other processes
# LoadCredential=db_password:/etc/myapp/db_password
# the app reads $CREDENTIALS_DIRECTORY/db_password
```

Environment variables are convenient but are inherited by child processes and can show up in crash dumps and `/proc/PID/environ` (readable by the same user and root). Files with tight permissions or a secret manager are better for high-value secrets.

## Secret managers

| Tool | Notes |
|---|---|
| HashiCorp Vault / OpenBao | self-hosted; dynamic DB credentials, leases, PKI |
| AWS Secrets Manager, SSM Parameter Store | IAM-controlled, rotation |
| GCP Secret Manager, Azure Key Vault | same idea in other clouds |
| Kubernetes Secrets | base64, **not encrypted** by default; enable encryption at rest, or use External Secrets / Sealed Secrets (lesson 51) |
| SOPS + age | encrypted files that **can** live in Git; decrypted at deploy time |

### SOPS with age (secrets in Git, encrypted)

```bash
age-keygen -o ~/.config/sops/age/keys.txt      # prints the public key
cat > .sops.yaml <<'YAML'
creation_rules:
  - path_regex: secrets/.*\.yaml$
    age: age1yourpublickey...
YAML
sops secrets/prod.yaml          # opens an editor; saves encrypted values
sops -d secrets/prod.yaml       # decrypt to stdout
```

Only the values are encrypted, so diffs still show which keys changed.

## Generating secrets

```bash
openssl rand -base64 32
head -c 32 /dev/urandom | base64
python3 -c 'import secrets; print(secrets.token_urlsafe(32))'
```

## Encryption in transit

- TLS everywhere, including internal traffic between services and to databases (`sslmode=require` / `verify-full` for PostgreSQL).
- SSH for admin access; no telnet/FTP.
- WireGuard for site-to-site or admin VPN:

```bash
wg genkey | tee private.key | wg pubkey > public.key
# /etc/wireguard/wg0.conf, then: sudo systemctl enable --now wg-quick@wg0
```

## Encryption at rest

```bash
# full disk / partition: LUKS
sudo cryptsetup luksFormat /dev/sdb1
sudo cryptsetup open /dev/sdb1 secure
sudo mkfs.ext4 /dev/mapper/secure && sudo mount /dev/mapper/secure /mnt/secure
sudo cryptsetup close secure

# single files: age or gpg
age -r age1publickey... -o backup.tar.zst.age backup.tar.zst
age -d -i key.txt backup.tar.zst.age > backup.tar.zst
gpg -c file                     # symmetric, password-based
```

Cloud disks and object storage usually support encryption with provider-managed keys (KMS) — turn it on, it is cheap.

Encrypt **backups** before they leave the server; restic and borg do it by default.

## Integrity

```bash
sha256sum release.tar.gz > release.sha256
sha256sum -c release.sha256
gpg --verify file.sig file
```

Verify downloaded binaries (kubectl, terraform) against published checksums.

## If a secret leaks

1. **Revoke/rotate** the secret immediately. Removing it from Git does not help: it is already copied.
2. Check audit logs for its use.
3. Clean history if needed (`git filter-repo`), force-push, ask collaborators to re-clone.
4. Find how it leaked and add a control (scanner, permission, review).

## Practice

1. Scan a repo for secrets with `gitleaks detect`.
2. Move a service's password from its unit file into an `EnvironmentFile` with `600`.
3. Encrypt a YAML file with SOPS + age, commit it, decrypt it in a different directory.
4. Create a LUKS-encrypted volume on a spare virtual disk.
5. Encrypt a backup archive with `age` and restore from it.

## Check yourself

- Why is deleting a leaked token from Git not enough?
- Are Kubernetes Secrets encrypted by default?
- What does SOPS encrypt, and why does that help code review?

## Next
[Lesson 36. Kernel: sysctl, modules, limits](../06-internals/36-kernel-sysctl-limits.md)
