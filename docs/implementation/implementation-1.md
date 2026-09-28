# Implementation 1: Tools and Floci

## Goal

Install the CLI tools the course needs, and run Floci as our local "AWS", so the AWS CLI and Terraform have something to talk to.

## Status (2026-09-28)

| # | Step | Status |
|---|------|--------|
| 1 | `unzip` | ⏭️ Skipped. Used `busybox unzip` instead (no sudo needed). |
| 2 | `~/.local/bin` on `PATH` | ✅ Already set by `~/.bash_profile` |
| 3 | AWS CLI v2 | ✅ `aws-cli/2.37.4` |
| 4 | goose | ✅ `v3.28.0` |
| 5 | Terraform | ✅ `v1.16.4` (checksum verified) |
| 6 | Floci running | ✅ `floci 2.1.0`, healthy |
| 7 | `floci/env.sh` (points the CLI at Floci) | ✅ |
| 8 | Smoke test (STS, S3, ECR, SSM, EC2) | ✅ All passed |

**Implementation 1 is complete.**

Everything installs into the home folder (`~/.local`, `~/go`). Nothing system-wide changed, and nothing already on the server was touched.

## What we did

### 1. unzip: used busybox instead

The AWS CLI and Terraform ship as zip files. `unzip` isn't installed, and `sudo apt install unzip` would need a password prompt. The server already has **busybox**, which includes an `unzip` that keeps executable permissions:

```bash
busybox unzip -q file.zip
```

### 2. PATH

Our SSH login shell reads **`~/.bash_profile`**, which already runs `export PATH="$HOME/.local/bin:$PATH"`. It does **not** load `~/.bashrc`, and `~/.bashrc` is where `~/go/bin` gets added. So `go install`-ed tools (like goose) aren't found in a normal SSH session.

**Fix:** instead of editing the shell files, we symlinked goose into `~/.local/bin` (see step 4).

### 3. AWS CLI v2

```bash
cd /tmp && mkdir awscli-inst && cd awscli-inst
curl -fsSLo awscliv2.zip https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip
busybox unzip -q awscliv2.zip
./aws/install -i ~/.local/aws-cli -b ~/.local/bin     # -i/-b: install into home, no sudo
cd /tmp && rm -rf awscli-inst
aws --version
# aws-cli/2.37.4 Python/3.14.6 Linux/6.12.94+deb13-amd64 exe/x86_64.debian.13
```

### 4. goose

```bash
go install github.com/pressly/goose/v3/cmd/goose@latest   # → ~/go/bin/goose
ln -sfn ~/go/bin/goose ~/.local/bin/goose                 # make it visible in login shells
goose -version
# goose version: v3.28.0
```

### 5. Terraform

We got the latest version from HashiCorp's API (`curl -s https://checkpoint-api.hashicorp.com/v1/check/terraform | jq -r .current_version`), then installed it:

```bash
cd /tmp && TF=1.16.4
curl -fsSLO https://releases.hashicorp.com/terraform/${TF}/terraform_${TF}_linux_amd64.zip
curl -fsSL https://releases.hashicorp.com/terraform/${TF}/terraform_${TF}_SHA256SUMS \
  | grep linux_amd64.zip | sha256sum -c -                    # terraform_1.16.4_linux_amd64.zip: OK
busybox unzip -q -o terraform_${TF}_linux_amd64.zip terraform -d ~/.local/bin
rm terraform_${TF}_linux_amd64.zip
terraform version
# Terraform v1.16.4
```

### 6. Floci

[`floci/docker-compose.yml`](../../floci/docker-compose.yml):

```yaml
name: floci
services:
  floci:
    image: floci/floci:latest
    container_name: floci
    restart: unless-stopped
    ports:
      - "127.0.0.1:4566:4566"             # the AWS API (all services)
      - "127.0.0.1:7001-7099:7001-7099"   # RDS databases
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock   # lets Floci run ECS/EC2/RDS containers
      - floci-data:/app/data
    environment:
      FLOCI_DEFAULT_REGION: us-west-2     # the course hardcodes us-west-2
      FLOCI_STORAGE_MODE: persistent      # survive restarts
      FLOCI_STORAGE_PERSISTENT_PATH: /app/data
volumes:
  floci-data:
```

```bash
docker compose -f floci/docker-compose.yml up -d
docker logs floci --tail 20
# floci 2.1.0 native (powered by Quarkus 3.39.2) started in 0.021s. Listening on: http://0.0.0.0:4566
```

| Choice | Why |
|--------|-----|
| `127.0.0.1:` on ports | Only reachable from the server itself, not the network. The server is shared. |
| Docker socket mount | Floci runs **real** containers for ECS tasks, EC2 instances and RDS databases. They show up in `docker ps`. |
| `FLOCI_DEFAULT_REGION: us-west-2` | The course's makefile and Terraform hardcode `us-west-2` and the zones `us-west-2a/b/c` |
| `persistent` storage | Resources survive a Floci restart |
| No port 8081 yet | The ALB listener is only needed in stage 03. We'll add it then. |

The ports we checked before starting were all free: 4566 and 7001–7099, plus the container name `floci`.

### 7. CLI environment

[`floci/env.sh`](../../floci/env.sh):

```bash
export AWS_ENDPOINT_URL=http://localhost:4566
export AWS_ACCESS_KEY_ID=test
export AWS_SECRET_ACCESS_KEY=test
export AWS_DEFAULT_REGION=us-west-2
```

These are dummy credentials, so the file is committed. **Run `source floci/env.sh` from the repo root** before any `aws` or `terraform` command.

### 8. Smoke test

```bash
source floci/env.sh
aws sts get-caller-identity
# { "UserId": "000000000000", "Account": "000000000000", "Arn": "arn:aws:iam::000000000000:root" }

aws s3 mb s3://floci-smoke-test && aws s3 ls && aws s3 rb s3://floci-smoke-test
# make_bucket: floci-smoke-test
# 2026-09-28 18:27:33 floci-smoke-test
# remove_bucket: floci-smoke-test

aws ecr describe-repositories                 # { "repositories": [] }
aws ssm describe-parameters                   # { "Parameters": [] }
aws ec2 describe-availability-zones --query "AvailabilityZones[].ZoneName" --output text
# us-west-2a  us-west-2b  us-west-2c
```

## Key facts for later

| Thing | Value |
|-------|-------|
| Account ID | `000000000000`. The course hardcodes `677459762413`, so override it. |
| Region | `us-west-2` |
| API endpoint | `http://localhost:4566` |
| ECR registry host | `000000000000.dkr.ecr.us-west-2.localhost:4566` |
| Enabled services (relevant) | ecr, ecs, ec2, rds, ssm, iam, elbv2, cloudfront, autoscaling, cloudwatchlogs, s3 |

## Handy commands

```bash
docker logs floci --tail 50                           # Floci log
docker compose -f floci/docker-compose.yml restart    # restart (state is kept)
docker compose -f floci/docker-compose.yml down       # stop (state is kept in the floci-data volume)
```

## Next

Implementation 2: stage 01 on Floci. Push the `goals` image to Floci's ECR, store the secrets in SSM Parameter Store, and run the image from the registry. The course uses App Runner, which Floci doesn't have, so we'll use ECS instead.
