# Docker

## See what is running, and what it is doing to the host

```bash
docker ps --format 'table {{.Names}}\t{{.Image}}\t{{.Status}}\t{{.Ports}}'
docker stats --no-stream
```

The default `docker ps` output is too wide for an 80 column terminal, which is why
people pipe it through `cut` and lose the ports column. `--format` keeps it
readable on a jump box.

## Logs, without a stuck terminal

```bash
docker logs -f --tail=100 api
docker logs --since 15m api
docker inspect --format '{{json .State.Health}}' api
```

`--since` is faster than `--tail` when you know the incident window. Health state
is the first thing to check when a container is "up" but not answering: it tells
you whether the image's own healthcheck is passing.

## Get a shell, and know when not to

```bash
docker exec -it api sh
docker exec -it api env | sort | grep -i proxy
```

If `bash` is missing, the image is probably Alpine: use `sh`. Running `exec` in
production is a diagnostic step, not a workflow; anything you change inside the
container disappears on the next deploy.

## Build cache, the honest way

```bash
docker build --progress=plain -t api:local .
docker history --no-trunc api:local | head -20
```

`--progress=plain` shows the layer that actually failed instead of a spinner.
Order the Dockerfile so that the dependency install happens before the source
copy, otherwise every code change rebuilds the world.

## Clean up without deleting what you still need

```bash
docker system df
docker image prune          # dangling images only
docker builder prune        # build cache
```

Read `docker system df` first. The usual surprise is that build cache, not images,
is what filled the disk. Avoid `docker system prune -a --volumes` on a host with
named volumes you care about: it does not ask twice.

## Compose, the two commands that matter

```bash
docker compose up -d --build
docker compose ps && docker compose logs -f --tail=50
```

`up -d --build` after a code change, and `ps` to confirm the container was
recreated rather than reused. A "successful" deploy that still serves the old code
is almost always a container that was never recreated.
