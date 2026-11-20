# Lesson 10. Redirection, pipes, exit codes, xargs

## Goal
Connect commands into pipelines, control where output goes, and know whether a command succeeded.

## Theory

Every process starts with three open **streams** (file descriptors):

| FD | Name | Default |
|---|---|---|
| 0 | stdin | keyboard |
| 1 | stdout | screen |
| 2 | stderr | screen |

Errors go to stderr on purpose: you can save results and still see errors, or the other way round.

A **pipe** `|` connects stdout of one command to stdin of the next. Both run **at the same time**; data flows as it is produced.

Every command ends with an **exit code**: `0` = success, anything else = failure. Shell logic (`&&`, `||`, `if`) is built on it.

## Redirection

```bash
cmd > out.txt          # stdout to file (overwrite)
cmd >> out.txt         # append
cmd 2> err.txt         # stderr to file
cmd > all.txt 2>&1     # both to one file (order matters!)
cmd &> all.txt         # same, bash short form
cmd 2>/dev/null        # throw errors away
cmd < input.txt        # stdin from file
cmd <<< "text"         # here-string
```

Here-doc: multi-line stdin. The word after `<<` is the end marker; quote it to stop variable expansion inside.

```bash
cat <<'END' > note.txt
Hello, $USER is printed literally
END
```

`2>&1` means "make FD 2 point where FD 1 points **now**". So `cmd 2>&1 > file` sends errors to the screen, not the file.

`sudo echo x > /etc/file` fails: the redirect is done by **your** shell, not by sudo. Use `echo x | sudo tee /etc/file` (`tee -a` to append).

## Pipes

```bash
ps aux | grep nginx
awk '{print $1}' access.log | sort | uniq -c | sort -rn | head
ls -l | tee listing.txt | wc -l      # tee: save AND pass along
```

## Exit codes and chaining

```bash
ls /nope; echo $?        # 2
true; echo $?            # 0
cmd1 && cmd2             # run cmd2 only if cmd1 succeeded
cmd1 || cmd2             # run cmd2 only if cmd1 failed
mkdir -p /opt/app && cd /opt/app || exit 1
set -o pipefail          # pipeline fails if ANY part fails (not only the last)
echo "${PIPESTATUS[@]}"  # exit code of every part of the last pipeline
```

Common codes: `1` general error, `2` misuse, `126` not executable, `127` command not found, `128+N` killed by signal N (`137` = 128+9 = SIGKILL, often the OOM killer; `130` = Ctrl+C; `143` = SIGTERM).

## xargs

Many commands do not read stdin, they need arguments. `xargs` turns lines into arguments.

```bash
find . -name "*.log" -print0 | xargs -0 rm         # safe with spaces
xargs -I{} ping -c1 {} < hosts.txt                 # one run per line
xargs -P 8 -n 1 curl -sO < urls.txt                # 8 in parallel
```

## Substitution

```bash
echo "Today is $(date +%F)"       # command substitution
diff <(sort a.txt) <(sort b.txt)  # process substitution: output as a file
```

## Practice

1. Run `ls /etc /nope > ok.txt 2> err.txt`. Look at both files.
2. Prove that `2>&1 > file` and `> file 2>&1` differ.
3. Write `hello` into `/etc/motd` using `tee` and sudo.
4. Run `false | true; echo $?`, then again after `set -o pipefail`. Explain.
5. Find the 5 most common shells in `/etc/passwd`: `cut -d: -f7 /etc/passwd | sort | uniq -c | sort -rn | head -5`.
6. Compare installed packages on two servers using process substitution and `ssh host dpkg -l`.

## Check yourself

- Why does `sudo echo x > /etc/file` fail?
- A container exited with code 137. What happened?
- Why do we use `-print0 | xargs -0`?

## Next
[Lesson 11. Text processing](11-text-processing.md)
