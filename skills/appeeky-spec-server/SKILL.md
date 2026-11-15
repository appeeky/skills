---
name: appeeky-spec-server
description: Set up and operate Appeeky Spec on a machine or container host the user owns, end to end over SSH, with the published ghcr.io/appeeky/appeeky-spec image. Covers choosing a host (a VM with Docker on Hetzner, DigitalOcean, AWS, Google Cloud or an office machine, or a platform scheduler such as Railway cron, Fly, Render or Cloud Run Jobs), installing Docker, docker-compose.yml and a chmod 600 .env, the first dry run, reading the logs, the healthcheck ping, updating the image, several apps on one host, moving hosts and fixing a failed routine. Use when asked to "run Spec on my server", "set it up on a VPS / Hetzner / EC2", "deploy the Docker image", "docker compose for Appeeky Spec", "it stopped pushing on the server", "update the container", or "move it to another server". For GitHub Actions and workflow files use appeeky-spec-automation; for the weekly review content use appeeky-spec-cofounder.
metadata:
  version: 1.0.0
---

# Run Appeeky Spec on your own server

The image runs the same routines as the GitHub workflows. Each run clones the repo, does one thing, commits what it wrote and exits, so the server holds nothing but `docker-compose.yml` and `.env`. Nothing listens on a port: the container only calls out to GitHub, Appeeky and the stores.

| Routine | Does |
| --- | --- |
| `daily` | `category pull`, `visibility pull`, `verify --playbook`, `reviews pull`, `competitors pull`, `social pull`, `ugc pull` (+ `seo pull`, `search-ads pull`, `seo ranks`, `visibility scan --local` when their keys are set), then commits to `BRANCH` |
| `push` | pushes listing changes merged since the `appeeky-spec/pushed` tag, sends merged review replies, social posts and UGC clips, commits the release |
| `weekly` | the co-founder review, opened as a pull request `listing/weekly-<date>` (needs the `-agent` image) |
| `schedule` | stays up and runs the three on cron (supercronic). The compose file uses this |

Images (linux/amd64 and arm64): `ghcr.io/appeeky/appeeky-spec:latest`, and `:latest-agent` with Claude Code and `gh` for `weekly`. Versioned tags exist too: `:0.5.0`, `:0.5`, `:0.5.0-agent`, `:0.5-agent`.

## 1. Choose where (the person decides)

Ask before creating anything that costs money: a server, a Railway or Fly service, a Cloud Run job. Suggest, then wait for a yes and for SSH access or platform access.

- **A VM with Docker** (Hetzner, DigitalOcean, AWS EC2/Lightsail, Google Cloud, a machine in the office): `docker compose up -d` with `command: schedule`. The smallest instance is enough; nothing is served.
- **A platform with an always-on container** (Railway, Fly, Render): run the image with `MODE=schedule` and the `.env` variables as the platform's secrets.
- **A scheduler you already have** (cron, Railway cron, Cloud Run Jobs): one job per routine, the routine as the argument or `MODE=<routine>`:

```bash
docker run --rm --env-file app.env ghcr.io/appeeky/appeeky-spec daily     # once a day
docker run --rm --env-file app.env ghcr.io/appeeky/appeeky-spec push      # every 10–15 minutes
docker run --rm --env-file app.env ghcr.io/appeeky/appeeky-spec:latest-agent weekly
```

Before choosing, make sure the repo is ready: `appeeky.spec.yaml` committed, stores connected in Appeeky (the container has no store CLIs or saved accounts, so it reaches the stores through the hosted gateways with `APPEEKY_API_KEY`; keep `via: auto` or `hosted` in the spec, not `local`), and for `weekly` the files `npx @appeeky/spec agents install` writes, committed.

## 2. Install on the VM (over SSH)

1. Docker and the compose plugin, Docker's official convenience script for Ubuntu/Debian:

```bash
curl -fsSL https://get.docker.com -o get-docker.sh && sudo sh get-docker.sh
sudo usermod -aG docker "$USER"     # log out and back in, then: docker compose version
sudo systemctl enable --now docker  # start on boot
```

2. A folder with the compose file and an empty env file:

Both files ship with this skill, in `files/` next to this SKILL.md (`docker-compose.yml` and `env.example`). Copy them over:

```bash
ssh <host> 'mkdir -p ~/appeeky-spec'
scp <skill-dir>/files/docker-compose.yml <host>:~/appeeky-spec/docker-compose.yml
scp <skill-dir>/files/env.example <host>:~/appeeky-spec/.env
ssh <host> 'chmod 600 ~/appeeky-spec/.env'
```

3. The person fills in the secrets themselves: `nano ~/appeeky-spec/.env` in their own SSH session. Never ask them to paste a secret into the chat, never write one into a command line, and never `cat` the file. To check what is set without printing values:

```bash
grep -oE '^[A-Z_]+=.' .env | cut -d= -f1    # names that have a value
```

`.env` variables (only these exist):

| Variable | What to put |
| --- | --- |
| `REPO_URL` | `https://github.com/<owner>/<repo>.git`, the repo with `appeeky.spec.yaml`. Required |
| `GITHUB_TOKEN` | fine-grained token, "Only select repositories" → this repo, Contents: Read and write; Pull requests: Read and write too for `weekly`. Passed to git through a credential helper, never in a URL or log |
| `APPEEKY_API_KEY` | appeeky.com/dashboard/settings: store access, analytics, AI visibility |
| `BRANCH` (`main`), `SPEC_DIR` (`.`) | the branch merges land on; the folder with `appeeky.spec.yaml` |
| `DAILY_AT` (`0 6 * * *`), `PUSH_EVERY` (`*/15 * * * *`, `off` to push by hand), `WEEKLY_AT` (off, e.g. `0 7 * * 1`) | cron times, UTC |
| `PUSH_DRY_RUN=1` | `push` only shows what it would change, sends nothing |
| `HEALTHCHECK_URL` | e.g. a healthchecks.io ping URL; `daily` pings it on success, `<url>/fail` on failure |
| `CLAUDE_CODE_OAUTH_TOKEN` or `ANTHROPIC_API_KEY` | `weekly` only: the person runs `claude setup-token` on their computer (Claude Pro/Max plan), or an API key billed per use |
| `APPEEKY_SPEC_IMAGE` | read by compose: `ghcr.io/appeeky/appeeky-spec:latest-agent` for `weekly`, or a pinned tag |
| `FAL_KEY` | render UGC clips on their own fal account instead of Appeeky credits |
| `GSC_CREDENTIALS` | Search Console service account JSON (or a path) → `seo pull` |
| `ASA_CLIENT_ID`, `ASA_TEAM_ID`, `ASA_KEY_ID`, `ASA_PRIVATE_KEY` | Apple Search Ads API user → `search-ads pull` |
| `DATAFORSEO_LOGIN`, `DATAFORSEO_PASSWORD`, `SEO_RANKS_EVERY` (7) | `seo ranks` every N days (paid per call) |
| `VISIBILITY_EVERY` + `OPENAI_API_KEY` / `GEMINI_API_KEY` / `PERPLEXITY_API_KEY` / `ANTHROPIC_API_KEY` | `visibility scan --local` every N days on their keys (paid per call) |
| `ATTRIBUTION_TOKEN` | a self-hosted attribution server's admin token |
| `GIT_USER_NAME`, `GIT_USER_EMAIL` | commit author (default `appeeky-spec`) |

The paid keys (`DATAFORSEO_*`, `VISIBILITY_EVERY`, `FAL_KEY`) start spending on a schedule: add them only when the person asks.

## 3. First run

1. Start with `PUSH_DRY_RUN=1` in `.env`, then `docker compose up -d` and `docker compose logs -f`. Healthy output:

```
[appeeky-spec schedule] Running on this schedule (UTC):
  0 6 * * * appeeky-spec-run daily
  */15 * * * * appeeky-spec-run push
[appeeky-spec push] Dry run: started following main at 3f2a9c1. Nothing is pushed while PUSH_DRY_RUN=1.
[appeeky-spec push] The spec against the live store today (what the next listing merge would push with it):
```

supercronic adds its own lines around each job. `Dry run: started following main …` proves the clone and the token work (it set the `appeeky-spec/pushed` tag, which only marks where to start). The `push --dry-run` output after it is how the spec differs from the live store right now.

2. Run `daily` once by hand instead of waiting for 06:00: `docker compose run --rm appeeky-spec daily`. It ends with `[appeeky-spec daily] Committed: Daily: …` or `Nothing to commit.`, and a commit appears on `BRANCH`. With `HEALTHCHECK_URL` set, the check turns green; set its period to 1 day with a few hours' grace so a missed run alerts too.
3. After the next listing merge, the dry run shows `Dry run: listing changed since <sha>.` plus the `push --dry-run` output: exactly what a real run would push. (Images before the version with this behaviour stopped at `Dry run: would start following main`; update the image if you see that line on every run.)
4. After a few days of correct dry runs, and only when the person says so, delete `PUSH_DRY_RUN` from `.env` and run `docker compose up -d --force-recreate` so the container picks up the new env.

## 4. Operate it

- **Update:** `docker compose pull && docker compose up -d`. To pin, set `APPEEKY_SPEC_IMAGE=ghcr.io/appeeky/appeeky-spec:0.5` and change it deliberately.
- **Reboots:** `restart: unless-stopped` plus Docker enabled at boot brings it back. A container stopped with `docker compose stop` stays stopped; `docker compose up -d` again.
- **Status:** `docker compose ps`, `docker compose logs --since 24h`. Logs rotate at 3 × 10 MB.
- **Several apps:** one service and one env file each (`chmod 600` every one). Never point two services or two hosts at the same repo.

```yaml
services:
  sleep-app:
    image: ghcr.io/appeeky/appeeky-spec:latest
    command: schedule
    env_file: sleep-app.env
    restart: unless-stopped
  focus-app:
    image: ghcr.io/appeeky/appeeky-spec:latest
    command: schedule
    env_file: focus-app.env
    restart: unless-stopped
```

- **Moving hosts:** all state is in the repo (the tag, releases, data), so: `docker compose down` on the old host first, set up the new one (section 2, the person re-enters secrets), `docker compose up -d`, then delete the old `.env`. If the old machine is gone for good, the person rotates `GITHUB_TOKEN` and `APPEEKY_API_KEY`.
- **Stopping:** `docker compose down`. Nothing else to clean up; the `appeeky-spec/pushed` tag stays in the repo for next time.

## 5. When a routine fails

1. `docker compose logs --since 48h | grep -E 'appeeky-spec|rror'`, or a `/fail` email from the healthcheck. Rerun the routine by hand (`docker compose run --rm appeeky-spec daily`) to see the full output; for `push`, rerun only with `PUSH_DRY_RUN=1` unless the person says otherwise.
2. Common causes:
   - Clone fails / `Authentication failed`: token expired, wrong repo selected, or missing Contents write.
   - `git push` rejected: branch protection blocks direct commits to `BRANCH`, or the token is read-only. The routines commit straight to `BRANCH`.
   - Nothing ever pushed: merges land on another branch than `BRANCH`, or the listing change is outside `appeeky.spec.yaml` / `appeeky/store` (under `SPEC_DIR`).
   - `Fix these first, or pass --force.`: `check` found errors in the merged listing. Fix in a pull request; never add `--force`.
   - `<n> change(s) wait for the store`: name, subtitle or keywords need a new App Store version; a person runs `push --create-version <x.y.z>`. The tag stays, so it retries.
   - Store errors: the store is not connected in Appeeky, `APPEEKY_API_KEY` missing, or the spec says `via: local`.
   - Daily red at `verify --playbook`: the one daily step that fails the run; run `npx @appeeky/spec verify --playbook --json` in the repo.
   - Weekly: `This image has no agent` (use `:latest-agent`), `Set CLAUDE_CODE_OAUTH_TOKEN …`, `No review prompt …` (commit `agents install` output), `Nothing proposed.` (often no store is free to change).
3. Run `npx @appeeky/spec doctor --json` locally in the repo with the same keys for anything else. Propose fixes to the repo as a pull request.

## Hand-offs

- GitHub Actions, the pull request check and its report: appeeky-spec-automation. Pull request comments still need GitHub; the server does not post them.
- What the weekly review reads and proposes: appeeky-spec-cofounder.

## Mistakes to avoid

- Creating a server, a paid plan or a platform service without the person's yes.
- Asking for secrets in the chat, putting them in a command, `cat .env`, `docker compose config` (it prints every value) or `docker inspect` output in the chat.
- Committing `.env` or leaving it world-readable; it lives only on the host, `chmod 600`.
- Turning off `PUSH_DRY_RUN` before the person has seen the dry runs and agreed.
- Running the same repo from two hosts (or the server and a second copy) at once: double commits and pushes.
- Using `:latest` with `WEEKLY_AT` set: `weekly` needs `:latest-agent`.
- Setting `via: local` for a container that has no store CLIs or accounts.
- Adding paid keys (DataForSEO, `VISIBILITY_EVERY`, `FAL_KEY`) the person did not ask for.
- Forcing a push (`--force`) or running `push --yes` by hand to "unstick" the routine: fix the cause and let the next run push.
