# Docker CLI for DevOps

The commands behind a normal day: build it, run it, read it, and clean up after it.
Each entry says what the command is for and the flag that decides whether it works.

## 1. Images and builds

```bash
docker build -t api:1.4.0 .
docker build -f docker/Dockerfile.prod -t api:1.4.0 .
```

Tag with something you can trace back to a commit, not just `latest`. A registry
full of `latest` is a registry where a rollback is a guess.

```bash
docker build --progress=plain --no-cache -t api:debug . 2>&1 | tail -40
docker build --build-arg NODE_ENV=production -t api:1.4.0 .
```

`--progress=plain` prints every layer, which is how you find the one that actually
failed. `--no-cache` is for debugging a broken cache, not for every build: it throws
away the layer cache you spent minutes filling.

```bash
docker images --format 'table {{.Repository}}\t{{.Tag}}\t{{.Size}}'
docker history --no-trunc api:1.4.0 | head -20
docker inspect --format '{{.Config.Env}}' api:1.4.0
```

`docker history` shows layer sizes, so it answers "why is this image 1.2 GB" before
you start deleting things. `inspect --format` pulls one field out instead of dumping
a hundred lines of JSON.

Order in a Dockerfile is the whole performance story: dependencies before source,
so a code change only rebuilds the last layers. `.dockerignore` matters just as much,
because a fat build context is uploaded to the daemon before the first instruction
runs.

## 2. Running containers

```bash
docker run -d --name api -p 8080:80 --env-file .env \
  -v api-data:/data --restart unless-stopped api:1.4.0
```

`-p host:container` publishes a port; without it the container is reachable only from
other containers. `--env-file` keeps secrets out of your shell history. `--restart
unless-stopped` is what makes a container come back after a host reboot, which
`docker run` alone never does.

```bash
docker run --rm -it alpine:3.20 sh
docker run --rm --entrypoint sh api:1.4.0 -c 'ls -la /app'
```

`--rm` deletes the container when it exits: right for one-off checks, wrong for
anything you will want logs from later. `--entrypoint` overrides the image's start
command without building a new image.

```bash
docker ps --format 'table {{.Names}}\t{{.Image}}\t{{.Status}}\t{{.Ports}}'
docker ps -a --filter status=exited --filter status=dead
```

`ps` only shows running containers; `-a` is where the crashed one is hiding.
`--filter` beats piping to `grep`, and it is the version that works in scripts.

## 3. Logs and state

```bash
docker logs -f --tail=100 --timestamps api
docker logs --since 15m --until 5m api
docker logs api 2>&1 | grep -i "error\|panic"
```

`--since` is faster than a large `--tail` when you know the incident window, and
`2>&1` matters because application errors usually go to stderr, which is easy to
lose in a pipe.

```bash
docker inspect --format '{{.State.Status}} {{.State.ExitCode}}' api
docker inspect --format '{{json .State.Health}}' api | head -c 400
docker stats --no-stream --format 'table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}'
docker top api
```

Exit code 137 is SIGKILL, which on a container almost always means the kernel OOM
killer, not a crash in your code. `--no-stream` turns `stats` into a single sample
you can paste into a ticket.

## 4. Getting inside, and getting files out

```bash
docker exec -it api sh
docker exec -u root -it api sh
docker exec api env | sort | grep -i proxy
docker cp api:/var/log/app.log ./app.log
```

Use `sh` first: many slim images have no bash. `-u root` is for diagnosis only, and
anything you change inside a running container disappears at the next deploy. `cp`
works both ways and is the quickest way to read a log the container will not print.

## 5. Compose

```bash
docker compose up -d --build
docker compose ps
docker compose logs -f --tail=50 api
docker compose exec api sh
```

`up -d --build` rebuilds and recreates changed services; `ps` then confirms the
container was recreated rather than reused. A deploy that "worked" but still serves
old code is nearly always a container that was never replaced.

```bash
docker compose config | less          # the file after merge, vars and overrides
docker compose up -d --force-recreate api
docker compose restart api            # restarts the process, does not reload env
```

`config` renders the final YAML, including `.env` substitution and override files,
which settles most "but my variable is set" arguments. `restart` keeps the old
environment: a changed `.env` needs `up -d`, not `restart`.

```bash
docker compose down            # stops and removes containers and networks
docker compose down -v         # also deletes named volumes
```

`down -v` deletes the database volume. It is the right command on a throwaway lab
host and a career-defining mistake on anything with data.

## 6. Networks and ports

```bash
docker network ls
docker network inspect bridge --format '{{range .Containers}}{{.Name}} {{end}}'
docker run --rm --network app_default alpine:3.20 nslookup api
```

On a user-defined network, containers resolve each other by name or service name;
on the default `bridge` they only get IPs. That is why a Compose stack works and a
pile of `docker run` commands does not.

```bash
docker port api                       # which host port maps where
docker inspect --format '{{json .NetworkSettings.Ports}}' api
```

`EXPOSE` in a Dockerfile is documentation, not a publish. Only `-p` or a Compose
`ports:` entry puts the container on the host.

## 7. Volumes and data

```bash
docker volume ls
docker volume inspect api-data --format '{{.Mountpoint}}'
docker run --rm -v api-data:/data alpine:3.20 ls -la /data
```

A named volume survives `down` and image rebuilds; a bind mount is just a host path
and inherits the host's permissions. When a container cannot write to a bind mount,
the usual cause is a UID mismatch between the image user and the host directory
owner, not a Docker problem.

```bash
docker run --rm -v api-data:/data -v "$PWD":/backup alpine:3.20 \
  tar -czf /backup/api-data.tar.gz -C /data .
```

Backing up a volume is a throwaway container with two mounts. It is the safest way
to copy data out without stopping the service for long.

## 8. Registry, auth and housekeeping

```bash
docker login ghcr.io
docker tag api:1.4.0 ghcr.io/heinsitt/api:1.4.0
docker push ghcr.io/heinsitt/api:1.4.0
docker manifest inspect ghcr.io/heinsitt/api:1.4.0 | head -30
```

`manifest inspect` shows the platforms and layers in the registry without pulling,
which is how you confirm an arm64 build actually landed.

```bash
docker system df -v            # what is using the disk, by image and volume
docker image prune             # dangling images only
docker container prune         # stopped containers
docker builder prune --filter until=24h
docker events --filter type=container --since 10m
```

Read `df -v` before deleting anything: build cache, not images, is usually the
biggest number. `events` is the live feed of starts, dies, OOM kills and health
status changes, and it is the fastest way to see what a restart loop is doing.

## Dockerfile habits worth keeping

- Pin the base image (`node:20.11-alpine`, not `node:latest`) so a build is repeatable.
- Install dependencies before copying source, so the cache survives code edits.
- Run as a non-root `USER`: a container with root inside is root on the volume it mounts.
- Add a `HEALTHCHECK` when the app has an endpoint, so `docker ps` can say `unhealthy`
  instead of just `Up`.
- Use multi-stage builds: build tools in the first stage, runtime in the last, and the
  final image gets small enough to pull in seconds.
