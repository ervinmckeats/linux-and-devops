# Lesson 15. Package managers: apt, dnf, pacman

## Goal
Install, update, pin and remove software the way each distribution expects.

## Theory

A **package** is an archive with files + metadata: version, dependencies, install scripts. A **repository** is a signed server with package lists. The **package manager** resolves dependencies, verifies signatures (GPG) and tracks every installed file, so it can remove them cleanly later.

Install from the distro repository first. Vendor repository second. Source or random `curl | bash` last, and only after reading the script.

## Command map

| Task | Debian/Ubuntu | RHEL/Fedora | Arch |
|---|---|---|---|
| refresh lists | `apt update` | `dnf makecache` | `pacman -Sy` |
| upgrade all | `apt upgrade` | `dnf upgrade` | `pacman -Syu` |
| install | `apt install nginx` | `dnf install nginx` | `pacman -S nginx` |
| remove | `apt remove` / `purge` | `dnf remove` | `pacman -R` / `-Rns` |
| search | `apt search` | `dnf search` | `pacman -Ss` |
| info | `apt show` | `dnf info` | `pacman -Si` |
| which package owns a file | `dpkg -S /usr/bin/ls` | `rpm -qf /usr/bin/ls` | `pacman -Qo` |
| files of a package | `dpkg -L nginx` | `rpm -ql nginx` | `pacman -Ql` |
| list installed | `apt list --installed` | `dnf list installed` | `pacman -Q` |
| clean unused deps | `apt autoremove` | `dnf autoremove` | `pacman -Rns $(pacman -Qdtq)` |
| history | `/var/log/apt/history.log` | `dnf history` | `/var/log/pacman.log` |

On Ubuntu, `purge` also removes config files in `/etc`; `remove` keeps them.

## Debian/Ubuntu details

```bash
# local file, resolves deps
sudo apt install ./package.deb

# candidate version and which repo
apt-cache policy nginx

apt list --upgradable

# do not upgrade this package
sudo apt-mark hold nginx

sudo apt-mark unhold nginx

# exact version
sudo apt install nginx=1.24.0-1ubuntu1

# repositories
ls /etc/apt/sources.list.d/
```

Adding a vendor repository (modern way, key scoped to one repo):

```bash
curl -fsSL https://example.com/key.gpg | sudo gpg --dearmor -o /etc/apt/keyrings/example.gpg
echo "deb [signed-by=/etc/apt/keyrings/example.gpg] https://example.com/apt stable main" \
  | sudo tee /etc/apt/sources.list.d/example.list
sudo apt update
```

## RHEL family details

```bash
# extra packages repo
sudo dnf install epel-release

dnf repolist

# roll back the last transaction
sudo dnf history undo last

# plugin: pin a version
sudo dnf versionlock add nginx
```

## Other formats

- **snap / flatpak**: sandboxed apps with bundled dependencies. Common on desktops; on servers prefer native packages or containers.
- **Language managers** (`pip`, `npm`, `cargo`): never `sudo pip install` into the system Python: use `venv`, `pipx` or containers.
- **Static binaries** (Go tools like `kubectl`, `terraform`): download to `/usr/local/bin`, verify the checksum.

## Practice

1. Find which package provides `/usr/bin/curl`, then list all its files.
2. Install `nginx`, check its version and candidate with `apt-cache policy`.
3. Hold `nginx`, run `apt upgrade`, see it is kept back, unhold.
4. Remove vs purge: install a package, edit its config, `remove`, check `/etc`, then `purge`.
5. Read the last 20 lines of the apt history log.

## Check yourself

- Why should you avoid `curl | sudo bash`?
- What is the difference between `apt remove` and `apt purge`?
- How do you stop one package from upgrading?

---

<p align="left">
  <a href="media/15-package-managers.mp4"><img width="640" src="media/15-package-managers.gif" alt="Lesson 15 animation: Package managers: apt, dnf, pacman"></a>
</p>

## Next
[Lesson 16. systemd and services](16-systemd.md)
