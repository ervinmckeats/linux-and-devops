# Lesson 62. Dangerous commands

## Goal
Recognize commands that destroy data or lock you out, and build habits that stop accidents before Enter.

**Never run these on a system you care about. Try them only in a VM with a snapshot.**

## Data destruction

| Command | Why it is dangerous | Safer habit |
|---|---|---|
| `rm -rf /` or `rm -rf /*` | deletes everything (modern `rm` protects `/` but not `/*`) | never use `-rf` on a path you did not just `ls` |
| `rm -rf "$DIR/"*` with empty `$DIR` | becomes `rm -rf /*` | `"${DIR:?}"` (lesson 30) |
| `rm -rf ~ /tmp/x` (extra space) | deletes your home | type the path, then add `rm -rf` at the start; use Tab |
| `dd if=... of=/dev/sda` | overwrites the wrong disk | `lsblk` first; double-check `of=` |
| `mkfs.ext4 /dev/sdb` | formats the disk | `lsblk -f`, check mountpoints and data |
| `> important.conf` | truncates the file to zero | `>>` for append; `set -o noclobber` |
| `chmod -R 777 /` / `chown -R user /` | breaks the system and security (sudo, ssh stop working) | never recursive on `/`, `/etc`, `/usr` |
| `mv /etc /tmp` | system unusable | — |
| `find / -name "*.log" -delete` | deletes logs that apps need, and more than you think | run without `-delete` first |
| `rsync --delete src dst` with swapped arguments | mirrors emptiness onto your data | `rsync -n` (dry run) first |
| `git reset --hard`, `git clean -fdx` | loses uncommitted work | `git stash`, `git status` first |
| `git push --force` to a shared branch | rewrites others' history | `--force-with-lease`, protect branches |

## Lockouts

| Command | Effect |
|---|---|
| `ufw enable` without allowing SSH | you are cut off from the server |
| `iptables -P INPUT DROP` over SSH | same |
| `PasswordAuthentication no` before a key login works | locked out |
| broken `/etc/sudoers` | no more sudo — always use `visudo` |
| broken `/etc/fstab` | boot stops in emergency mode — `mount -a` before reboot |
| `chmod 600 /etc/passwd` / wrong perms on `/etc/shadow` | logins fail |
| `passwd -l root` + no other sudo user | no admin access |

Protection: keep a second session open, use `netplan try`, schedule automatic rollback with `at`, keep console access (lessons 26, 28, 33).

## System and resource killers

```bash
:(){ :|:& };:          # fork bomb: infinite processes  → limit with pids.max / TasksMax
cat /dev/zero > /dev/sda   # wipe disk
echo c > /proc/sysrq-trigger   # crash the kernel immediately
kill -9 1              # kill PID 1 → kernel panic
```

## Running code from the internet

```bash
curl https://example.com/install.sh | sudo bash
```

You run unknown code as root. Download, read, verify checksum/signature, then run. Same for "copy this command from a forum": read every part before pressing Enter.

Look-alike traps: text copied from a web page can contain hidden characters or a newline that executes the command as soon as you paste it. Paste into an editor first.

## Kubernetes and cloud

| Command | Risk |
|---|---|
| `kubectl delete ns prod` | deletes everything in the namespace, including PVCs |
| `kubectl delete pods --all -A` | restarts the world |
| `kubectl apply` in the wrong context | prod changed instead of dev — show context in the prompt |
| `helm uninstall` of a release with PVCs | data may be gone with them |
| `terraform destroy` / an unchecked `-/+` replace in a plan | deletes databases and disks |
| `aws s3 rm s3://bucket --recursive` | deletes all objects (versioning helps) |
| `docker system prune -a --volumes` | deletes all unused images and **volumes with data** |
| `docker compose down -v` | deletes the project's volumes |

## Habits that prevent accidents

1. **Look before you destroy**: `ls`, `echo`, `--dry-run`, `-n`, `terraform plan`, `kubectl diff`.
2. **Know where you are**: `hostname`, `pwd`, current `kubectl` context and cloud account in your prompt. Color the prompt red on production.
3. **Snapshots and backups** before risky changes, and tested restores.
4. **Least privilege**: use read-only credentials for looking around; escalate only to change.
5. **Two-person rule** for irreversible production actions; changes through PRs.
6. **Slow down** when tired or under pressure. Most disasters are typed at 3 a.m.
7. **Aliases with confirmation** on your own machine: `alias rm='rm -I'`, `alias cp='cp -i'`, `alias mv='mv -i'` (do not rely on them on servers).

## Practice

In a VM with a snapshot:
1. Create a test directory and simulate the empty-variable `rm` bug safely with `echo rm -rf "$EMPTY/"*`. Then fix it with `${EMPTY:?}`.
2. Enable `set -o noclobber` and try to overwrite a file with `>`.
3. Lock yourself out with ufw, recover via the console.
4. Run a fork bomb inside `systemd-run --scope -p TasksMax=50 bash` and see that the host survives.
5. Set up a prompt that shows the host name and current Kubernetes context.

## Check yourself

- Why is `rm -rf "$DIR/"*` dangerous, and how do you fix it?
- What three things do you check before `dd`?
- Name two ways to avoid locking yourself out when changing a firewall over SSH.

## Next
[Lesson 63. Real-world scenarios](63-real-world-scenarios.md)
