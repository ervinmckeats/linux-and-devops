# Lesson 11. Text processing: cut, sort, uniq, sed, awk

## Goal
Turn raw logs and command output into answers with one line of shell.

## The toolbox

```bash
cut -d: -f1,7 /etc/passwd       # columns 1 and 7, delimiter ":"
cut -c1-10 file                 # characters 1-10
sort file                       # alphabetical
sort -n / -h / -r               # numeric / human sizes (2K, 1G) / reverse
sort -t, -k3 -n data.csv        # by 3rd comma-separated column
sort -u                         # sort and remove duplicates
uniq                            # remove ADJACENT duplicates (sort first!)
uniq -c                         # count them
tr 'a-z' 'A-Z'                  # translate characters
tr -d '\r'                      # delete Windows line endings
tr -s ' '                       # squeeze repeated spaces
paste a.txt b.txt               # join files side by side
column -t -s,                   # pretty table
wc -l                           # count lines
```

## sed — stream editor

Edits text line by line as it flows by.

```bash
sed 's/foo/bar/' file            # replace first match per line
sed 's/foo/bar/g' file           # all matches
sed -n '10,20p' file             # print lines 10-20 only
sed '/^#/d' file                 # delete comment lines
sed '/^$/d' file                 # delete empty lines
sed -i.bak 's/80/8080/' app.conf # edit in place, keep app.conf.bak
sed 's|/old/path|/new/path|g'    # any delimiter, handy for paths
sed -E 's/([0-9]+)ms/\1 ms/'     # groups and back-references
```

Always run without `-i` first and check the output.

## awk — columns and small programs

awk splits each line into fields: `$1`, `$2`, ... `$NF` (last), `$0` (whole line). `NR` = line number.

```bash
awk '{print $1}' access.log                     # first column
awk -F: '{print $1, $7}' /etc/passwd            # custom delimiter
awk '$9 >= 500' access.log                      # filter: HTTP 5xx
awk '{sum += $10} END {print sum}' access.log   # total bytes
awk '{c[$1]++} END {for (ip in c) print c[ip], ip}' access.log | sort -rn | head
awk 'NR % 2 == 0' file                          # every second line
awk -F: '$3 >= 1000 {print $1}' /etc/passwd     # regular users
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

## Next
Module 2 — System administration: lesson 12, Users, groups and sudo.
