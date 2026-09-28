# Implementation 0: Stage 01, run the app locally

## Goal

Run the `goals` app on the server with a real Postgres database, and log in with Google from the laptop. This is the **local** half of stage 01. The AWS/Floci half comes later.

## Status (2026-09-28)

| # | Step | Status |
|---|------|--------|
| 1 | Google OAuth client created (redirect URI `http://localhost:8080/auth/google/callback`) | ✅ Done |
| 2 | `.env` filled in on the server (git-ignored) | ✅ Done and checked |
| 3 | Postgres running (`docker compose up -d`) | ✅ Running, Postgres 17.11 on `:5432` |
| 4 | Create the database tables (SQL from the README) | ✅ 6 tables |
| 5 | Build the app's Docker image | ✅ `goals:stage-01` |
| 6 | Run the app container on host port **8090** | ✅ Running, connected to the DB |
| 7 | SSH tunnel from the laptop, then log in at `http://localhost:8080` | ✅ Logged in, profile saved |
| 8 | Make yourself admin (`INSERT INTO administrators …`) | ✅ Done |

**The local half of stage 01 is complete.**

## What we did

### `.env`

It holds four values: `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET`, `GOOGLE_REDIRECT_URL` and `POSTGRES_URL`. The lines have no `export` prefix, which is the format Docker's `--env-file` expects.

If you ever load the file in a shell, run `set -a; source .env; set +a`, so the variables are passed on to programs.

### Postgres

```bash
docker compose up -d
```

**Issue:** the course's `docker-compose.yml` uses `image: postgres:alpine`, which now pulls **Postgres 18**. Postgres 18 refuses to start with the old mount at `/var/lib/postgresql/data`:

```
there appears to be PostgreSQL data in:
  /var/lib/postgresql/data (unused mount/volume)
...
The suggested container configuration for 18+ is to place a single mount
at /var/lib/postgresql ...
```

**Fix:** pin `postgres:17-alpine`. The course made the same fix later on its `main` branch, and 17 matches the RDS version used in stage 03.

```diff
-    image: postgres:alpine
+    image: postgres:17-alpine
```

```bash
docker compose down -v      # remove the broken container and its empty volume (this project only)
docker compose up -d
docker compose exec postgres pg_isready -U postgres
# /var/run/postgresql:5432 - accepting connections
```

### 4. Created the tables

We took the first `sql` block from the course README (six `CREATE TABLE` statements) and ran it with `psql` inside the Postgres container:

```bash
awk '/^```sql/{f++; next} /^```/{if(f==1) exit} f==1' README.md > /tmp/goals-schema.sql
docker compose exec -T postgres psql -v ON_ERROR_STOP=1 -U postgres -d postgres < /tmp/goals-schema.sql
docker compose exec -T postgres psql -U postgres -d postgres -c '\dt'
```

The result is 6 tables: `administrators`, `aspiration_updates`, `comments`, `followers`, `likes` and `users`.

Stage 02 replaces this manual step with goose migrations.

### 5. Built the image

```bash
docker build -t goals:stage-01 .
```

The image is **553 MB**, because stage 01's Dockerfile is single-stage: the whole Go toolchain ships with the app. Stage 02 switches to a multi-stage build to shrink it.

### 6. Ran the app in Docker (not `go run`)

The app hardcodes port `:8080`, and the server's 8080 belongs to another service, which we don't touch. So the app runs in a container, mapped to a free port:

```bash
docker run -d --name goals --restart unless-stopped \
  --env-file .env \
  -e "POSTGRES_URL=postgresql://postgres:password@host.docker.internal:5432/postgres?sslmode=disable" \
  --add-host host.docker.internal:host-gateway \
  -p 127.0.0.1:8090:8080 \
  goals:stage-01
```

| Flag | Why |
|------|-----|
| `--env-file .env` | Loads the Google values. This needs lines with no `export` prefix. |
| `-e POSTGRES_URL=…host.docker.internal…` | Inside the container, `localhost` means the container itself. This points it at the server's Postgres instead. |
| `--add-host host.docker.internal:host-gateway` | On Linux, this name isn't set up by default. The flag maps it to the server. |
| `-p 127.0.0.1:8090:8080` | Server port 8090 → container port 8080, published only on the server itself (reached through the tunnel). |
| `--restart unless-stopped` | Comes back up after a server reboot. |

Result:

```
$ docker logs goals
Successfully connected to the database
Server starting on http://localhost:8080

$ curl -s -o /dev/null -w '%{http_code}' http://127.0.0.1:8090/
200
$ curl -s -o /dev/null -w '%{http_code}' http://127.0.0.1:8090/auth/google/login
307   → accounts.google.com, redirect_uri=http://localhost:8080/auth/google/callback
```

### 7. Log in from the laptop

```bash
ssh -L 8080:localhost:8090 coz@coz
# open http://localhost:8080 → "Login with Google"
```

```
laptop browser → localhost:8080 ──tunnel──▶ server :8090 → container :8080 → app
```

Result: Google login worked on the first try. The first login creates a row in `users`, and the profile form fills in the rest:

```
 id | username | email          | is_banned
----+----------+----------------+-----------
  1 | chinmay  | <our gmail>    | f
```

### 8. Made ourselves admin

```bash
docker compose exec -T postgres psql -U postgres -d postgres -v email='<your gmail>' <<'SQL'
INSERT INTO administrators (email, username)
SELECT email, username FROM users WHERE email = :'email'
ON CONFLICT DO NOTHING;
SQL
```

`administrators` now has one row, which gives our account the admin (ban) features.

### Handy checks

```bash
docker logs goals --tail 20                                   # app log
docker compose exec postgres psql -U postgres -d postgres     # SQL shell
docker restart goals                                          # after changing .env
```

To pick up code changes, rebuild and re-run: `docker build -t goals:stage-01 .`, then `docker rm -f goals`, then the `docker run` command from step 6.

## Next

Implementation 1: install the AWS CLI, goose and Terraform, then start Floci.
