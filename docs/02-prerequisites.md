# 02: Prerequisites

## Goal

Have every account, tool and port ready before starting stage 01.

## Accounts

| Account | Needed for | Status |
|---------|------------|--------|
| **Google Cloud Console** | OAuth client for app login (stage 01 onward) | To do |
| GitHub (ChinmayNoob) | This repo, and CI in stage 02 | ✅ Done ([01](01-repo-and-git-setup.md)) |
| ~~AWS~~ | Replaced by Floci | Not needed |
| ~~Supabase~~ | Replaced by local Postgres or Floci RDS | Not needed |

### Google OAuth client

1. Open [Google Cloud Console](https://console.cloud.google.com/) and create a project, for example `cloud-infra-learning`.
2. Go to **APIs & Services → OAuth consent screen** (Google Auth Platform). Choose **External**, fill in the app name and your email, and leave it in **Testing** mode.
3. Under **Audience → Test users**, add the Google account you'll log in with. In Testing mode, only listed test users can sign in.
4. Go to **Clients → Create client**, choose **Web application**, and name it `goals-local`.
5. Add this **Authorized redirect URI**: `http://localhost:8080/auth/google/callback`
6. Save the **Client ID** and **Client Secret**. They go in `.env` later, and are never committed.

**Why `localhost`?** Google only allows plain `http` redirect URIs for `localhost`. The app runs on the server, so we'll reach it from the laptop through an SSH tunnel (see [Ports](#ports)). That way the browser still sees `localhost:8080`.

Stage 03 serves the app from a CloudFront or ALB hostname. That will need its own redirect URI, and probably HTTPS. We'll deal with it when we get there.

## Tools on the server

Checked on 2026-09-28 (Debian 13 "trixie", x86_64, 8 cores, 30 GB RAM):

| Tool | Needed for | Status |
|------|------------|--------|
| Docker Engine | Everything | ✅ 29.6.2 |
| Docker Compose | Local Postgres, Floci | ✅ v5.3.1 |
| Docker buildx | `make build-image` | ✅ v0.35.0 |
| Go ≥ 1.24.2 | Building the app | ✅ 1.24.4 |
| make | Stage 02+ | ✅ 4.4.1 |
| git | Everything | ✅ 2.47.3 |
| jq | `deploy.sh` | ✅ 1.7 |
| curl | Installs | ✅ |
| **goose** | DB migrations | ✅ v3.28.0 (installed in [implementation 1](implementation/implementation-1.md)) |
| **AWS CLI v2** | Talking to Floci | ✅ 2.37.4 (implementation 1) |
| **Terraform** | Stage 03 | ✅ v1.16.4 (implementation 1) |
| unzip | AWS CLI installer | Not needed. `busybox unzip` is used instead. |
| psql (client) | Poking at the DB (optional) | Not needed. `psql` runs inside the Postgres container. |
| Floci | Fake AWS | ✅ Running (implementation 1) |

The user is in the `docker` group, so Docker works without `sudo`.

## Tools on the laptop

- An SSH client (built into Windows) and a browser. Nothing else is needed.
- For the tunnel, run `ssh -L 8080:localhost:8090 coz@coz` and open `http://localhost:8080`.

## Ports

> **Rule: nothing already on the server gets changed.** The course moves to free ports instead.

The server is **shared**. A reverse proxy for other apps already holds some ports, so the course uses these (all checked free on 2026-09-28):

| Course port | Used for | Why not the course default |
|-------------|----------|----------------------------|
| **8090** | The app. The container still listens on `:8080` internally, so we run it with `-p 8090:8080`. | Host `8080` is taken |
| **8081** | Floci's ALB listener (stage 03), mapped to the listener's port 80 | Host `80` and `443` are taken |
| 5432 | Local Postgres (compose uses `network_mode: host`) | Default is free |
| 4566 | Floci API | Default is free |
| 7001–7099 | Floci RDS instances | Default is free |

To use the app from the laptop, tunnel laptop port `8080` to server port `8090`:

```bash
ssh -L 8080:localhost:8090 coz@coz
# then open http://localhost:8080 in the laptop browser
```

This keeps the Google redirect URI at `http://localhost:8080/auth/google/callback`, exactly as the course expects.

## Shared-server safety rules

The server runs other apps, so some course commands need care:

- **Never run `docker system prune --all`.** The course's troubleshooting tips suggest it, but it deletes every unused image on the server, including ones other apps need in order to restart. Remove course images by name instead, e.g. `docker image rm <image>`.
- `make down` runs `docker compose down --volumes`. That is scoped to the compose project in the current folder, so it's safe, but only run it from inside `~/cloud-infra-learning`.
- Floci starts containers through the Docker socket (ECS tasks, RDS, EC2). Watch them with `docker ps`, and clean up with `terraform destroy` or Floci itself, not by bulk-deleting containers.

## Next

[03: Course code](03-course-code.md), then the [implementation log](README.md#implementation-log).
