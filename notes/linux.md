# Linux

## Find out what is actually eating the disk

```bash
du -xh --max-depth=1 / 2>/dev/null | sort -rh | head -15
```

`-x` stays on one filesystem, so a mounted share does not hide the real answer.
`-h` plus `sort -rh` is the only part people forget: without it you read a wall of
blocks.

Trap: `df` says the disk is full but `du` finds nothing. That is usually a deleted
file still held open by a process:

```bash
lsof +L1
```

Restart or reload the process holding it; the space comes back.

## Services, without systemctl guesswork

```bash
systemctl status nginx --no-pager
systemctl list-units --failed
journalctl -u nginx -n 100 --no-pager
```

`--no-pager` matters in scripts and in SSH sessions that drop you into `less`.
`list-units --failed` is the fastest triage on an unfamiliar box: it answers "what
is broken" before "what is installed".

## Permissions in one line

```bash
chmod 640 app.env      # owner read/write, group read, nobody else
chown -R www-data:www-data /var/www/app
```

A blanket world-writable `chmod -R` is not a fix, it is a future incident. If a
service cannot read a file, check the directory it lives in as well: traversal
needs `x`, not `r`.

## Ports

```bash
ss -tulpn | grep -E ':(80|443|8080)\b'
```

`ss` replaces `netstat` on every current distro. If the port is listening on
`127.0.0.1` and the client is remote, the firewall is innocent: the bind address
is the problem.

## Memory pressure, not just "free memory"

```bash
free -h
vmstat 1 5
```

Linux uses spare memory for cache, so a low `free` number is normal. Watch
`available` and the `si`/`so` columns in `vmstat`: swap activity is the signal,
not the cache.
