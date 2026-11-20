# Lesson 5. Files and directories, globbing

## Goal
Create, copy, move and delete files safely, and select many files with patterns.

## Theory

- There is **no trash bin** in the terminal. `rm` deletes for good.
- File extensions mean nothing to Linux. The content decides the type: `file photo.jpg` checks it.
- **Globbing**: the *shell* expands patterns into file names *before* the command runs. `rm *.log` becomes `rm a.log b.log`.

| Pattern | Matches |
|---|---|
| `*` | any characters (not hidden files) |
| `?` | exactly one character |
| `[abc]`, `[0-9]` | one character from the set |
| `{jpg,png}` | brace expansion: generates both words, even if no files exist |

```bash
echo *.conf          # see what a glob expands to before using it
echo file{1..5}.txt  # file1.txt ... file5.txt
```

**Always test a glob with `echo` or `ls` before `rm`.**

## Commands

```bash
touch notes.txt                 # create empty file / update its time
mkdir projects                  # create directory
mkdir -p a/b/c                  # create the whole path
cp file.txt backup.txt          # copy
cp -r dir/ dir-copy/            # copy a directory
cp -a src/ dst/                 # archive: keep owner, time, permissions, links
mv old.txt new.txt              # rename
mv file.txt /tmp/               # move
rm file.txt                     # delete
rm -i *.txt                     # ask before each
rm -r dir/                      # delete directory recursively
rmdir empty-dir                 # delete only if empty
file /bin/ls                    # detect real file type
stat notes.txt                  # all metadata
```

### Safety habits

- Use `cp -i` / `mv -i` to avoid overwriting by accident (`-n` never overwrites).
- `rm -rf "$DIR/"` with an empty `$DIR` becomes `rm -rf /`. In scripts use `"${DIR:?}"` (lesson 30).
- Names with spaces need quotes: `rm "my file.txt"` or `rm my\ file.txt`. Tab completion escapes for you.
- A file named `-rf`? Use `rm -- -rf` or `rm ./-rf`.

## Practice

1. Create `lab/{src,docs,logs}` in one command.
2. Create `report-01.txt` ... `report-20.txt` in `lab/docs` in one command.
3. Using a glob, list only reports 10–19.
4. Copy reports 01–05 to `lab/src` preserving timestamps.
5. Create a file named `-n` and delete it.
6. Rename all `.txt` in `lab/src` to `.md` with a loop:
   ```bash
   for f in lab/src/*.txt; do mv -- "$f" "${f%.txt}.md"; done
   ```

## Check yourself

- Who expands `*` — the shell or `rm`?
- What is the difference between `cp -r` and `cp -a`?
- Why is `{a,b}` different from `[ab]`?

## Next
[Lesson 6. Viewing files](06-viewing-files.md)
