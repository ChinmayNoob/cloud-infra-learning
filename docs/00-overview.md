# 00: Overview

## Goal

Learn how to take a containerised web app from local development to production on AWS: containers, a registry, secrets, CI/CD, and infrastructure as code with Terraform. The AWS parts run locally on Floci instead of a paid account.

## The course app

`goals` is a Go web app for sharing life goals. It uses Google OAuth2 for login and PostgreSQL for data.

The app code **never calls AWS**. It only reads environment variables:

| Variable | Used for |
|----------|----------|
| `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET`, `GOOGLE_REDIRECT_URL` | Google login |
| `POSTGRES_URL` | Database connection |
| `GOOSE_DBSTRING`, `GOOSE_DRIVER` | Database migrations (from stage 02) |

AWS only appears in how the app is built, stored, configured and run. The app listens on the fixed port `:8080`, which is hardcoded in `main.go`.

## Course stages (branches of fem-fd-service)

| Branch | What it adds | AWS services used |
|--------|--------------|-------------------|
| `stage-01-start-up` | Go app, `Dockerfile`, `docker-compose.yml` (local Postgres) | Done by hand in the course: **ECR** (image registry), **SSM Parameter Store** (secrets), **App Runner** (runs the container). The database is Supabase. |
| `stage-02-growth` | `makefile`, goose migrations, multi-stage Dockerfile, GitHub Actions | **ECR** login, push, pull and tag promotion from CI |
| `stage-03-scale` | `terraform/`, `deploy.sh`, Terraform CI workflow | **S3** (state), **VPC** + NAT, **security groups**, **EC2** bastion, **RDS** Postgres, **ECS** on an EC2 spot **Auto Scaling** group, internal **ALB**, **CloudFront** (VPC origin), **SSM**, **CloudWatch Logs**, **IAM/STS** |
| `workshop`, `main`, `prod` | The finished course version | Same as stage 03 |

## AWS → Floci mapping

| AWS service | Floci | Notes |
|-------------|-------|-------|
| ECR | ✅ Real registry | Push to `000000000000.dkr.ecr.<region>.localhost:4566/<repo>` from the Floci host |
| App Runner | ❌ Not supported | Stage 01's runtime. Use ECS or a plain `docker run` instead. |
| SSM Parameter Store | ✅ | |
| IAM, STS, CloudWatch Logs | ✅ | |
| S3 (Terraform state) | ✅ | Needs path-style addressing |
| VPC, subnets, NAT, EC2 | ✅ | Instances are Docker containers. **Security groups are not enforced.** |
| RDS Postgres | ✅ Real Postgres in Docker | Default is Postgres 16. Connect on ports 7001–7099. |
| ECS | ✅ Runs real containers | Pulls from Floci ECR, supports SSM secrets and `awslogs` |
| Auto Scaling → ECS container instances | ⚠️ Undocumented | Fallback: switch to Fargate |
| ALB (ELBv2) | ✅ Proxies HTTP | Reached via `{name}-{id}.elb.localhost.floci.io` |
| CloudFront VPC origin | ❌ Not supported | Use a custom origin pointing at the ALB, or skip CloudFront |

Sources: [Floci docs](https://floci.io/floci/), [Terraform with Floci](https://floci.io/floci/getting-started/terraform/).

## Our setup

```
Laptop (Windows) ──ssh──▶  server "coz" (Debian 13, x86_64, Docker Engine)
                             ├── ~/cloud-infra-learning   ← this repo: course code at the root + docs/
                             └── Floci container          ← fake AWS on :4566
```

## Decisions

- **Floci instead of real AWS.** No cloud bill and no account needed.
- **The course code lives in this repo, at the root.** It was copied unchanged from `stage-01-start-up`, so the course's commands work as is. Each later stage is added as our own commits (see [03](03-course-code.md)).
- **Stage 01 has no App Runner.** We run the pushed image with ECS or Docker instead.
- **Google OAuth still needs the real Google Cloud.** Floci only covers AWS.
- **Nothing already on the server gets changed.** It's shared with other apps, so the course moves to free ports (app on `8090`, ALB on `8081`) instead. See [02: Prerequisites](02-prerequisites.md#ports).

## Next

[01: Repo and git setup](01-repo-and-git-setup.md)
