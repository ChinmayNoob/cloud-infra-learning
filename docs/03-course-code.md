# 03: Course code

## Goal

Work directly in `~/cloud-infra-learning`, with the course app in the repo, so the git history follows our progress through the course.

## What we did

We copied the `stage-01-start-up` branch of [ALT-F4-LLC/fem-fd-service](https://github.com/ALT-F4-LLC/fem-fd-service) (commit `385d044`) into the **repo root** without changing it. We used `git archive`, so only the files came over and none of the course's git history:

```bash
git clone https://github.com/ALT-F4-LLC/fem-fd-service.git /tmp/fem-fd-service-src
git -C /tmp/fem-fd-service-src archive stage-01-start-up | tar -x -C ~/cloud-infra-learning
```

The only file that collided was `README.md`. We kept the course README and added a two-line header on top that points to `docs/`.

## Why the repo root

The course's commands assume they run from the project root: `docker compose up`, `make …` (stage 02), `.github/workflows/` (GitHub only reads workflows from the root), and `terraform/` (stage 03). Keeping the same layout means no paths change.

## Repo layout now

```
cloud-infra-learning/
├── docs/                 ← learning log
├── main.go               ← the app (listens on :8080)
├── templates/, static/   ← HTML, CSS, JS
├── Dockerfile            ← single-stage Go build
├── docker-compose.yml    ← local Postgres (network_mode: host → :5432)
├── .env.example          ← copy to .env (git-ignored) and fill in the OAuth values
├── go.mod, go.sum
└── README.md             ← course README + pointer to docs/
```

`.env` is already in `.gitignore`, so the OAuth client secret never gets committed.

## Moving to later stages

When we reach stage 02 or 03, we bring in that stage's changes as a new commit on top of ours. To see what a stage changes:

```bash
git clone https://github.com/ALT-F4-LLC/fem-fd-service.git /tmp/fem
git -C /tmp/fem diff stage-01-start-up origin/stage-02-growth --stat
```

Then we apply the changes and add our Floci tweaks in separate commits, so it's clear what came from the course and what's ours.

## Next

[Implementation 0: Stage 01, run the app locally](implementation/implementation-0.md)
