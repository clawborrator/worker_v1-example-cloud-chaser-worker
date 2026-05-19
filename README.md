# worker_v1-example-cloud-chaser-worker

Docker compose stack for the **cloud-chaser** agent — a clawborrator
worker that runs on every server in your fleet, evaluates local
system + docker health every hour, and maintains a per-server report
in a shared GitHub repo. A static dashboard (`public/index.html`)
gives an at-a-glance health view of every server, with click-through
to per-server history.

Multi-instance, single-repo. Every server runs the SAME image with the
SAME `REPO_URL`. Each instance scopes its writes to its own
`data/<server-name>/` subfolder, so concurrent commits across servers
don't fight.

## Companion repo

The agent's playbook (`CLAUDE.md`), collection script
(`specialists/collect.sh`), and dashboard renderer
(`specialists/render.js`) live in the sibling repo:

```
https://github.com/clawborrator/worker_v1-example-cloud-chaser-repo
```

`docker compose up` clones that repo into `/workspace/repo` and
hands control to the playbook in its `CLAUDE.md`.

## Setup (one time, on each server)

### 1. Confirm `~/.clawborrator-spawn.env` exists on the host

Every clawborrator worker on this host shares one env file at
`~/.clawborrator-spawn.env`. All secrets live there — the
per-worker `.env` has zero secrets and zero per-host edits. The
file must contain:

```
CLAUDE_CODE_OAUTH_TOKEN=<from `claude setup-token`>
CLAWBORRATOR_TOKEN=<from `npx clawborrator-cli token mint --name cloud-chaser-$(hostname)`>
CLAWBORRATOR_HUB_URL=wss://next.clawborrator.com
REPO_PAT=<GitHub PAT, repo scope, for git push>
GIT_USER_EMAIL=<your email>
GIT_USER_NAME=<your name>
```

If you've already brought up any other clawborrator worker on this
host, this file is in place — verify the new fields are present
and add the missing ones. If not, create it before continuing.

GitHub PAT generation: Settings → Developer settings → Personal
access tokens → Tokens (classic) → Generate new token, repo scope.
Same PAT can be reused across servers.

### 2. Configure .env (no edits required)

```bash
cp .env.example .env
```

That's it. The defaults in `.env.example` are valid out of the
box. No editing required unless:

- you want a different notification target (`NOTIFY_PEER=`)
- the kernel hostname isn't a good slug (e.g. AWS gives you
  `ip-10-0-1-23`) and you'd rather see `prod1` in the dashboard:
  set `SERVER_NAME_OVERRIDE=prod1`

### 3. Start

```bash
export HOSTNAME=$(hostname)
docker compose up -d
docker compose logs -f cloud-chaser
```

The `export HOSTNAME=$(hostname)` is belt-and-suspenders — most
interactive shells set `$HOSTNAME` already, but non-interactive
contexts (cron, systemd unit, fresh `sudo`) sometimes don't. If
docker-compose can't read `$HOSTNAME`, the container ends up named
`cloud-chaser-unnamed` even though the agent still derives the
correct SERVER_NAME from `/host/etc/hostname` for its data folder.
Setting it explicitly avoids that cosmetic mismatch.

First warmup cycle runs in ~30-60s. After that, `CronCreate
0 * * * *` paces hourly cycles (top of every hour).

### 4. Open the dashboard

After the first push, the dashboard lives at:

```
https://github.com/clawborrator/worker_v1-example-cloud-chaser-repo/blob/main/public/index.html
```

For a render-friendly URL, enable GitHub Pages on the companion repo
(Settings → Pages → main branch → /public). Then your dashboard is
at `https://clawborrator.github.io/worker_v1-example-cloud-chaser-repo/`.

## Fleet rollout via clawborrator peer routing

The setup above assumes you SSH into each server and run the docker
commands yourself. That doesn't scale past a handful of hosts. For
fleet rollout, the recommended pattern is **master-to-master peer
routing**: your local laptop session (e.g. `@clauderemote`) routes a
single deployment prompt to a long-running Claude Code session on
each target server (e.g. `@manager-023`, `@manager-024`, …) which
then executes the docker lifecycle there. No SSH from your laptop;
no per-host hands-on time after the first one.

### Prerequisites on each target server

1. A long-running master Claude Code session is already up,
   registered as a peer on the clawborrator hub with a stable
   routing name. Confirm via `claw peers ls` from your laptop.
2. `~/.clawborrator-spawn.env` exists on the server with all six
   secrets (see step 1 of `Setup`). Every clawborrator worker on
   that host reads from this file.

If the server doesn't yet have a master session, bring one up first
(see the top-level clawborrator docs for `claw spawn`). The
cloud-chaser is a *worker* — the master session is the operator's
remote presence on that host, and it's the thing the laptop session
talks to.

### The deployment prompt

From your local master session, route this to each target server's
master session via `mcp__clawborrator__route_to_peer` (mode `ask`
so you get back a status report), or `claw chat @manager-023 "..."`
from a terminal:

```
Stand up the cloud-chaser worker on this host.

1. cd ~
2. git clone https://github.com/clawborrator/worker_v1-example-cloud-chaser-worker
3. cd worker_v1-example-cloud-chaser-worker
4. cp .env.example .env
5. Confirm ~/.clawborrator-spawn.env has CLAUDE_CODE_OAUTH_TOKEN,
   CLAWBORRATOR_TOKEN, CLAWBORRATOR_HUB_URL, REPO_PAT,
   GIT_USER_EMAIL, GIT_USER_NAME. (If missing, stop and ask.)
6. export HOSTNAME=$(hostname) && docker compose up -d
7. docker compose logs --tail 200 cloud-chaser
   (use --tail not -f, -f would block your session)

Reply with: container name docker assigned, any errors, and
whether the warmup cycle commits a green/amber/red snapshot to
data/<hostname>/ within ~5 minutes.
```

Zero secrets cross the wire in this prompt. Everything sensitive
already lives in the spawn-env on the target host.

### Verification

The target server's master session will (a) report immediate
success / failure of the `up` and (b) poll the companion repo for
the warmup commit. The new server shows up automatically on
`public/index.html` once the renderer regenerates the dashboard.

### Pushing fixes fleet-wide afterward

Once cloud-chaser is running on a server, future fixes deploy via
the companion repo, not via the worker stack:

| Change                                | Fleet-wide rollout                              |
|---------------------------------------|--------------------------------------------------|
| `specialists/*.sh` or `render.js`     | Push to companion repo. Self-deploys at the next cycle (the cycle's Step 1 runs `git pull --ff-only`). |
| `CLAUDE.md` (the playbook)            | Push to companion repo, then route `docker compose restart cloud-chaser` to each server. The entrypoint git-pulls on boot and the agent re-reads the playbook. |
| `.env.example` / `docker-compose.yml` | Route `cd ~/worker_v1-example-cloud-chaser-worker && git pull && docker compose up -d --force-recreate` to each server. |

This is the "edit-push-restart" loop scaled fleet-wide via peer
routing — no SSH, no per-host shell time.

## What the agent collects

Per cycle, the agent shells out to `specialists/collect.sh` which
captures:

**Host system** (via `/host/proc`, `/host/sys`, `/host/var/log` mounts):

- `loadavg` — 1/5/15-minute load averages.
- `cpu_pct` — current CPU utilisation.
- `mem_total_mb`, `mem_used_mb`, `mem_pct`.
- Per-mount disk usage (size, used, percent).
- Kernel errors in the last hour (dmesg / journalctl).
- Failed systemd units.
- Uptime.

**Docker** (via the read-only docker socket):

- Container inventory: name, image, status, uptime, CPU%, mem.
- Recent error lines per container (`docker logs --since 1h`,
  grep'd for `ERROR|FATAL|panic|fail`).
- Healthcheck state where containers define one
  (`docker inspect --format '{{json .State.Health}}'`).

**Overall health** (derived):

| Tier  | Rule                                                                   |
|-------|------------------------------------------------------------------------|
| red   | any disk ≥ 95%, any kernel error in the last hour, any container marked `unhealthy`, any container that exited unexpectedly |
| amber | any disk ≥ 80%, mem ≥ 90%, load > cores, any container restarted in the last hour, any error-line spike |
| green | none of the above                                                      |

## Mounts

The host mounts the agent needs to read are read-only. The agent
never writes to the host filesystem:

| Host path                | Container path           | Mode | Why                          |
|--------------------------|--------------------------|------|------------------------------|
| `/var/run/docker.sock`   | `/var/run/docker.sock`   | ro   | `docker ps`, `docker inspect`, `docker logs` |
| `/proc`                  | `/host/proc`             | ro   | loadavg, meminfo, diskstats  |
| `/sys`                   | `/host/sys`              | ro   | cpu count, block device info |
| `/var/log`               | `/host/var/log`          | ro   | syslog / journal             |
| `/etc/hostname`          | `/host/etc/hostname`     | ro   | source of SERVER_NAME at runtime |

The agent reads from `/host/...` paths in its specialists. The docker
socket is genuinely sensitive — anyone who can write to the socket
controls the host. Mounting it `:ro` keeps the kernel from accepting
writes through this bind, but defense in depth still applies: don't
run untrusted playbooks against this image.

## Stopping / restarting

```bash
docker compose down                       # stop + remove
docker compose up -d                      # restart (re-clones repo, fresh cron)
docker compose logs -f cloud-chaser
```

Restarts pull the latest playbook from `REPO_URL` (the
`worker_v1` entrypoint runs `git pull --ff-only` on boot when the
workspace already has a checkout). So pushing a CLAUDE.md change to
the companion repo + restarting all the cloud-chaser containers is
how you ship new behavior fleet-wide.

## What this is NOT

- **Not a substitute for proper observability.** This is for
  small fleets where deploying Datadog / Prometheus / Grafana is
  overkill or out of budget. The reasoning Claude does on top of the
  raw metrics is the value; the metrics themselves are basic.
- **Not a remediation system.** The agent reports. It does not
  restart containers or page humans. Add a `route_to_peer` notification
  to `@oncall` in CLAUDE.md if you want that, or wire a separate
  worker that consumes the per-server JSON.

## Companion repo conventions

- One subfolder per server under `data/`.
- 7-day rolling history: each cycle prunes its own subfolder of
  snapshots older than 168 hours.
- `public/index.html` regenerated every cycle by every worker
  (idempotent; deterministic from the data files).
- Commits scoped to `data/<server-name>/` and `public/` so two
  servers committing simultaneously don't touch the same paths.
- `git pull --rebase` before each commit; one retry on push
  rejection.
