# Services overview

What's running, and the role each piece plays. Updated after each implementation step.

**As of:** [implementation 2](implementation/implementation-2.md) (2026-09-28)

## Summary

The app now runs two ways, and both use the same Postgres:
1. **Stage 01 local:** our own `docker run` container on server port 8090, with secrets from `.env` ([implementation 0](implementation/implementation-0.md)).
2. **Stage 01 on Floci:** an **ECS** task that pulls the image from **ECR** and gets its secrets from **SSM Parameter Store**, using an **IAM** role. Its logs go to **CloudWatch Logs** ([implementation 2](implementation/implementation-2.md)).

## What's running

```
Laptop browser  http://localhost:8080
   │
   ├── tunnel A ─▶ server :8090 ─────────────▶ goals (docker run, .env)            ─┐
   │                                                                                 │ SQL
   └── tunnel B ─▶ task IP :8080 (floci_default) ▶ floci-ecs-…-goals (ECS task)     ─┤ :5432
                                                    ▲ image from ECR                 ▼
                                                    ▲ secrets from SSM          Postgres 17
                                                    ▼ logs to CloudWatch
                     ┌──────────────── Floci :4566 ────────────────┐
                     │ ECR · SSM · IAM · ECS · CloudWatch Logs     │
                     └─────────────────────────────────────────────┘
Both app copies ◀──── login ────▶ Google OAuth
```

Use one tunnel at a time, since both use laptop port 8080.

| Service | Where | Role | In use? |
|---------|-------|------|---------|
| **goals app, local** (`goals:stage-01`) | Container `goals`, server port 8090 | The web app, run by hand with `.env` | ✅ Still running |
| **goals app, ECS** | Container `floci-ecs-<task-id>-goals` on `floci_default` | The same image, run by ECS from ECR, with secrets from SSM | ✅ Running |
| **Postgres 17** | Container from `docker-compose.yml`, port 5432 | Stores users, posts, comments, likes, followers and admins. Shared by both app copies. | ✅ Yes |
| **Google OAuth** | Google Cloud (real internet) | Login | ✅ Yes |
| **SSH tunnel** | Laptop → server | Browser access as `localhost:8080` | ✅ Yes |
| **Floci** | Container `floci`, port 4566 | Local stand-in for AWS | ✅ Now in use |
| **GitHub** | github.com | Code and docs. CI starts using it in stage 02. | Only for git |

## AWS services in use (in Floci)

| AWS service | Resource | Role |
|-------------|----------|------|
| **ECR** | Repo `fem-fd-service-preview`, tag `stage-01` | Stores the app image. ECS pulls from here. |
| **SSM Parameter Store** | `/fem-fd-service/google-client-id`, `…/google-client-secret`, `…/postgres-url` (`SecureString`) | Stores the secrets, injected into the app as environment variables at start-up |
| **IAM** | Role `fem-fd-service-execution` | Lets ECS pull the image, read those three parameters and write logs |
| **ECS** | Cluster `fem-fd-service`, service `goals`, task definition `fem-fd-service:1` (Fargate) | Runs and supervises the app container (the course uses App Runner, which Floci doesn't have) |
| **CloudWatch Logs** | Log group `/ecs/fem-fd-service` | Collects the app's logs |

## Installed tools

| Tool | Role | In use? |
|------|------|---------|
| **AWS CLI** | Sends commands to Floci (`source floci/env.sh` first) | ✅ Since implementation 2 |
| **goose** | Database migrations | Stage 02 |
| **Terraform** | Builds the infrastructure from code | Stage 03 |

## What changes next (stage 02)

- **goose migrations** replace the hand-pasted SQL.
- A **multi-stage Dockerfile** shrinks the 553 MB image.
- A **makefile** wraps build, push and migrate.
- **GitHub Actions CI** builds and tests on every push. It needs a self-hosted runner on the server to reach Floci.
