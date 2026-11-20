# Lesson 13. Permissions, special bits, umask, ACL

## Goal
Read and set permissions correctly, and fix "Permission denied" in minutes.

## Theory

Each file has an **owner**, a **group** and three permission sets: **u**ser (owner), **g**roup, **o**thers.

```
-rwxr-x---  alice  devs  deploy.sh
 └┬┘└┬┘└┬┘
  u  g  o
```

| | File | Directory |
|---|---|---|
| `r` (4) | read content | list names (`ls`) |
| `w` (2) | change content | create, delete, rename entries inside |
| `x` (1) | run as program | enter (`cd`) and access files inside |

Key facts:
- Deleting a file needs `w` on the **directory**, not on the file.
- To reach `/a/b/c/file` you need `x` on every directory in the path.
- The kernel checks **one** set: owner if you are the owner, else group if you are in it, else others.

### Numeric mode

Add the numbers: `rwx`=7, `rw-`=6, `r-x`=5, `r--`=4.

| Mode | Typical use |
|---|---|
| `644` | normal files |
| `755` | directories, scripts |
| `600` | private keys, secrets |
| `700` | `~/.ssh`, private directories |
| `640` | configs readable by a service group |

Never use `777`. If it "fixes" a problem, you have not found the problem.

### Special bits

| Bit | Number | On file | On directory |
|---|---|---|---|
| SUID | 4000 | runs as file **owner** (`passwd` runs as root) | - |
| SGID | 2000 | runs as file group | new files inherit the directory's group |
| Sticky | 1000 | - | only owner can delete own files (`/tmp`) |

`ls -l` shows them as `s` / `t`: `-rwsr-xr-x` (SUID), `drwxrwsr-x` (SGID), `drwxrwxrwt` (sticky).

### umask

Default permissions = `666` for files / `777` for dirs **minus** umask. umask `022` → files `644`, dirs `755`. umask `027` → `640` / `750`.

### ACL

When owner/group/others is not enough (two groups need different access):

```bash
setfacl -m u:bob:r report.txt
setfacl -m g:qa:rx /srv/app
setfacl -d -m g:qa:rx /srv/app
getfacl report.txt
```

- `setfacl -m u:bob:r report.txt`: give bob read
- `setfacl -m g:qa:rx /srv/app`: give group qa read+enter
- `setfacl -d -m g:qa:rx /srv/app`: default for new files inside
`ls -l` shows `+` at the end of the mode when ACLs exist.

## Commands

```bash
chmod 640 file
chmod u+x,g-w,o= script.sh
chmod -R u=rwX,go=rX dir/
chown alice:devs file
chown -R www-data: /var/www/site
chgrp devs file
umask; umask 027
find / -perm -4000 -type f 2>/dev/null
namei -l /var/www/site/index.html
lsattr; sudo chattr +i file
```

- `chmod u+x,g-w,o= script.sh`: symbolic
- `chmod -R u=rwX,go=rX dir/`: X = x only for directories (and already-executable files)
- `chown -R www-data: /var/www/site`: owner and its primary group
- `find / -perm -4000 -type f 2>/dev/null`: all SUID files (security audit)
- `namei -l /var/www/site/index.html`: permissions of every path component
- `lsattr; sudo chattr +i file`: immutable: even root cannot change it until `-i`

### Debugging "Permission denied"

1. `id`: who am I, which groups?
2. `namei -l /full/path`: is `x` missing on some directory?
3. `ls -l` on the file: which set applies to me?
4. ACL (`getfacl`), immutable bit (`lsattr`), mount options (`mount | grep noexec`).
5. SELinux/AppArmor (lesson 34): `ausearch -m avc`, `dmesg`.

## Practice

1. Create `/srv/shared` where group `devs` can create files, every new file belongs to `devs`, and users cannot delete each other's files (SGID + sticky).
2. Make a file only you can read. Prove another user cannot `cat` it.
3. Remove `x` from a directory and try to `cd` into it and `cat` a file inside.
4. List all SUID binaries on your system.
5. Give one extra user read access to a file with ACL without changing the group.

## Check yourself

- Why can a user delete a file they cannot write to?
- What does `chmod -R 755` do wrong to files, and how does `X` fix it?
- Which umask gives new files `640`?

## Next
[Lesson 14. Processes and signals](14-processes-signals.md)
