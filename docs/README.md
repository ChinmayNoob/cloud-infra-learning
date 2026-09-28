# Learning log

Notes from working through **[Fullstack Deployment: From Containers to Production AWS](https://frontendmasters.com/courses/fullstack-deployment)** (Frontend Masters), using the course app [ALT-F4-LLC/fem-fd-service](https://github.com/ALT-F4-LLC/fem-fd-service).

Instead of a real AWS account, everything runs against **[Floci](https://floci.io/)**, a local AWS emulator hosted on a home server.

**Current state:** see the [Services overview](services-overview.md) for what's running and the role each piece plays.

## Setup

Each file covers one step, in the order it happened.

| # | File | Status |
|---|------|--------|
| 00 | [Overview](00-overview.md): the course, its stages, and the Floci swap | Done |
| 01 | [Repo and git setup](01-repo-and-git-setup.md): this repo, and pushing from the server | Done |
| 02 | [Prerequisites](02-prerequisites.md): accounts, tools, ports | In progress |
| 03 | [Course code](03-course-code.md): stage-01 code imported into this repo | Done |

## Implementation log

The build work goes in [`implementation/`](implementation/), one file per step, starting at `implementation-0`.

| # | File | Status |
|---|------|--------|
| 0 | [Stage 01: run the app locally](implementation/implementation-0.md) (Postgres, tables, app container, Google login) | Done |
| 1 | [Tools and Floci](implementation/implementation-1.md) (AWS CLI, goose, Terraform, Floci) | Done |
| 2 | [Stage 01 on Floci](implementation/implementation-2.md) (ECR, SSM Parameter Store, IAM, ECS, CloudWatch Logs) | Done (login check pending) |
| 3 | Stage 02: makefile, goose migrations, CI | Next |
| 4 | Stage 03: Terraform (VPC, RDS, ECS, ALB, CloudFront) | Planned |

## Template for each file

- **Goal**: what this step is for
- **What we did**: the steps, with commands
- **Result**: what worked, and what the output looked like
- **Issues**: what broke, and how it was fixed
- **Next**: what comes after
