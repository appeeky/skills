---
name: appeeky-spec-cofounder
description: Run the weekly growth review (the co-founder review) in an Appeeky Spec repo as one agent, without subagents, in Codex, Cursor, Gemini CLI, opencode or Claude Code. It reads the open alerts first, checks which stores are free to change, refreshes reviews, competitors, data, social and UGC, works through the ASO, reviews, ads, releases, UGC and social checklists, picks at most one listing change, writes the brief to appeeky/briefs/, drafts posts and a UGC brief, and prepares the pull request. Use when asked "what should we do this week?", for the weekly growth or ASO review, the co-founder review, /appeeky-cofounder, a growth brief, or "propose this week's listing change". To schedule it, see appeeky-spec-automation; to judge one past release, see appeeky-spec-review; for a single listing edit, use appeeky-spec.
metadata:
  version: 1.0.0
---

# The co-founder review with Appeeky Spec

Once a week: read everything, decide one thing, write it down, hand the rest to a person. The output is a pull request with a brief, at most one listing change, and drafts that go out only after a merge. You never push to a store, post, reply or spend credits.

Run the CLI as `npx @appeeky/spec <command> --json`, or `appeeky-spec <command> --json` when installed (the docker image has it). Scheduled routines only allow plain CLI commands: one command per shell call, no pipes, `&&` or `2>&1`; read files with your file tools. In Claude Code with `agents install` done, `/appeeky-cofounder` runs the same review with six subagents; this skill is the same review done by one agent.

If the run starts with "Today is <date>. Stores free to change: …", a scheduled routine started you: it already refreshed the data and will open the pull request after you finish. Skip step 2 and the git part of step 8.

## 1. Where things stand

1. `alerts --json` → `{ alerts }`: what needs attention now (rank, category, rating, Search Console and AI visibility drops, lost releases, competitor moves on targets, late checks), each with numbers, source file and next step. Read them first; every critical one goes into the brief under Where we stand, with its next step. Never run it with `--webhook`, `--issue` or `--record`; the daily run delivers them.
2. `status --json` → `{ loops, free }`. Only stores in `free` may get a listing change; the others have a release still being measured. It also rewrites `appeeky/loops.md`.
3. Read `appeeky/loops.md` (warnings name competitor moves inside a release's window), `appeeky/playbook.md` (what won and lost) and the newest file in `appeeky/briefs/`, if any.
4. `log --json` for recent releases and their notes.

## 2. Refresh what you read

Needs `APPEEKY_API_KEY`. Each writes only into the repo; failures are fine, note them as missing data.

```bash
npx @appeeky/spec data pull        # .appeeky/data/*.csv + README.md
npx @appeeky/spec reviews pull     # appeeky/reviews/
npx @appeeky/spec competitors pull # appeeky/competitors/, changes.md
npx @appeeky/spec social pull      # appeeky/social/
npx @appeeky/spec ugc pull         # appeeky/ugc/personas.yaml, clips.jsonl
```

Never run the listing `pull`: it overwrites `appeeky/store/`.

## 3. The six checklists

Work through each, one after another (the deep versions: appeeky-spec-targets, appeeky-spec-reviews, appeeky-spec-ads, appeeky-spec-review, appeeky-spec-ugc, appeeky-spec-social). Skip one whose data is missing and say which command fills it. Every finding carries its number, window and source file; never estimate or invent.

**ASO.** Read `appeeky.spec.yaml` (targets, storefronts, personas), `appeeky/store/**`, `coverage --json`, `suggest --json`, `check --json`, `appeeky/competitors/*.yaml` and `changes.md`, `appeeky/visibility/*.yaml`, `appeeky/seo/summary.md`, `appeeky/search-ads/summary.md`, the last 30 rows of `appeeky/category.csv`. Look for missing or `partial` targets by popularity, spare keyword space (`crossLocalization`), words repeated across name/subtitle/keyword field, competitors that climbed on your targets, AI answers naming competitors, Google queries that are not targets, the category trend. Candidates give before → after with counts (`27/30`).

**Reviews.** Read `appeeky/reviews/summary.md` (themes, ratings per version, words users use that no listing has), the latest `appeeky/reviews/*/*.jsonl`, `reviews list --rating 1-2 --unanswered --json`, `appeeky/reviews/replies/`, low ratings in `appeeky/competitors/*.yaml`. Look for recurring requests, features users look for that the listing does not mention, unanswered low reviews, pricing complaints next to the RevenueCat numbers. A listing change describes what the app does today; a missing feature is a product finding. Quote users only from the files.

**Ads.** Read `ads --json`, `appeeky/ads/*.yaml`, `.appeeky/data/README.md` and the Search Ads, Meta, TikTok and attribution files it lists, `appeeky/search-ads/keywords.csv` and `search-terms.csv`, experiments of type `ads`. Look for paid search terms that convert but are not targets (the strongest evidence for a new target), terms that spend without installs, cost per install per channel. Bid, budget and pause ideas go to a person; Spec does not change ads.

**Releases.** Read `status --json`, `log --json`, `appeeky/releases/*.yaml`, `diff --json` (changes `blocked` until a new App Store version), and `git log --oneline` / `git tag` since the last version. Look for releases decided since the last brief, late checks, a lost release worth reverting (name it; do not run `revert`), and What's New text built only from commits, in plain words.

**UGC.** Read `appeeky/ugc/personas.yaml` (`ready`), `clips.jsonl`, `appeeky/ugc/briefs/`, experiments of type `ugc`, `experiment list --json`, attribution files, the review summary. Look for which hook, persona and channel won by link installs or revenue. A link without installs is "too early". One variable per experiment.

**Social.** Read `appeeky/social/accounts.yaml` (draft only for these), `summary.md`, `posts.jsonl`, `native-posts.jsonl`, `comments.jsonl`, `insights.yaml`, `drafts/`, `social comments --unanswered --json`, What's New, the review summary. Look for something shipped or won worth saying, hooks that did best per account, questions commenters repeat, unanswered comments worth a reply (skip `canReply: false`, spam and abuse).

## 4. Decide

Pick exactly one listing change for a store in `free`, the one with the strongest evidence: an uncovered high-popularity target, words many reviewers use, a paid search that converts, a search a competitor just climbed. One hypothesis. Never repeat a change the playbook says lost unless you say what changed. No free store, or no strong candidate: change nothing and say why.

## 5. Make it

1. Edit `appeeky/store/<store>/<locale>.yaml`. Keep the brand and every word that covers a target unless you name what it costs. No other app's name. Limits and rules: see appeeky-spec.
2. `check --json`: zero errors. `diff --json`: confirm only your change. `blocked` means it needs a new App Store version: say so in the brief.

## 6. Write the brief

`appeeky/briefs/<YYYY-MM-DD>.md`:

```markdown
# Growth brief 2026-11-16
## Where we stand
<open alerts (critical first) and what each needs; what is being measured and when it is checked>
## What the specialists found
### ASO / Reviews / Ads / Releases / UGC / Social   (a few lines each, with numbers and files)
## The change
<store locale field: "before" → "after" (n/limit). Why, the metric, checked day 7 and day 14 after the push. Or why none.>
## For a person
<review replies to send, `social reply` commands to approve, ads changes, product requests, the next UGC experiment, posts>
## Not now
<good candidates not picked, and why>
```

## 7. Drafts a merge sends

Full field rules: appeeky-spec-social (posts, comments) and appeeky-spec-ugc (briefs).

Social posts, one file each, `appeeky/social/drafts/<YYYY-MM-DD>-<slug>.yaml`:

```yaml
accounts: [tiktok:@sleepysounds]       # handles from accounts.yaml; a bare platform only with one account on it
text: "Fan sounds are here. Pick a fan, set a timer, sleep."
when: "2026-11-18T16:00:00Z"            # ISO, at least a day ahead
media: ["https://…/fan.mp4"]            # Instagram, TikTok, Pinterest need one; YouTube one video
tiktok: { disclosure: own_brand, inbox_draft: true }   # inbox_draft when the media is AI-generated
why: "Users asked for fan sounds in 9 reviews; it shipped in 3.2."
```

YouTube posts add `youtube: { title, visibility, synthetic_media: true }` for AI video; Reddit needs `reddit: { subreddit, title }`. Run `social check` and fix every problem, or drop the post.

The next UGC video, only for a persona with `ready: true`: `appeeky/ugc/briefs/<YYYY-MM-DD>-<slug>.yaml` with `persona`, `script` (spoken, under 25 words for 8s), `scene`, `caption`, `durationSec`, `aspectRatio` (`9:16` or `16:9`), `why`. Run `ugc generate --dry-run` and put the credit cost in the brief. No ready persona: say a person picks a look in Appeeky first.

Review replies: put the reply text in the brief. Drafting them with `reviews reply <id> --text "…"` writes `appeeky/reviews/replies/`; do it only when the user asked.

Comment replies: write the exact `appeeky-spec social reply <comment id> "<reply>"` commands in the brief, within the platform limit (X 280, Bluesky 300, TikTok 150, Threads 500). The owner runs them.

## 8. The pull request

1. Write `.appeeky/proposal.md`: first line a short PR title; then what changes, why with the numbers, the metric and its day-7/day-14 checks, and a link to the brief.
2. Only when nobody else opens the PR and the user asked for one: a new branch `listing/weekly-<date>`, add only `appeeky/store appeeky.spec.yaml appeeky/briefs appeeky/social/drafts appeeky/ugc/briefs` (plus `appeeky/reviews/replies` if you drafted any), commit with the title, push the branch, `gh pr create --title "<title>" --body-file .appeeky/proposal.md`. Never push to `main`, never merge.
3. Optionally `report --base main` for the listing, check and coverage summary in the PR body.

After the merge, the push workflow (or the docker push routine) runs `push`, `reviews push`, `social push` and `ugc generate`.

## Never run

`push`, `revert`, `pull`, `reviews push`, `social push`, `social reply`, `social hide`, `social like`, `social delete-comment`, `ugc generate` without `--dry-run`, `visibility scan --local --yes`, `seo ranks|keywords|domain … --yes`, `alerts --webhook|--issue|--record`, `experiment add` or `experiment close`. Never put keys or tokens in any file.

## Mistakes to avoid

- Two listing changes, or a change on a store not in `free`: the results mix and nothing is learned.
- A number or quote with no source file, or a guess for a channel with no data.
- A listing line promising a feature users ask for but the app lacks.
- Posts for accounts not in `accounts.yaml`, or a `when` in the past.
- Dropping a word that covers a target without saying so.
- Running `pull` "to refresh" and wiping uncommitted listing edits.
