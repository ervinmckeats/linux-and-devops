# Lesson 30. Bash for professionals

## Goal
Write scripts that are safe to run in production: they clean up, handle errors, parse options, and do not destroy data on edge cases.

## Strict mode and its limits

```bash
set -Eeuo pipefail
IFS=$'\n\t'
```

- `-e` exits on error: **but not** inside `if`, `while`, `&&`/`||` chains, or in command substitutions in some contexts. Do not treat it as full error handling.
- `-u` fails on unset variables. Use `${VAR:-default}` for optional ones.
- `-o pipefail` makes `cmd | grep` fail if `cmd` fails.
- `-E` makes `ERR` traps work inside functions.

## Parameter expansion: no need for `sed` or `awk`

```bash
file="/var/log/nginx/access.log.1"
echo "${file##*/}"
echo "${file%/*}"
echo "${file%.*}"
echo "${file/log/LOG}"
echo "${file//log/LOG}"
echo "${#file}"
echo "${VAR:-default}"
echo "${VAR:=default}"
echo "${VAR:?must be set}"
echo "${name^^} ${name,,}"
echo "${str:0:5}"
```

- `echo "${file##*/}"`: `access.log.1` (basename)
- `echo "${file%/*}"`: `/var/log/nginx` (dirname)
- `echo "${file%.*}"`: `/var/log/nginx/access.log` (drop last extension)
- `echo "${file/log/LOG}"`: replace first
- `echo "${file//log/LOG}"`: replace all
- `echo "${#file}"`: length
- `echo "${VAR:-default}"`: default if unset/empty
- `echo "${VAR:=default}"`: also assign
- `echo "${VAR:?must be set}"`: exit with error if unset/empty
- `echo "${name^^} ${name,,}"`: upper / lower case
- `echo "${str:0:5}"`: substring

The most important one: `rm -rf "${BUILD_DIR:?}/"*`: if `BUILD_DIR` is empty, the script stops instead of running `rm -rf /*`.

## Arrays

```bash
hosts=(web1 web2 "db 1")
hosts+=(cache1)
echo "${hosts[0]}"
echo "${#hosts[@]}"
for h in "${hosts[@]}"; do echo "$h"; done

declare -A port=([web]=80 [db]=5432)
port[cache]=6379
for k in "${!port[@]}"; do echo "$k=${port[$k]}"; done

mapfile -t lines < file.txt
```

- `echo "${hosts[0]}"`: first
- `echo "${#hosts[@]}"`: count
- `for h in "${hosts[@]}"; do echo "$h"; done`: quoted: "db 1" stays one item
- `declare -A port=([web]=80 [db]=5432)`: associative array
- `mapfile -t lines < file.txt`: file into array, one line per element

Build commands as arrays to keep arguments safe:

```bash
args=(-avz --delete)
[[ $DRY_RUN == 1 ]] && args+=(-n)
rsync "${args[@]}" src/ dst/
```

## Cleanup with trap

```bash
tmp=$(mktemp -d)
cleanup() { rm -rf "$tmp"; }
trap cleanup EXIT

trap 'echo "error on line $LINENO: $BASH_COMMAND" >&2' ERR
trap 'echo "interrupted"; exit 130' INT TERM
```

- `trap cleanup EXIT`: runs on normal exit, error, or Ctrl+C

## Option parsing with getopts

```bash
usage() { echo "usage: $0 [-n] [-v] -d DIR" >&2; exit 2; }

dry_run=0 verbose=0 dir=""
while getopts ":nvd:h" opt; do
    case $opt in
        n) dry_run=1 ;;
        v) verbose=1 ;;
        d) dir=$OPTARG ;;
        h) usage ;;
        :) echo "-$OPTARG needs a value" >&2; usage ;;
        \?) echo "unknown option -$OPTARG" >&2; usage ;;
    esac
done
shift $((OPTIND - 1))
[[ -n $dir ]] || usage
```

## Locks, parallelism, timeouts

```bash
exec 9>/var/lock/myjob.lock
flock -n 9 || { echo "already running"; exit 1; }

timeout 30 curl -sf http://slow/api || echo "timed out"

for h in "${hosts[@]}"; do
    ssh "$h" uptime > "out/$h.txt" 2>&1 &
done
wait

printf '%s\n' "${hosts[@]}" | xargs -P 10 -I{} ssh {} uptime
```

- `wait`: wait for all background jobs
- `printf '%s\n' "${hosts[@]}" | xargs -P 10 -I{} ssh {} uptime`: max 10 at a time

## Idempotency

A good ops script can run twice without breaking anything:

```bash
mkdir -p /opt/app
id app &>/dev/null || useradd -r -s /usr/sbin/nologin app
grep -qxF 'vm.swappiness=10' /etc/sysctl.conf || echo 'vm.swappiness=10' >> /etc/sysctl.conf
ln -sfn /opt/app/releases/v2 /opt/app/current
```

- `mkdir -p /opt/app`: not mkdir

## Logging and debugging

```bash
log()  { printf '%s [%s] %s\n' "$(date -Is)" "$1" "${*:2}" >&2; }
log INFO "starting"
log ERROR "failed to connect"

bash -n script.sh
bash -x script.sh
set -x; ...; set +x
PS4='+ ${BASH_SOURCE}:${LINENO}: '
shellcheck script.sh
```

- `bash -n script.sh`: syntax check only
- `bash -x script.sh`: trace every command
- `set -x; ...; set +x`: trace a part
- `PS4='+ ${BASH_SOURCE}:${LINENO}: '`: better trace prefix
- `shellcheck script.sh`: static analysis - catches most bugs below

## Classic traps

| Trap | Fix |
|---|---|
| Unquoted `$var` with spaces | `"$var"` |
| `cd dir; rm -rf *` when cd fails | `cd dir || exit 1` |
| `cat file | while read ...; do count++`: count is lost (subshell) | `while ...; done < file` |
| Parsing `ls` output | use globs or `find -print0` |
| `[ $a == $b ]` with empty `$a` | `[[ "$a" == "$b" ]]` |
| Secrets in arguments (visible in `ps`) | env vars or files with 600 |
| Script edited while running | bash reads scripts lazily; deploy a new file and `mv` it |

## Template

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

readonly SCRIPT_NAME=${0##*/}
log() { printf '%s %s: %s\n' "$(date -Is)" "$SCRIPT_NAME" "$*" >&2; }
die() { log "ERROR: $*"; exit 1; }

tmp=$(mktemp -d)
trap 'rm -rf "$tmp"' EXIT
trap 'die "line $LINENO: $BASH_COMMAND"' ERR

main() {
    [[ $# -ge 1 ]] || die "usage: $SCRIPT_NAME TARGET"
    local target=$1
    log "working on $target"
    # ...
}

main "$@"
```

## Practice

1. Rewrite your `backup.sh` from lesson 29 with the template: `getopts` for `-n` (dry run) and `-k DAYS` (retention), `trap` cleanup, `flock`.
2. Use parameter expansion to rename `*.jpeg` to `*.jpg` without `sed`.
3. Run a command on 20 hosts in parallel (simulate with `sleep $((RANDOM%3))`) with at most 5 at a time.
4. Prove the subshell trap: count lines with `cat file | while read`, print the counter after the loop.
5. Make every script in this module pass `shellcheck` with no warnings.

## Check yourself

- When does `set -e` NOT stop a script?
- What does `${DIR:?}` protect you from?
- How do you guarantee a temp directory is removed even on Ctrl+C?

## Next
[Lesson 31. Git for operations](31-git.md)
