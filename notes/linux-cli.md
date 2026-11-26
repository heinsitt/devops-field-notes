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

## 5. Network, ports and DNS

```bash
ss -tulpn                  # listening sockets, with the process that owns them
lsof -i :8080              # who holds this port
ip -br a                   # interfaces, one line each
ip r                       # routing table, the default gateway line matters
```

`ss` is the current tool; `netstat` is deprecated on every modern distro. If a
service "is not reachable", check the bind address first: `127.0.0.1:8080` is
invisible from outside the host no matter what the firewall says.

```bash
curl -I https://api.internal/health          # headers only, fast check
curl -sv -o /dev/null https://api.internal   # verbose handshake and TLS detail
curl --resolve api.internal:443:10.0.0.12 https://api.internal/
```

`--resolve` tests a hostname against a specific IP before DNS is updated, which is
how you validate a new node without touching the zone file.

```bash
dig +short api.internal
dig @1.1.1.1 api.internal +trace
nc -zv db.internal 5432      # is the port actually open
mtr -rwc 20 api.internal     # where the packets slow down
```

`nc -zv` separates "the app is broken" from "the path is broken" in one command.
`mtr` beats `traceroute` because it keeps running and shows loss per hop.

## 6. Moving files around

```bash
rsync -avz --delete --dry-run src/ host:/srv/app/
rsync -avz --progress src/ host:/srv/app/
```

The trailing slash is the whole story: `src/` copies the contents, `src` copies
the directory itself. `--dry-run` first, always, because `--delete` removes files
at the destination that are missing at the source.

```bash
tar -czf backup-$(date +%F).tar.gz /srv/app
tar -tzf backup-2026-01-01.tar.gz | head   # list before extracting
tar -xzf backup.tar.gz -C /restore
```

`-t` lists, `-x` extracts, `-c` creates. Listing an archive before extracting it
is cheap insurance against unpacking into the wrong directory.

```bash
scp app.env host:/srv/app/.env
ssh -L 5432:db.internal:5432 bastion    # tunnel a database port through a jump host
```

`ssh -L` is how you reach a private database without opening it to the internet.
The tunnel lives as long as the session, so it is a debugging tool, not a
deployment mechanism.

## 7. Shell habits that save time

```bash
sudo !!            # re-run the last command with sudo
Ctrl+R             # search history backwards, type a fragment
history | grep ssh # find the long command you ran last week
```

```bash
export PATH="$HOME/bin:$PATH"   # prepend, never replace
env | sort | grep -i proxy      # what the process will actually see
```

`export` only affects the current shell and its children. A variable that works
interactively but not in a systemd unit is almost always an `Environment=` line
that was never added to the unit.

## 8. Log triage one-liners

```bash
awk '{print $9}' access.log | sort | uniq -c | sort -rn   # status code spread
awk '$9 >= 500' access.log | wc -l                        # how many failures
grep -c "Out of memory" /var/log/syslog                   # did the kernel act
journalctl --since "10 min ago" -p err --no-pager          # what just broke
tail -f app.log | grep --line-buffered -i "error\|timeout"
```

`--line-buffered` is why a `tail -f | grep` sometimes shows nothing for minutes:
without it, grep buffers and you think the app went quiet. The kernel line matters
because an OOM kill looks like a random crash from inside the container.

## A note on reading all of this

Flags differ between GNU and BSD tools, and between Ubuntu and Alpine. When a
command from a note fails, check `--help` on that host before assuming the note is
wrong: it is usually the build, not the idea.
