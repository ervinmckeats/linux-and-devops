# Lesson 29. Bash scripting basics

## Goal
Turn repeated commands into scripts with variables, conditions, loops and functions.

## First script

```bash
#!/usr/bin/env bash
# hello.sh - greet a user
set -euo pipefail

name="${1:-world}"
echo "Hello, $name"
```

```bash
chmod +x hello.sh
./hello.sh Alice
bash hello.sh       # also works without +x
```

- `#!` (shebang) says which interpreter runs the file. `/usr/bin/env bash` finds bash in `PATH`.
- `set -euo pipefail` — stop on error, on unset variable, on failure anywhere in a pipeline. Start every script with it (details and caveats in lesson 30).

## Variables

```bash
name="Alice"          # NO spaces around =
echo "$name"          # always quote variables
echo "${name}_backup" # braces when text follows
readonly VERSION=1.2
today=$(date +%F)     # command output
count=$((count + 1))  # arithmetic
export DB_HOST=db1    # visible to child processes
```

Special variables:

| Var | Meaning |
|---|---|
| `$0` | script name |
| `$1`..`$9`, `${10}` | arguments |
| `$#` | number of arguments |
| `"$@"` | all arguments, each one quoted separately (use this) |
| `$?` | exit code of last command |
| `$$` | PID of the script |

**Why quotes matter:** `file="my report.txt"; rm $file` tries to remove `my` and `report.txt`. `rm "$file"` removes the right one.

## Conditions

```bash
if [[ -f /etc/nginx/nginx.conf ]]; then
    echo "nginx config exists"
elif [[ -d /etc/apache2 ]]; then
    echo "apache found"
else
    echo "no web server" >&2
    exit 1
fi
```

| Test | True when |
|---|---|
| `-f file` / `-d dir` / `-e path` | regular file / directory / anything exists |
| `-r` / `-w` / `-x` | readable / writable / executable |
| `-s file` | not empty |
| `-z "$s"` / `-n "$s"` | string empty / not empty |
| `"$a" == "$b"`, `!=` | strings equal / differ |
| `"$s" == *.log` | glob match (inside `[[ ]]`) |
| `"$s" =~ ^[0-9]+$` | regex match |
| `$a -eq $b`, `-ne -lt -le -gt -ge` | numbers |
| `&&`, `||`, `!` | and, or, not |

Use `[[ ]]` in bash scripts; `[ ]` is the older POSIX form with more quoting traps. For numbers `(( a > b ))` also works.

The `if` checks an **exit code**, so any command works: `if grep -q error app.log; then ...`.

### case

```bash
case "$1" in
    start)   systemctl start app ;;
    stop)    systemctl stop app ;;
    restart) systemctl restart app ;;
    *)       echo "usage: $0 {start|stop|restart}" >&2; exit 2 ;;
esac
```

## Loops

```bash
for host in web1 web2 web3; do
    ssh "$host" uptime
done

for f in /var/log/*.log; do
    [[ -e "$f" ]] || continue        # no match → skip the literal pattern
    echo "$f: $(wc -l < "$f") lines"
done

for i in {1..5}; do echo "$i"; done
for ((i = 0; i < 3; i++)); do echo "$i"; done

while read -r user _ uid _; do
    echo "$user $uid"
done < <(tr ':' ' ' < /etc/passwd)

until curl -sf http://localhost:8080/health; do
    echo "waiting..."; sleep 2
done
```

`read -r` keeps backslashes; `while read` reads a file line by line correctly (do not use `for line in $(cat file)`).

## Functions

```bash
log() {
    echo "[$(date +%T)] $*" >&2
}

check_port() {
    local host=$1 port=$2       # local: do not leak into global scope
    if timeout 2 bash -c "</dev/tcp/$host/$port" 2>/dev/null; then
        log "$host:$port open"
        return 0
    fi
    log "$host:$port closed"
    return 1
}

check_port localhost 22 || exit 1
```

Functions return an exit code (0–255). To return data, `echo` it and capture: `result=$(myfunc)`.

## Input

```bash
read -rp "Continue? [y/N] " answer
[[ "$answer" =~ ^[Yy]$ ]] || exit 0
read -rsp "Password: " pass; echo     # -s: hidden input
```

## Aliases and your shell

```bash
# ~/.bashrc
alias ll='ls -lah'
alias k='kubectl'
export EDITOR=vim
```
`source ~/.bashrc` to apply. Aliases are for interactive use; do not rely on them in scripts.

## A complete small script

```bash
#!/usr/bin/env bash
# disk-alert.sh - warn when a filesystem is above a threshold
set -euo pipefail

THRESHOLD="${1:-80}"
status=0

while read -r use mount; do
    use=${use%\%}
    if (( use >= THRESHOLD )); then
        echo "WARNING: $mount is at ${use}%" >&2
        status=1
    fi
done < <(df --output=pcent,target -x tmpfs -x devtmpfs | tail -n +2)

exit "$status"
```

## Practice

1. Write `backup.sh SRC DST` that checks both arguments exist, then creates `DST/name-YYYY-MM-DD.tar.gz`.
2. Write `svc.sh {status|restart} NAME` with `case`.
3. Write a loop that pings hosts from `hosts.txt` and prints `UP`/`DOWN` for each.
4. Write a function `retry N CMD...` that runs a command up to N times with a 2-second pause.
5. Run `shellcheck` on all your scripts (`apt install shellcheck`) and fix every warning.

## Check yourself

- Why `"$@"` and not `$*`?
- What does `local` do in a function?
- Why is `for line in $(cat file)` wrong?

## Next
[Lesson 30. Bash for professionals](30-bash-pro.md)
