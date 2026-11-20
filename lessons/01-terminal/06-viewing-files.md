# Lesson 6. Viewing files

## Goal
Read files of any size, follow live logs, and compare files.

## Theory
Opening a 5 GB log in an editor loads it into memory. Pagers and `head`/`tail` read only what you need. On servers, read files with these tools, edit only when you must.

## Commands

```bash
cat file
cat -n file
less file
head file
head -n 50 file
tail -n 100 file
tail -f /var/log/syslog
tail -F app.log
wc -l file
diff old.conf new.conf
diff -u old.conf new.conf
cmp a.bin b.bin
md5sum file; sha256sum file
zcat app.log.1.gz | less
strings /usr/bin/ls | head
hexdump -C file | head
```

- `cat file`: print whole file (small files only)
- `cat -n file`: with line numbers
- `less file`: pager: scroll, search
- `head file`: first 10 lines
- `head -n 50 file`: first 50
- `tail -n 100 file`: last 100
- `tail -f /var/log/syslog`: follow new lines live
- `tail -F app.log`: follow even if the file is rotated
- `wc -l file`: count lines (also `-w` words, `-c` bytes)
- `diff old.conf new.conf`: differences
- `diff -u old.conf new.conf`: unified format, like git
- `cmp a.bin b.bin`: binary compare
- `md5sum file; sha256sum file`: checksums
- `zcat app.log.1.gz | less`: read compressed logs (also zless, zgrep)
- `strings /usr/bin/ls | head`: readable text inside binaries
- `hexdump -C file | head`: raw bytes

Inside `less`:

| Key | Action |
|---|---|
| `Space` / `b` | page down / up |
| `/text`, `?text` | search forward / backward |
| `n` / `N` | next / previous match |
| `g` / `G` | start / end |
| `F` | follow mode, like `tail -f` (`Ctrl+C` to stop) |
| `q` | quit |

`less +F app.log` starts in follow mode. Better than `tail -f` because you can stop, scroll and search.

## Practice

1. Show lines 20–30 of `/etc/services`: `sed -n '20,30p' /etc/services` and also with `head`/`tail`.
2. Open `/var/log/syslog` (or `journalctl` output) in `less`, search for `error`.
3. In terminal 1 run `tail -f /tmp/live.log`. In terminal 2 run `for i in $(seq 10); do date >> /tmp/live.log; sleep 1; done`.
4. Copy `/etc/hosts`, change a line, compare with `diff -u`.
5. Verify a download with `sha256sum`.

## Check yourself

- Why use `less` instead of `cat` for a big log?
- What is the difference between `tail -f` and `tail -F`?
- How do you read a `.gz` log without unpacking it?

## Next
[Lesson 7. Editors](07-editors.md)
