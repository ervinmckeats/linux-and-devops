# Lesson 21. Archives and backups

## Goal
Pack, compress and copy data, and build backups that you have actually restored at least once.

## Archives and compression

`tar` packs many files into one; a compressor shrinks it. They are separate steps that `tar` can do in one go.

```bash
tar -czf site.tar.gz /var/www/site
tar -cJf site.tar.xz /var/www/site
tar --zstd -cf site.tar.zst dir/
tar -tzf site.tar.gz | head
tar -xzf site.tar.gz -C /tmp/restore
tar -xzf site.tar.gz var/www/site/index.html
tar -czf - dir | ssh host 'tar -xzf - -C /backup'
zip -r site.zip dir/; unzip site.zip -d out/
gzip big.log; gunzip big.log.gz; zcat big.log.gz
```

- `tar -czf site.tar.gz /var/www/site`: create, gzip
- `tar -cJf site.tar.xz /var/www/site`: xz: smaller, slower
- `tar --zstd -cf site.tar.zst dir/`: zstd: fast and small, good default today
- `tar -tzf site.tar.gz | head`: list contents
- `tar -xzf site.tar.gz -C /tmp/restore`: extract into a directory
- `tar -xzf site.tar.gz var/www/site/index.html`: one file
- `tar -czf - dir | ssh host 'tar -xzf - -C /backup'`: stream over SSH, no temp file

Memory trick: **c**reate, e**x**tract, lis**t**; **z** gzip, **J** xz; **f** file name follows.

| Tool | Speed | Ratio |
|---|---|---|
| gzip | fast | ok |
| zstd | very fast | good |
| xz | slow | best |

## rsync

Copies only differences, resumes, keeps permissions.

```bash
rsync -avh src/ dst/
rsync -avh src dst/
rsync -avh --delete src/ dst/
rsync -avhn --delete src/ dst/
rsync -avzP src/ user@host:/backup/
rsync -aHAX --numeric-ids / /mnt/new/
```

- `rsync -avh src/ dst/`: trailing slash: copy CONTENTS of src
- `rsync -avh src dst/`: no slash: creates dst/src
- `rsync -avh --delete src/ dst/`: mirror: delete extras in dst
- `rsync -avhn --delete src/ dst/`: `-n` is a dry run, ALWAYS before `--delete`
- `rsync -avzP src/ user@host:/backup/`: over SSH, compress, progress
- `rsync -aHAX --numeric-ids / /mnt/new/`: full system copy: hard links, ACL, xattrs

## Backup strategy

### 3-2-1 rule
- **3** copies of the data
- on **2** different media/systems
- **1** off-site (other region, other provider, or offline)

Add: at least one copy **immutable or offline**, so ransomware or a stolen admin key cannot delete it.

### Questions that define a backup plan
- **RPO** (recovery point objective): how much data can we lose? 24 h → nightly backups. 5 min → WAL/binlog shipping.
- **RTO** (recovery time objective): how long can we be down? Drives the restore method.
- What exactly is backed up? Data, configs, secrets, and the list of installed packages.

### Databases need dumps, not file copies

Copying the files of a running database gives a corrupt backup.

```bash
pg_dump -Fc mydb > mydb-$(date +%F).dump
mysqldump --single-transaction mydb | zstd > mydb.sql.zst
```

- `pg_dump -Fc mydb > mydb-$(date +%F).dump`: PostgreSQL
- `mysqldump --single-transaction mydb | zstd > mydb.sql.zst`: MySQL/MariaDB

Or stop writes / use an LVM or cloud snapshot of a consistent state.

### Tools

- **restic**, **borg**: deduplicated, encrypted, incremental backups to local disk, SSH or S3. Use them instead of home-made tar scripts for real data.

```bash
restic -r /backup/repo init
restic -r /backup/repo backup /etc /var/www
restic -r /backup/repo snapshots
restic -r /backup/repo forget --keep-daily 7 --keep-weekly 4 --keep-monthly 6 --prune
restic -r /backup/repo restore latest --target /tmp/restore
```

### Simple script with rotation

```bash
#!/usr/bin/env bash
set -euo pipefail
DEST=/backup
STAMP=$(date +%F_%H%M)
tar --zstd -cf "$DEST/etc-$STAMP.tar.zst" /etc
find "$DEST" -name 'etc-*.tar.zst' -mtime +14 -delete
tar --zstd -tf "$DEST/etc-$STAMP.tar.zst" > /dev/null
```

- `tar --zstd -tf "$DEST/etc-$STAMP.tar.zst" > /dev/null`: verify it can be read

## A backup you have not restored is not a backup

Schedule restore tests: restore into a temporary VM or directory, start the app or DB on it, compare checksums or row counts. Write down how long it took: that is your real RTO.

## Practice

1. Archive `/etc` with gzip, xz and zstd. Compare size and time (`time`).
2. Mirror a directory with `rsync --delete`; first with `-n`. Then delete a file in the source and run again.
3. Set up restic to a local repo, back up `/etc` twice, list snapshots, restore one file.
4. Run your backup script from a systemd timer (lesson 18).
5. Do a full restore test and measure the time.

## Check yourself

- What is the difference between `rsync src/ dst` and `rsync src dst`?
- What are RPO and RTO?
- Why is copying `/var/lib/postgresql` of a running server not a backup?

## Next
[Lesson 22. Boot process and recovery](22-boot-and-recovery.md)
