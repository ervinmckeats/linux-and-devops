# Lesson 9. Searching: find and grep

## Goal
Find any file by name, size, age or owner, and any text inside files.

## find: files by attributes

`find` walks the tree live. Syntax: `find WHERE TESTS ACTIONS`.

```bash
# by name (quote the pattern!)
find /etc -name "*.conf"

# case-insensitive
find . -iname "readme*"

# files bigger than 100 MB
find /var -type f -size +100M

# modified more than 30 days ago
find /var/log -type f -mtime +30

# modified in the last 10 minutes
find /tmp -type f -mmin -10

# owned by alice, hide permission errors
find / -user alice 2>/dev/null

# empty directories
find . -type d -empty

# world-writable files
find . -perm -o+w -type f

# do not go deeper
find . -maxdepth 1 -type f

find . -name "*.log" -not -path "./keep/*"
```

Actions:

```bash
# delete (put -delete LAST)
find . -name "*.tmp" -delete

# run a command on all matches
find . -name "*.sh" -exec chmod +x {} +

# safe with spaces in names
find . -name "*.log" -print0 | xargs -0 gzip
```

Always run `find` without `-delete` first and look at the list.

## locate: fast, from a database

```bash
sudo apt install plocate

# refresh the database (also runs daily)
sudo updatedb

locate nginx.conf
```
Fast, but it does not see files created after the last `updatedb`.

## grep: text inside files

```bash
# lines containing error
grep "error" app.log

# ignore case
grep -i "error" app.log

# lines NOT matching
grep -v "debug" app.log

# with line numbers
grep -n "listen" nginx.conf

# count matching lines
grep -c "500" access.log

# recursive
grep -r "password" /etc 2>/dev/null

# only file names
grep -rl "TODO" src/

# whole word only
grep -w "port" config

# 3 lines after, 2 before
grep -A3 -B2 "Traceback" app.log

# extended regex: alternatives
grep -E "error|fail|crit" app.log

# print only matches (IPs)
grep -o "[0-9]\{1,3\}\(\.[0-9]\{1,3\}\)\{3\}" access.log

# config without comments and blank lines
grep -v '^\s*#' sshd_config | grep -v '^$'
```

`ripgrep` (`rg`) is a faster `grep -r` that respects `.gitignore`. Great on your laptop; do not assume it exists on servers.

### Regex basics

| Pattern | Meaning |
|---|---|
| `^` / `$` | start / end of line |
| `.` | any one character |
| `*` | previous item 0+ times |
| `+`, `?` | 1+ times, 0 or 1 time (`-E`) |
| `[0-9]`, `[^a-z]` | set / not in set |
| `\b` | word boundary |
| `(a|b)` | group, alternatives (`-E`) |

Quote patterns in single quotes so the shell does not expand them.

## Practice

1. Find all files over 50 MB on the system.
2. Find files in `/etc` changed in the last 7 days.
3. Show `/etc/ssh/sshd_config` without comments and empty lines.
4. In `/var/log`, find which files mention `ssh` (names only).
5. Find all `.conf` files in `/etc` that contain `Listen` or `listen`.

## Check yourself

- Why must you quote `"*.conf"` in `find -name`?
- `find -delete` before or after the other tests? Why?
- What is the difference between `find` and `locate`?

---

<p align="left">
  <a href="media/09-find-and-grep.mp4"><img width="640" src="media/09-find-and-grep.gif" alt="Lesson 9 animation: Searching: find and grep"></a>
</p>

## Next
[Lesson 10. Redirection and pipes](10-pipes-and-redirection.md)
