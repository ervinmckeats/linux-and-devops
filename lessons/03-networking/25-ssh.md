# Lesson 25. SSH: keys, config, scp, rsync, tunnels

## Goal
Log in to servers securely with keys, move files, and reach private services through tunnels.

## Theory: how key login works

- You have a **key pair**: private key (stays on your machine, `~/.ssh/id_ed25519`) and public key (`.pub`, safe to share).
- The server keeps allowed public keys in `~/.ssh/authorized_keys` of the target user.
- On login the server sends a challenge; your client signs it with the private key; the server checks the signature with the public key. **The private key never leaves your machine.**
- The server also has a **host key**. On first connect you see its fingerprint; it is saved in `~/.ssh/known_hosts`. If it changes later, SSH warns you: either the server was rebuilt, or someone is in the middle.

## Keys

```bash
ssh-keygen -t ed25519 -C "laptop-2026"
ssh-copy-id user@server
ssh user@server
ssh -i ~/.ssh/other_key user@server
eval "$(ssh-agent -s)"; ssh-add ~/.ssh/id_ed25519
ssh-keygen -lf ~/.ssh/id_ed25519.pub
ssh-keygen -R server
```

- `ssh-keygen -t ed25519 -C "laptop-2026"`: create (use a passphrase)
- `ssh-copy-id user@server`: install the public key on the server
- `ssh user@server`: log in
- `eval "$(ssh-agent -s)"; ssh-add ~/.ssh/id_ed25519`: type the passphrase once
- `ssh-keygen -lf ~/.ssh/id_ed25519.pub`: fingerprint
- `ssh-keygen -R server`: remove an old host key from known_hosts

Permissions matter; SSH refuses keys that others can read:

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/id_ed25519 ~/.ssh/authorized_keys
```

## ~/.ssh/config

```
Host web1
    HostName 203.0.113.10
    User deploy
    Port 2222
    IdentityFile ~/.ssh/id_ed25519

Host db-*
    User admin
    ProxyJump bastion

Host bastion
    HostName bastion.example.com
    User admin

Host *
    ServerAliveInterval 30
    AddKeysToAgent yes
```

Now `ssh web1` and `ssh db-1` just work. `ProxyJump` goes through the bastion host in one step (`ssh -J bastion db-1` on the command line).

## Run commands and copy files

```bash
ssh web1 'uptime; df -h /'
ssh web1 'sudo systemctl restart nginx'
scp file.txt web1:/tmp/
scp -r web1:/var/log/nginx ./logs
rsync -avzP ./site/ web1:/var/www/site/
for h in web1 web2 web3; do ssh "$h" 'hostname; uptime'; done
```

- `rsync -avzP ./site/ web1:/var/www/site/`: prefer rsync for anything big or repeated

## Tunnels

**Local forward**: reach a remote private service from your laptop:

```bash
ssh -L 5432:localhost:5432 db-1
ssh -L 8080:10.0.1.20:80 bastion
```

- `ssh -L 5432:localhost:5432 db-1`: then `psql -h localhost`
- `ssh -L 8080:10.0.1.20:80 bastion`: reach an internal web UI through bastion

**Remote forward**: expose your local port on the server:

```bash
ssh -R 9000:localhost:3000 web1
```

- `ssh -R 9000:localhost:3000 web1`: web1:9000 → your laptop:3000

**Dynamic (SOCKS proxy)**:

```bash
ssh -D 1080 bastion
```

- `ssh -D 1080 bastion`: set browser SOCKS5 proxy to localhost:1080

Add `-N` (no shell) and `-f` (background) for tunnels only.

## Keep sessions alive

- Use `tmux` on the server for long jobs (lesson 14).
- `ServerAliveInterval 30` in config against idle disconnects.
- `mosh` for unstable mobile connections.

## Troubleshooting

```bash
ssh -v user@server
sudo journalctl -u ssh -f
sudo sshd -t
```

- `ssh -v user@server`: `-vvv` for more
- `sudo journalctl -u ssh -f`: on the server (unit is "sshd" on RHEL)
- `sudo sshd -t`: test server config before restart

Typical causes: wrong permissions on `~/.ssh`, key in the wrong user's `authorized_keys`, `PasswordAuthentication no` without a key, firewall, wrong port.

Hardening the SSH server is in lesson 33.

## Practice

1. Create an ed25519 key with a passphrase, install it on your VM, log in without a password.
2. Write a `~/.ssh/config` entry so `ssh lab` works.
3. Use a local tunnel to open a service that listens only on the VM's `127.0.0.1`.
4. Set up two VMs: reach the second only through the first with `ProxyJump`.
5. Break `~/.ssh` permissions (`chmod 777`), read the error with `ssh -v` and the server log, then fix.

## Check yourself

- Why is it safe to share your public key but not the private one?
- What does the "REMOTE HOST IDENTIFICATION HAS CHANGED" warning mean and what do you do?
- Local vs remote port forwarding: which direction is each?

## Next
[Lesson 26. Firewalls](26-firewalls.md)
