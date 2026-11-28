# Lesson 6. Viewing files

<a href="media/06-viewing-files.mp4"><img align="right" width="440" src="media/06-viewing-files.gif" alt="Lesson 6 animation: viewing files"></a>

## Goal
Read files of any size, follow live logs, and compare files.

## Theory
Opening a 5 GB log in an editor loads it into memory. Pagers and `head`/`tail` read only what you need. On servers, read files with these tools, edit only when you must.

## Commands

```bash
# print whole file (small files only)
cat file

# with line numbers
cat -n file

# pager: scroll, search
less file

# first 10 lines
head file

# first 50
head -n 50 file

# last 100
tail -n 100 file

# follow new lines live
tail -f /var/log/syslog

# follow even if the file is rotated
tail -F app.log

# count lines (also -w words, -c bytes)
wc -l file

# differences
diff old.conf new.conf

# unified format, like git
diff -u old.conf new.conf

# binary compare
cmp a.bin b.bin

# checksums
md5sum file; sha256sum file

# read compressed logs (also zless, zgrep)
zcat app.log.1.gz | less

# readable text inside binaries
strings /usr/bin/ls | head

# raw bytes
hexdump -C file | head
```

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
