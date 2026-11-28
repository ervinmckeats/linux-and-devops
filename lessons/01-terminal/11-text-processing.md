# Lesson 11. Text processing: cut, sort, uniq, sed, awk

## Goal
Turn raw logs and command output into answers with one line of shell.

## The toolbox

```bash
# columns 1 and 7, delimiter ":"
cut -d: -f1,7 /etc/passwd

# characters 1-10
cut -c1-10 file

# alphabetical
sort file

# numeric / human sizes (2K, 1G) / reverse
sort -n / -h / -r

# by 3rd comma-separated column
sort -t, -k3 -n data.csv

# sort and remove duplicates
sort -u

# remove ADJACENT duplicates (sort first!)
uniq

# count them
uniq -c

# translate characters
tr 'a-z' 'A-Z'

# delete Windows line endings
tr -d '\r'

# squeeze repeated spaces
tr -s ' '

# join files side by side
paste a.txt b.txt

# pretty table
column -t -s,

# count lines
wc -l
```

## sed: stream editor

Edits text line by line as it flows by.

```bash
# replace first match per line
sed 's/foo/bar/' file

# all matches
sed 's/foo/bar/g' file

# print lines 10-20 only
sed -n '10,20p' file

# delete comment lines
sed '/^#/d' file

# delete empty lines
sed '/^$/d' file

# edit in place, keep app.conf.bak
sed -i.bak 's/80/8080/' app.conf

# any delimiter, handy for paths
sed 's|/old/path|/new/path|g'

# groups and back-references
sed -E 's/([0-9]+)ms/\1 ms/'
```

Always run without `-i` first and check the output.

## awk: columns and small programs

awk splits each line into fields: `$1`, `$2`, ... `$NF` (last), `$0` (whole line). `NR` = line number.

```bash
# first column
awk '{print $1}' access.log

# custom delimiter
awk -F: '{print $1, $7}' /etc/passwd

# filter: HTTP 5xx
awk '$9 >= 500' access.log

# total bytes
awk '{sum += $10} END {print sum}' access.log

awk '{c[$1]++} END {for (ip in c) print c[ip], ip}' access.log | sort -rn | head

# every second line
awk 'NR % 2 == 0' file

# regular users
awk -F: '$3 >= 1000 {print $1}' /etc/passwd
```

## Real log questions

nginx "combined" format: `IP - - [time] "METHOD PATH PROTO" STATUS BYTES "referer" "agent"`.

```bash
# Top 10 client IPs
awk '{print $1}' access.log | sort | uniq -c | sort -rn | head

# Status code distribution
awk '{print $9}' access.log | sort | uniq -c | sort -rn

# Top paths returning 404
awk '$9 == 404 {print $7}' access.log | sort | uniq -c | sort -rn | head

# Requests per minute (field 4 looks like [10/Jan/2026:12:00:01)
awk '{print substr($4, 2, 17)}' access.log | sort | uniq -c | tail

# Top user agents
awk -F'"' '{print $6}' access.log | sort | uniq -c | sort -rn | head
```

## Practice

Generate a test log:

```bash
for i in $(seq 1000); do
  printf '10.0.0.%d - - [10/Jan/2026:12:%02d:%02d +0000] "GET /p%d HTTP/1.1" %s %d "-" "curl"\n' \
    $((RANDOM%20)) $((RANDOM%60)) $((RANDOM%60)) $((RANDOM%10)) \
    $(shuf -n1 -e 200 200 200 301 404 500) $((RANDOM%5000))
done > access.log
```

1. Top 5 IPs.
2. Share of 5xx responses in percent (one awk command).
3. Total bytes sent for status 200.
4. Replace all `curl` agents with `bot` into a new file using sed.
5. List users with UID 1000 or higher and their shells as an aligned table.

## Check yourself

- Why must you `sort` before `uniq -c`?
- When do you pick `cut` and when `awk`?
- How do you make `sed -i` keep a backup?

---

<p align="left">
  <a href="media/11-text-processing.mp4"><img width="640" src="media/11-text-processing.gif" alt="Lesson 11 animation: Text processing: cut, sort, uniq, sed, awk"></a>
</p>

## Next
[Lesson 12. Users, groups, sudo](../02-sysadmin/12-users-groups-sudo.md)
