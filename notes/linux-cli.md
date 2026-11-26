# Linux CLI for DevOps

The commands I actually type, grouped the way I reach for them. Each one gets a
line on why it is the right tool and a line on the flag that trips people up.

## 1. Files, sizes and where the space went

```bash
ls -lah                     # -a shows dotfiles, -h makes bytes readable
stat app.log                # size, permissions, and the three timestamps
du -sh */ | sort -rh | head # biggest directories in the current one
df -h                       # free space per filesystem
```

`du` and `df` disagreeing is normal and useful: `df` counts the filesystem, `du`
counts files it can see. A gap means deleted-but-open files, a mount hiding a
directory, or a reserved block percentage.

```bash
find /var/log -name "*.log" -mtime -1 -size +50M
```

`-mtime -1` is "modified in the last 24 hours" (a negative number means "less than"),
which is the opposite of what most people guess. Combine `-size +50M` to find the
files worth opening first.

## 2. Text: grep, sed, awk, and the pipe

```bash
grep -rn --include="*.conf" "listen" /etc/nginx
grep -c " 500 " access.log          # count matches, not print them
grep -v "^#" nginx.conf             # drop comment lines
```

`-r` recurses, `-n` prints line numbers so the next command can jump there, and
`--include` keeps you out of binary files. `-v` inverts, which is how you read a
config file without the comments.

```bash
sed -n '40,60p' app.log             # print a line range, like head/tail in the middle
sed -i.bak 's/old-host/new-host/g' config.yml
```

`-i.bak` edits in place and keeps a copy. `-i` alone edits in place with no undo,
which is how people lose a config on a production box.

```bash
awk '{print $1}' access.log | sort | uniq -c | sort -rn | head
```

Top ten client IPs in an nginx access log. Read it as a sentence: take field one,
sort it so duplicates sit together, count them, sort by count, show ten. Nearly
every log question is a variation of this pipe.

```bash
awk -F: '$3 >= 1000 {print $1}' /etc/passwd   # human accounts only
awk '{sum += $5} END {print sum}' sizes.txt   # a column total without a spreadsheet
```

`-F` sets the field separator. `END` runs once after the last line, which is where
totals and counts belong.

```bash
sort -k2 -n -r report.txt   # sort by column 2, numerically, descending
uniq -c                     # counts runs, so sort first
cut -d: -f1 /etc/passwd     # field one, colon separated
wc -l file                  # lines, the cheapest sanity check there is
```

`uniq` only collapses adjacent duplicates, which is why it is always paired with
`sort`. Forgetting that produces a "unique" list with repeats in it.

## 3. Processes

```bash
ps aux --sort=-%mem | head -10   # the ten hungriest processes
pgrep -af nginx                  # pid plus full command line
top -b -n1 | head -20            # one snapshot, script friendly
```

`ps aux` alone is noise. `--sort` answers "who is using the box" in one screen.
`pgrep -af` is the safe version of `ps aux | grep`: it does not match its own grep.

```bash
kill -TERM 4242     # ask nicely, lets the app close connections
kill -KILL 4242     # no cleanup, last resort
pkill -f "worker.py --queue slow"
```

Always try `TERM` first. `KILL` skips shutdown hooks, so a database can come back
with a recovery on the next start.

```bash
nice -n 10 backup.sh      # start something at lower priority
renice -n 5 -p 4242       # change a running process
ulimit -n                 # open file limit for this shell
```

`ulimit -n` is the first thing to check when a service reports "too many open
files". Raising it in a shell does not raise it for systemd units: that needs
`LimitNOFILE` in the unit file.

## 4. Services, timers and logs

```bash
systemctl daemon-reload         # after editing a unit file, always
systemctl enable --now myapp    # enable at boot and start right now
systemctl list-timers --all     # what will run, and when
journalctl -u myapp --since "30 min ago" -p warning
```

`daemon-reload` is the missing step behind most "I changed the unit and nothing
happened" moments. `-p warning` filters by priority, which is how you read a noisy
unit without paging through a week of info lines.

```bash
crontab -e
crontab -l
```

```text
*/5 * * * *  /usr/local/bin/healthcheck.sh >> /var/log/healthcheck.log 2>&1
```

Five fields: minute, hour, day of month, month, day of week. Two habits that save
incidents: redirect both stdout and stderr (`>> file 2>&1`), and use absolute
paths, because cron runs with a minimal `PATH` and a different working directory.
