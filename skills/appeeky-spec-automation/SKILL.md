---
name: appeeky-spec-automation
description: Run Appeeky Spec unattended and fix it when a run fails. Covers `init --ci` and its three GitHub workflows (pull request check, push on merge, daily refresh), the repository secrets they need, the pull request report, the daily alerts (one GitHub issue, a Slack webhook), cron or other CI, and scheduling the weekly co-founder review. Use when asked to set up CI or GitHub Actions for the listing, "push on merge", a daily job, alerts or a notification when ranks, ratings or the category drop, run Spec with cron, GitLab or other CI, schedule the weekly review, which secrets to add, or why a listing workflow, push or daily run failed. For the review itself use appeeky-spec-cofounder; for a VM, Docker or Railway, Fly and Render use appeeky-spec-server; for first-time spec setup use appeeky-spec.
metadata:
  version: 1.0.0
---

# Running Appeeky Spec unattended

Three jobs keep a spec repo alive: a check on every pull request, a push after a merge, and a daily refresh that measures releases. A fourth, optional, runs the weekly co-founder review. The merge is the gate everywhere: nothing reaches a store, a social account or the UGC credits unless a person merged it.

Run commands as `npx @appeeky/spec <command>`. Start every setup or diagnosis with `doctor --json`: it lists store access, targets, `APPEEKY_API_KEY`, attribution, workflows, skills and `.gitignore`, each with its fix; exit code 1 means something must be fixed.

## 1. Pick where it runs

| Where | Use |
| --- | --- |
| GitHub Actions | `init --ci`; the default |
| A server (VM, Railway, Fly, Render) or a scheduler you already have (Cloud Run Jobs, Railway cron) | the Docker image; see appeeky-spec-server |
| Other CI (GitLab, Bitbucket, CircleCI) | the same three triggers and commands as the workflows below |
| The user's machine | `github.com/appeeky/spec/examples/weekly/weekly.sh` with cron or launchd (weekly review) |

## 2. GitHub Actions

1. `npx @appeeky/spec init --ci` writes into `.github/workflows/` (it skips a file that already exists; delete it first to get the current version):

| File | Trigger | Runs |
| --- | --- | --- |
| `listing-check.yml` | `pull_request` | the `appeeky/spec@v0` action: `report --base origin/<base>`, comments it, fails on check errors |
| `listing-push.yml` | push to `main` touching `appeeky/store/**`, `appeeky.spec.yaml`, `appeeky/reviews/replies/**`, `appeeky/social/drafts/**`, `appeeky/ugc/briefs/**` | `push --yes --note "<merge commit message>"`, `reviews push --yes`, `social push --yes`, `ugc generate --yes` (`--local` with `FAL_KEY`), then commits the release and sent markers |
| `listing-daily.yml` | `0 6 * * *` UTC and manual | `category pull`, `visibility pull`, `verify --playbook`, `reviews pull`, `competitors pull`, `social pull`, `ugc pull`; with keys also `seo pull`, `search-ads pull`, `seo ranks --every`, `visibility scan --local --every`; then `alerts --record`, commits, and keeps the alert issue (`issues: write`) |

2. Secrets: tell the user which to add under Settings → Secrets and variables → Actions. Never ask them to paste one into chat, and never write one into a file.

| Name | Kind | Needed for |
| --- | --- | --- |
| `APPEEKY_API_KEY` | secret | hosted store access, reviews, competitors, social, UGC, data, visibility |
| `FAL_KEY` | secret | optional: UGC clips on the user's fal account |
| `GSC_CREDENTIALS` | secret | `seo pull` (service account JSON) |
| `ASA_CLIENT_ID`, `ASA_TEAM_ID`, `ASA_KEY_ID`, `ASA_PRIVATE_KEY` | secrets | `search-ads pull` on the user's own key |
| `DATAFORSEO_LOGIN`, `DATAFORSEO_PASSWORD` | secrets | `seo ranks`; `SEO_RANKS_EVERY` variable, default 7 |
| `VISIBILITY_EVERY` | variable | days between `visibility scan --local`, with `OPENAI_API_KEY`, `ANTHROPIC_API_KEY`, `GEMINI_API_KEY`, `PERPLEXITY_API_KEY` secrets |

The push workflow reads `APPEEKY_API_KEY`, or your own store keys from secrets of these names when there is no Appeeky key: `ASC_KEY_ID`, `ASC_ISSUER_ID`, `ASC_PRIVATE_KEY` (the .p8 contents), `GPS_CREDENTIALS` (the service account JSON). A `listing-push.yml` written by an older version lacks them: delete it and run `init --ci` again (it skips files that exist), or add them to that step's `env:`.

3. `listing-push.yml` runs in the `stores` environment: suggest adding required reviewers to it for a second approval.
4. Commit the workflows on a branch and open a pull request; the user merges.

### The pull request report

`listing-check.yml` (or any workflow using `appeeky/spec@v0` after `actions/checkout@v4` with `fetch-depth: 0`) posts one comment, edited on each run: listing changes, check results, coverage before → after, and the replies, posts and clips the merge sends out. Action inputs: `working-directory` (folder with the spec), `base`, `comment` (`"false"` to skip), `version`, `github-token`. The same Markdown locally: `report --base main` (`-o <file>` to save it). It needs `pull-requests: write`.

## 3. Alerts

`alerts` turns the files the daily run wrote into what needs a person: `rank-drop` (from the ranks in `appeeky/releases/`), `release-lost`, `category-drop`, `low-ratings`, `competitor-change` (on targets only), `visibility-drop`, `search-drop` (`appeeky/seo/daily.csv`) and `late-check`. Each has `severity` (`warn`/`critical`), `numbers`, `window`, `source` and `next`. It makes no network call except the delivery flags.

- Read them: `alerts --json` (`--new` for the ones not reported yet). `--markdown -o <file>` writes an issue body; the file is empty when nothing is open.
- Dedupe: `--record` writes `appeeky/alerts.json`; the daily run commits it, so an alert is new once, again when it turns critical, and again only after it cleared and came back. Never hand-edit it; delete it to report everything again.
- Thresholds: `alerts:` in `appeeky.spec.yaml` (`rank-drop.places`, `category-drop.places`/`days`, `low-ratings.days`/`min`/`ratio`, `search-drop.days`/`drop`/`min`, `competitor-change.days`, `visibility-drop.days`, `release-lost.days`, `late-check.days`), `off: [<kind>]` to turn kinds off. Too noisy: raise a threshold or turn the kind off in a pull request, not by deleting the state file.
- GitHub: `listing-daily.yml` runs `alerts --new -o .appeeky/alerts-new.md` and `alerts --record -o .appeeky/alerts.md`, then with `gh` keeps one open issue labelled `appeeky-spec-alert` (body: every open alert; a comment with the new ones; closed when the file is empty). A `listing-daily.yml` from an older version has neither the steps nor `issues: write`: delete it and run `init --ci` again (doctor's `alerts` line says so).
- Docker `daily`: `alerts --record`, plus `--webhook` when `ALERT_WEBHOOK_URL` is set (Slack-compatible `{"text"}`, new alerts only) and `--issue` when `GITHUB_TOKEN` is set (same issue through the GitHub API; needs Issues: Read and write; `ALERT_ISSUE=0` skips it). A failed post is not recorded and is retried the next day. `! GitHub answered 403 … Issues: Read and write` in the log: add that permission to the token or set `ALERT_ISSUE=0`.
- Other CI or cron: `alerts --record --webhook` (and `--issue` with `GITHUB_TOKEN`) after the daily commands, then commit `appeeky/alerts.json`.
- The weekly agent may run `alerts --json` but never `--webhook` or `--issue`; the routines deny them.

## 4. The weekly co-founder review

1. `npx @appeeky/spec agents install` (`--for opencode` or `all`) writes `.claude/agents/appeeky-*.md` and `.claude/commands/appeeky-cofounder.md`; commit them. Or copy `github.com/appeeky/spec/examples/weekly/propose.md` to `.github/appeeky-propose.md` to keep an own version; the routines prefer it.
2. Pick the runner:
   - GitHub: copy `github.com/appeeky/spec/examples/weekly/appeeky-spec-weekly.yml` to `.github/workflows/` (Mondays `0 7 * * 1`, and manual).
   - The user's machine: `github.com/appeeky/spec/examples/weekly/weekly.sh`, e.g. crontab `0 7 * * 1  cd /path/to/app-repo && sh weekly.sh >> .appeeky/weekly.log 2>&1`. Needs git with push access, `gh` logged in, Node 20+, `claude`.
   - Docker: `WEEKLY_AT=0 7 * * 1` with `APPEEKY_SPEC_IMAGE=ghcr.io/appeeky/appeeky-spec:latest-agent`.
3. Agent credentials, one of: `CLAUDE_CODE_OAUTH_TOKEN` (the user runs `claude setup-token` once; billed to their Claude Pro or Max plan) or `ANTHROPIC_API_KEY` (per use). Optional: `PROPOSAL_TOKEN` (a fine-grained or GitHub App token with contents and pull requests write), because a PR opened with the default `GITHUB_TOKEN` does not trigger `listing-check.yml`; `ATTRIBUTION_TOKEN` for a self-hosted attribution server.
4. Each routine refreshes `data`, `reviews`, `competitors`, `social`, `ugc`, reads `status --json` for the free stores, runs the review with an allow-list (`appeeky-spec *`, `git log`, `git tag`, file tools) and a deny-list (`push`, `revert`, `reviews push`, `social push|reply|hide|like|delete-comment`, `ugc generate --yes`, paid `--yes` scans), then opens `listing/weekly-<date>` with `.appeeky/proposal.md` as the body. Never loosen those lists.

## 5. On a server

The same routines run from the Docker image on a VM or container host (daily, push, weekly, schedule; `.env`, `PUSH_DRY_RUN`, `HEALTHCHECK_URL`). Setting it up, operating it and its failures: see appeeky-spec-server. The failure messages below are the same there.

## 6. Diagnose a failed run

1. Read the log: `gh run view <id> --log-failed`, `docker compose logs`, or `.appeeky/weekly.log`. Run `doctor --json` with the same env.
2. Match the message:
   - `Fix these first, or pass --force.`: `check` found errors in the merged listing. Fix them in a new pull request; never add `--force`.
   - `Not a terminal: pass --yes to push.`: a custom script calls `push` without `--yes`.
   - Changes listed but nothing pushed, `blocked`: name, subtitle or keywords need a new App Store version. A person runs `push --create-version <x.y.z>`.
   - Daily red at `verify --playbook`: the only daily step without `|| true`; rerun it locally with `--json`.
   - Nothing committed: the commit step ends in `|| true`; look for a rejected `git push` and `permissions: contents: write`.
   - `The agent proposed nothing.` / `Nothing proposed.`: no `.appeeky/proposal.md` or no change under the PR paths; often no store is `free`.
   - `This image has no agent`, `Set CLAUDE_CODE_OAUTH_TOKEN`, `No review prompt: run appeeky-spec agents install`: image, secret or committed `.claude/` missing.
   - Weekly PR without a check comment: set `PROPOSAL_TOKEN`.
   - `Started at <sha>` / `Up to date.`: normal first and idle push runs.
3. Propose the fix as a pull request. Rerunning a push job is the user's call.

## Mistakes to avoid

- Running `push --yes` from a scheduled job on anything but merged `main`: it removes the gate.
- Committing keys, `.env` or `.appeeky/` (data and cache stay gitignored).
- Skipping `fetch-depth: 0`: the report has no base to compare with.
- Turning off `PUSH_DRY_RUN` before the user saw a dry run.
- Adding `push`, `social push` or `reviews push` to the weekly agent's allowed tools.
