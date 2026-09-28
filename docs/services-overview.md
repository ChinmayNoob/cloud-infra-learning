# Services overview

What's running, and the role each piece plays. Updated after each implementation step.

**As of:** [implementation 1](implementation/implementation-1.md) (2026-09-28)

## Summary

Right now the app runs on plain Docker and Google. Floci is running but idle: no AWS resources exist in it yet. The only AWS calls so far were the smoke tests, and the test bucket was deleted afterwards.

## What's running

```
Laptop browser
   │  http://localhost:8080
   ▼
SSH tunnel ──────────▶ server :8090
                          │
                          ▼
                   ┌──────────────┐   login redirect    ┌──────────────┐
                   │ goals app    │ ◀─────────────────▶ │ Google OAuth │
                   │ (container)  │                     └──────────────┘
                   └──────┬───────┘
                          │ SQL (:5432)
                          ▼
                   ┌──────────────┐
                   │ Postgres 17  │
                   │ (container)  │
                   └──────────────┘

                   ┌──────────────┐
                   │ Floci        │  ← running on :4566, nothing uses it yet
                   └──────────────┘
```

| Service | Where | Role | Used by the app now? |
|---------|-------|------|----------------------|
| **goals app** (`goals:stage-01`) | Docker container `goals`, server port 8090 | The Go web app: pages, profiles, posts, likes, follows, admin | It *is* the app |
| **Postgres 17** | Docker container from `docker-compose.yml`, port 5432 | Stores users, posts, comments, likes, followers and admins | ✅ Yes |
| **Google OAuth** | Google Cloud (real internet) | Login: Google confirms who you are and sends back your email and name | ✅ Yes, on every login |
| **SSH tunnel** | Laptop → server | Lets the browser reach the app as `localhost:8080`, which Google accepts as a redirect address | ✅ Yes, how we access the app |
| **Floci** | Docker container `floci`, port 4566 | Local stand-in for AWS | ❌ Running but idle |
| **GitHub** | github.com | Stores the code and docs. CI starts using it in stage 02. | Only for git |

### Installed tools (not in use yet)

| Tool | Role | Used from |
|------|------|-----------|
| **AWS CLI** | Sends commands to Floci, the same way it would to real AWS | Implementation 2 |
| **Terraform** | Builds whole infrastructure from code | Stage 03 |
| **goose** | Database migrations. Replaces the SQL we pasted by hand. | Stage 02 |

## What changes in implementation 2

The app stays the same. What changes is **where the image comes from and where the secrets live**. Today the image is built locally and the secrets sit in a `.env` file. That's the "start-up" way the course moves away from.

| AWS service (in Floci) | Role in implementation 2 | Replaces |
|------------------------|--------------------------|----------|
| **ECR** (container registry) | Stores the `goals` image. Anything that runs the app pulls it from here. | The image that only exists on the server |
| **SSM Parameter Store** | Stores the four secrets, and hands them to the app when it starts | The `.env` file |
| **ECS** (container runner) | Pulls the image from ECR, injects the secrets from SSM, and runs the container | Our manual `docker run`. The course uses App Runner, which Floci doesn't have. |
| **IAM** | Gives ECS permission to read the image and the secrets | Nothing (new) |
| **CloudWatch Logs** | Collects the app's logs | `docker logs goals` |

Postgres stays the same local container for now. Stage 03 moves it to **RDS** (a managed database), along with the VPC, load balancer and CloudFront.
