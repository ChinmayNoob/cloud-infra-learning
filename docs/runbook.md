# Runbook

How to check, shut down and start up everything the course runs on the server. Updated as new pieces are added.

**Covers:** everything up to [implementation 2](implementation/implementation-2.md).

Run all commands **on the server, from `~/cloud-infra-learning`**.

## What runs

| Container | What it is | Starts again after a reboot? |
|-----------|------------|------------------------------|
| `cloud-infra-learning-postgres-1` | The database (Postgres 17, port 5432) | No (no restart policy) |
| `goals` | The app copy run with `docker run` (port 8090) | **Yes** (`unless-stopped`) |
| `floci` | The Floci emulator (port 4566) | **Yes** (`unless-stopped`) |
| `floci-ecr-registry` | Floci's ECR storage (port 5100) | No, but Floci deliberately leaves it running when Floci stops |
| `floci-ecs-<task-id>-goals` | The app copy run by ECS | No, but Floci recreates it while the ECS service wants 1 copy |

External services (Google OAuth, GitHub) cost nothing when idle, so there's nothing to turn off there.

## Check status

```bash
docker ps -a --format "table {{.Names}}\t{{.Status}}" \
  | grep -E "NAMES|^(goals|floci|cloud-infra-learning)"

source floci/env.sh
aws ecs describe-services --cluster fem-fd-service --services goals \
  --query "services[0].[desiredCount,runningCount]" --output text   # "1 1" = the ECS copy is up
```

## Shut down

Follow this order:

```bash
# 1. Tell ECS to run 0 copies. Otherwise Floci starts the task again.
source floci/env.sh
aws ecs update-service --cluster fem-fd-service --service goals --desired-count 0 \
  --query "service.desiredCount" --output text

# 2. Stop the docker-run copy of the app
docker stop goals

# 3. Stop Floci, then its registry, which Floci leaves running on purpose
docker compose -f floci/docker-compose.yml stop
docker stop floci-ecr-registry

# 4. Stop Postgres
docker compose stop
```

**Why this order:** ECS goes to 0 **before** Floci stops. If Floci stops first, the ECS task container is left running on its own, with nothing managing it.

### What `stop` keeps

`stop` only pauses containers. Everything is kept:

| Kept | Where |
|------|-------|
| Database (profile, admin, posts) | Docker volume `cloud-infra-learning_postgres` |
| Floci state (SSM secrets, IAM role, ECS cluster, service, task definition, log group) | Docker volume `floci_floci-data` (persistent mode) |
| Pushed images | Docker volume `floci-ecr-registry-data` |
| Local image `goals:stage-01` | The server's Docker image store |

### Don't use these (they delete data)

| Command | What it destroys |
|---------|------------------|
| `docker compose down -v` / `make down` | The Postgres volume, i.e. the whole database |
| `docker compose -f floci/docker-compose.yml down -v` | All Floci state (secrets, ECS setup, …) |
| `docker system prune --all` | Unused images across the **whole shared server**, including other apps' |

## Start up

Follow this order:

```bash
# 1. Postgres first. Both app copies need it.
docker compose start

# 2. Floci. It reuses its existing registry container.
docker compose -f floci/docker-compose.yml start
docker start floci-ecr-registry 2>/dev/null || true   # in case Floci didn't restart it

# 3. The docker-run copy of the app (optional)
docker start goals

# 4. The ECS copy of the app
source floci/env.sh
aws ecs update-service --cluster fem-fd-service --service goals --desired-count 1 \
  --query "service.desiredCount" --output text
```

Then check with [Check status](#check-status).

## Open the app from the laptop

Use one tunnel at a time, since both use laptop port 8080.

| App copy | Tunnel (run on the laptop) |
|----------|----------------------------|
| docker-run copy | `ssh -L 8080:localhost:8090 coz@coz` |
| ECS copy | `ssh -L 8080:<task-ip>:8080 coz@coz` |

The ECS task's IP **can change** each time a task starts. Look it up on the server:

```bash
docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' \
  $(docker ps -q --filter name=floci-ecs-)
```

Then open **http://localhost:8080** in the laptop browser.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| The app logs `connection refused` to the DB | Postgres isn't running | `docker compose start` |
| `aws …` errors with `Could not connect to the endpoint URL` | Floci isn't running, or `env.sh` isn't loaded | `docker compose -f floci/docker-compose.yml start`, then `source floci/env.sh` |
| The ECS service shows `1 0` (wants 1, runs 0) | The task failed to start | `aws logs tail /ecs/fem-fd-service` and `docker ps -a --filter name=floci-ecs-` |
| The ECS task can't pull its image | The registry container is stopped | `docker start floci-ecr-registry` |
| The ECS task can't reach the DB after a Floci rebuild | `floci_default` got a new subnet | Get the gateway with `docker network inspect floci_default --format '{{range .IPAM.Config}}{{.Gateway}}{{end}}'`, update `/fem-fd-service/postgres-url` in SSM, then `aws ecs update-service --cluster fem-fd-service --service goals --force-new-deployment` |
| The tunnel says `bind: Address already in use` | Another tunnel already uses laptop port 8080 | Close the other tunnel |
| Google says `redirect_uri_mismatch` | The browser isn't on `localhost:8080` | Use one of the tunnels above, and open `http://localhost:8080` |
