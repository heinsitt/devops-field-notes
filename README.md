# DevOps Field Notes

Short notes from daily DevOps work. Every entry is a command or a setting I have
actually used, followed by why it is the right one and what usually goes wrong.

The point is not to repeat the manual. It is to keep the handful of things I look
up twice, and the reason I chose them the first time.

## Index

| Note | What is inside |
| :-- | :-- |
| [Linux](notes/linux.md) | Processes, disks, services and logs on a fresh host |
| [Linux CLI](notes/linux-cli.md) | The command line itself: files, text pipelines, processes, network, transfers and log triage |
| [Docker](notes/docker.md) | Build, run, inspect and clean up without guessing |
| [Docker CLI](notes/docker-cli.md) | Images and builds, running and reading containers, Compose, networks, volumes, registry and disk housekeeping |
| [Kubernetes](notes/kubernetes.md) | The ten commands that cover most of a shift |
| [Kubernetes CLI](notes/kubernetes-cli.md) | Contexts, reading the cluster, logs and debug, apply and rollback, jobs and secrets, nodes and capacity, RBAC, output shaping |
| [Observability](notes/observability.md) | PromQL worth memorising, and alert rules that do not page at 3am for nothing |

## How these are written

- Command first, then one line of reasoning, then the trap.
- Distro or version specific behaviour is marked, not assumed.
- Anything destructive is labelled before it appears, not after.

Notes grow as I hit the same problem a second time. If something here is wrong for
your setup, it is probably a version difference: check `--help` before trusting a
flag.
