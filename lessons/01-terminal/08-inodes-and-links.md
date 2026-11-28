# Lesson 8. Inodes, hard and symbolic links

## Goal
Understand how a file name relates to file data, and why "disk full" can happen with free space left.

## Theory

A file has two parts:

- **Inode**: a record with metadata: size, owner, permissions, timestamps, and where the data blocks are. **The inode does not store the name.**
- **Directory entry**: a pair *name → inode number*. A directory is a table of these pairs.

```
directory /home/alice        inode 1234             data blocks
┌──────────────┬───────┐     ┌────────────────┐     ┌──────────┐
│ notes.txt    │ 1234 ─┼───▶ │ owner, mode,   │ ──▶ │ hello... │
│ backup.txt   │ 1234 ─┼───▶ │ size, links=2  │     └──────────┘
└──────────────┴───────┘     └────────────────┘
```

### Hard link
A second name for the **same inode**. Both names are equal; there is no "original". Data is freed only when the **link count** reaches 0 **and** no process has the file open.
Limits: same filesystem only, not for directories.

### Symbolic (soft) link
A small file that contains a **path** to another file. Works across filesystems and for directories. If the target is removed, the link is **dangling** (broken).

| | Hard link | Symlink |
|---|---|---|
| Points to | inode | path |
| Across filesystems | no | yes |
| Directories | no | yes |
| Target deleted | data stays | link breaks |

### Why it matters in operations

- **Deleted but still open.** `rm` of a big log that nginx still writes to frees **nothing**: the process holds the inode open. `df` says full, `du` says empty. Find it: `lsof +L1`. Fix: restart/reload the process, or truncate the file instead: `: > app.log`.
- **Out of inodes.** Millions of tiny files (sessions, cache) can exhaust inodes while bytes are free: `df -i`.
- **Symlinks for releases.** `current -> releases/2026-01-10` lets you switch versions atomically: `ln -sfn releases/new current`.

## Commands

```bash
# inode number
ls -i file

# inode, links, times
stat file

# hard link
ln file hardlink

# symlink
ln -s /path/target link

# shows link -> target
ls -l link

# resolve full path
readlink -f link

# broken symlinks
find . -xtype l

# all names of an inode
find / -inum 1234 2>/dev/null

# inode usage per filesystem
df -i

# open files with zero links (deleted but held)
lsof +L1
```

## Practice

1. Create `a.txt`, hard link `b.txt`. Change `b.txt`, read `a.txt`. Check link count with `stat`.
2. Delete `a.txt`. Is the data still there?
3. Create a symlink to `b.txt`, delete `b.txt`, look at the link with `ls -l`.
4. Reproduce the "deleted but open" problem:
   ```bash
   dd if=/dev/zero of=/tmp/big bs=1M count=200

   # keep it open
   tail -f /tmp/big > /dev/null &

   rm /tmp/big; df -h /tmp; lsof +L1 | grep big
   kill %1; df -h /tmp
   ```

## Check yourself

- Where is a file's name stored?
- Why can you not hard link a directory?
- `df` shows 100% but `du` finds only 40%. First two things you check?

---

<p align="left">
  <a href="media/08-inodes-and-links.mp4"><img width="640" src="media/08-inodes-and-links.gif" alt="Lesson 8 animation: Inodes, hard and symbolic links"></a>
</p>

## Next
[Lesson 9. Searching: find and grep](09-find-and-grep.md)
