# Implementation 2: Stage 01 on Floci (ECR, SSM, ECS)

## Goal

Do the "cloud" half of stage 01:
- store the app image in a **registry** (ECR),
- keep the secrets in a **secret store** (SSM Parameter Store) instead of a `.env` file,
- let a **container service** run the app from those two.

The course uses **App Runner** for the last part. Floci doesn't have App Runner, so we use **ECS (Fargate)**, the service stage 03 uses anyway.

## Status (2026-09-28)

| # | Step | Status |
|---|------|--------|
| 1 | Floci settings for ECR and ECS | ✅ Done (after two problems, see [Problems](#problems-we-hit)) |
| 2 | ECR repo `fem-fd-service-preview`, image pushed | ✅ `stage-01` tag, same digest as the local build |
| 3 | Secrets in SSM Parameter Store | ✅ 3 × `SecureString` under `/fem-fd-service/` |
| 4 | IAM execution role | ✅ `fem-fd-service-execution` |
| 5 | CloudWatch log group | ✅ `/ecs/fem-fd-service` |
| 6 | ECS cluster, task definition, service | ✅ The task is RUNNING and connected to the DB, and serves `200` |
| 7 | Log in to the ECS-run app from the laptop | ⬜ You do this |

## How it fits together

```
                    ┌──────── Floci (fake AWS) ─────────────────────────────┐
 docker push ─────▶ │ ECR  fem-fd-service-preview:stage-01                  │
                    │   │                                                   │
 aws ssm put ─────▶ │ SSM  /fem-fd-service/{google-client-id, …secret,      │
                    │   │                   postgres-url}                   │
                    │   ▼        (IAM execution role allows reading both)   │
                    │ ECS  cluster fem-fd-service → service goals → 1 task  │
                    │   │                                                   │
                    │   ▼                                                   │
                    │ CloudWatch Logs  /ecs/fem-fd-service                  │
                    └───┼───────────────────────────────────────────────────┘
                        ▼ Floci starts a real container
          floci-ecs-<task-id>-goals  (network floci_default, e.g. 10.0.7.4:8080)
                        │ SQL
                        ▼
          Postgres on the server  (10.0.7.1:5432, the host as seen from floci_default)
```

| AWS piece | What it does here |
|-----------|-------------------|
| **ECR** repository | Holds the image. ECS pulls from here, not from the server's local images. |
| **SSM** `SecureString` parameters | Hold the secrets. ECS reads them at start-up and injects them as environment variables. The app code doesn't change. |
| **IAM** execution role | The permission ECS uses to pull the image, read the three parameters and write logs |
| **CloudWatch Logs** | Collects the app's stdout/stderr (the `awslogs` driver) |
| **ECS** cluster, task definition, service | The **task definition** is the recipe: image, CPU/memory, env, secrets, logs. The **service** keeps 1 copy running and replaces it if it dies. |

## What we did

All commands run on the server from `~/cloud-infra-learning` after `source floci/env.sh`.

### 1. Floci settings

We added these three settings to [`floci/docker-compose.yml`](../../floci/docker-compose.yml), then ran `docker compose -f floci/docker-compose.yml up -d` (state is kept):

```yaml
      FLOCI_SERVICES_ECR_URI_STYLE: path
      FLOCI_SERVICES_DOCKER_NETWORK: floci_default
      FLOCI_SERVICES_ECS_DOCKER_NETWORK: floci_default
```

Why each one is needed is under [Problems](#problems-we-hit) 1 and 2.

### 2. ECR: create the repo and push

```bash
aws ecr create-repository --repository-name fem-fd-service-preview
# repositoryUri: localhost:4566/000000000000/us-west-2/fem-fd-service-preview

aws ecr get-login-password | docker login --username AWS --password-stdin localhost:4566
# Login Succeeded

docker tag goals:stage-01 localhost:4566/000000000000/us-west-2/fem-fd-service-preview:stage-01
docker push              localhost:4566/000000000000/us-west-2/fem-fd-service-preview:stage-01
# stage-01: digest: sha256:b9aac437…  (same digest as the local image)
```

The repo name `fem-fd-service-preview` matches what the course's stage 02 makefile and stage 03 Terraform expect.

### 3. SSM: store the secrets

The values are read from `.env` and never printed. The DB URL uses `10.0.7.1` (see [Problem 3](#3-the-task-cant-use-localhost-for-postgres)):

```bash
( set -a; . ./.env; set +a
  aws ssm put-parameter --type SecureString --overwrite --name /fem-fd-service/google-client-id     --value "$GOOGLE_CLIENT_ID"
  aws ssm put-parameter --type SecureString --overwrite --name /fem-fd-service/google-client-secret --value "$GOOGLE_CLIENT_SECRET"
  aws ssm put-parameter --type SecureString --overwrite --name /fem-fd-service/postgres-url \
    --value "postgresql://postgres:password@10.0.7.1:5432/postgres?sslmode=disable" )
aws ssm describe-parameters --query "Parameters[].[Name,Type]" --output text
```

`GOOGLE_REDIRECT_URL` isn't secret, so it goes in the task definition as a plain environment variable, the same split stage 03 uses.

### 4. IAM, logs, cluster, task definition, service

The JSON files are in [`floci/stage-01/`](../../floci/stage-01/):

| File | Contents |
|------|----------|
| `execution-role-trust.json` | Lets ECS tasks (`ecs-tasks.amazonaws.com`) assume the role |
| `execution-role-policy.json` | `ssm:GetParameters` on `/fem-fd-service/*`, ECR pull, and CloudWatch log writes. The Floci version of the course's `stage-01-apprunner-iam.json`. |
| `task-definition.json` | Fargate, 256 CPU / 512 MB, image from ECR, env + secrets, `awslogs` |

```bash
cd floci/stage-01
aws iam create-role --role-name fem-fd-service-execution --assume-role-policy-document file://execution-role-trust.json
aws iam put-role-policy --role-name fem-fd-service-execution --policy-name fem-fd-service-execution \
  --policy-document file://execution-role-policy.json
aws logs create-log-group --log-group-name /ecs/fem-fd-service
aws ecs create-cluster --cluster-name fem-fd-service
aws ecs register-task-definition --cli-input-json file://task-definition.json
aws ecs create-service --cluster fem-fd-service --service-name goals \
  --task-definition fem-fd-service --desired-count 1 --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={subnets=[subnet-default-us-west-2-a],securityGroups=[sg-default-us-west-2],assignPublicIp=DISABLED}"
```

Floci comes with a default VPC (`172.31.0.0/16`), a subnet in each zone and a default security group, so no network setup was needed.

## Result

```
$ aws ecs describe-tasks … → lastStatus: RUNNING
$ docker ps --filter label=floci=true
floci-ecs-b90a4559…-goals   Up   floci_default      ← the ECS task, a real container
floci-ecr-registry          Up   floci_default      ← Floci's ECR storage

env injected into the task:  GOOGLE_REDIRECT_URL, GOOGLE_CLIENT_ID, GOOGLE_CLIENT_SECRET, POSTGRES_URL
docker logs / CloudWatch:    Successfully connected to the database
                             Server starting on http://localhost:8080
curl http://10.0.7.4:8080/                 → 200
curl http://10.0.7.4:8080/auth/google/login → 307 (to Google)
```

The database connection proves the secrets flowed **SSM → ECS → app**.

## Log in to the ECS-run app

The task isn't published on a host port. That's intentional, because the server's 8080 is taken. Instead, tunnel straight to the task's IP on `floci_default`:

```bash
# on the server: find the current task IP (it changes whenever the task is replaced)
docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' \
  $(docker ps -q --filter label=floci=true --filter name=floci-ecs-)

# on the laptop (close any other tunnel using local port 8080 first)
ssh -L 8080:<task-ip>:8080 coz@coz
# → http://localhost:8080
```

## Problems we hit

### 1. The ECR hostname doesn't resolve

**Symptom:** Floci's default ECR address, `000000000000.dkr.ecr.us-west-2.localhost:4566`, doesn't resolve on the server:

```
$ getent hosts 000000000000.dkr.ecr.us-west-2.localhost
(no output)
socket.gaierror: [Errno -2] Name or service not known
```

**Cause:** Floci relies on `*.localhost` resolving to 127.0.0.1 (RFC 6761). Many systems do that through systemd-resolved, but this server uses plain `hosts: files dns` in `/etc/nsswitch.conf`, and neither source knows the name.

**Fix:** use `FLOCI_SERVICES_ECR_URI_STYLE: path`. Image addresses become `localhost:4566/<account>/<region>/<repo>`. Plain `localhost` always resolves, and Docker trusts loopback registries without TLS. We rejected adding a line to `/etc/hosts`, because that changes the shared server.

**Side effect:** our addresses don't look like real ECR (`<account>.dkr.ecr.<region>.amazonaws.com/<repo>`). When the stage 02 makefile builds its image name, override `AWS_ECR_DOMAIN=localhost:4566/000000000000/us-west-2`.

### 2. ECS needs its own Docker network

**Cause:** Floci's docs say ECS task containers need `FLOCI_SERVICES_ECS_DOCKER_NETWORK` set to a created network, because the default `bridge` won't work. Tasks also aren't published on host ports by default.

**Fix:** use Floci's own compose network, `floci_default` (`10.0.7.0/24`). We left host-port publishing off (`FLOCI_SERVICES_ECS_PUBLISH_AWSVPC_PORTS_TO_HOST`), because it would bind the container port, **8080**, on the host, and the server's 8080 is taken.

### 3. The task can't use `localhost` for Postgres

**Cause:** inside the task container, `localhost` is the container itself. In implementation 0, we fixed this with `--add-host host.docker.internal:host-gateway`. The ECS equivalent, `extraHosts`, is **rejected for Fargate** task definitions.

**Fix:** `POSTGRES_URL` in SSM points at `10.0.7.1`, the server's address on `floci_default`. Postgres runs with host networking and listens on all interfaces, so the task can reach it there.

**Caveat:** if `floci_default` is ever recreated with a different subnet, update the parameter. Find the address with `docker network inspect floci_default --format '{{range .IPAM.Config}}{{.Gateway}}{{end}}'`. Stage 03 replaces this with a proper RDS database.

### 4. The config edit failed on a quote

**Symptom:** a one-line edit to `floci/docker-compose.yml`, run over SSH, failed with `SyntaxError: unterminated triple-quoted string literal`.

**Cause:** the comment contained an apostrophe ("doesn't"). It closed the shell's single-quoted command early.

**Fix:** we checked with `git status` that nothing had changed, then wrote the whole file locally and copied it over with `scp`. **Lesson:** for anything more than a one-liner, copy a file rather than building it inside a quoted SSH command.

### 5. Floci's SecureString isn't encrypted on read ⚠️

**Symptom:** to check that the secret was stored encrypted, we read it back **without** `--with-decryption`. We expected encrypted text, as real AWS returns. Floci returned the **plain value**, and the first 20 characters of the Google client secret were printed in our session output. It was never committed or written to a file.

**Cause:** Floci stores `SecureString` parameters but doesn't encrypt them on read.

**Fix and lesson:**
- Rotate the Google client secret: Google Cloud → the OAuth client → **Add secret**, update `.env` and SSM, then disable the old secret.
- **Never print a parameter value to check it**, even an "encrypted" one. Check the name and type, or the length only:
  ```bash
  v=$(aws ssm get-parameter --name /fem-fd-service/google-client-secret --with-decryption --query Parameter.Value --output text); echo ${#v}
  ```

### 6. Floci has no App Runner

The course's stage 01 runs the image on App Runner, which isn't in Floci's service list. We used ECS Fargate instead. It covers the same ideas: run an image from a registry, with secrets from Parameter Store. Stage 03 uses ECS anyway.

## Heads-up for stage 03

This server's Docker networks use the `10.0.x.0/24` range (`docker0` is `10.0.0.1/24`, `floci_default` is `10.0.7.0/24`). The course's Terraform VPC uses **`10.0.0.0/16`**, which overlaps all of them. If Floci turns VPC ranges into Docker networks, that could clash. Plan: use a different VPC range for Floci, for example `10.100.0.0/16`.

## Handy commands

```bash
aws ecs describe-services --cluster fem-fd-service --services goals --query "services[0].[status,runningCount]"
aws ecs list-tasks --cluster fem-fd-service
aws logs tail /ecs/fem-fd-service --follow                   # app logs via CloudWatch
aws ecs update-service --cluster fem-fd-service --service goals --force-new-deployment   # restart with a fresh task
docker ps --filter label=floci=true                          # containers Floci started
```

## Next

- Log in through the ECS-run app (step 7), then rotate the Google client secret (Problem 5).
- Implementation 3: stage 02, which adds the makefile, goose migrations, the multi-stage Dockerfile and CI.
