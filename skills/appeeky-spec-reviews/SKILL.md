---
name: appeeky-spec-reviews
description: Read and answer App Store and Google Play customer reviews in an Appeeky Spec repo (appeeky/reviews/). Use when asked what users are saying, to pull reviews, summarize complaints or feature requests, find low ratings without a reply, draft or send review replies, check whether the rating changed after a version, or turn what reviews ask for into a listing change or a product request. Not for choosing target keywords (appeeky-spec-targets), judging a pushed release (appeeky-spec-review) or competitors' reviews and moves (appeeky-spec-competitors).
metadata:
  version: 1.0.0
---

# Customer reviews with Appeeky Spec

Reviews live in the repo next to the listing. Replies are drafted as files, reviewed in a pull request, and sent after the merge, like any listing change.

Run commands as `npx @appeeky/spec <command> --json`. Every `reviews` command that talks to the stores needs `APPEEKY_API_KEY` with the stores connected in Appeeky; `reviews list` and `reviews reply` only read the repo.

| Path | What it holds |
| --- | --- |
| `appeeky/reviews/<store>/<YYYY-MM>.jsonl` | one review per line, newest first (`app-store`, `google-play`) |
| `appeeky/reviews/summary.md` | written by `reviews pull`; never edit by hand |
| `appeeky/reviews/replies/<store>-<id>.yaml` | one reply: `draft` until sent, then `sent` with `sentAt` |

## 1. Pull

```bash
npx @appeeky/spec reviews pull --json                     # everything the connection has, up to 2000 per store
npx @appeeky/spec reviews pull --since 2026-10-01 --json  # only newer reviews (YYYY-MM-DD)
```

Output: `{ "results": { "app-store": { "fetched", "added", "updated" } | { "error" }, ... }, "summary": "<path>" }`. A store with `error` was skipped; say so instead of reporting zero reviews. Pulling again adds new reviews and replaces edited or answered ones. The daily workflow (`listing-daily.yml`) already runs it; if the files are recent, read them instead of pulling.

One line of a monthly file:

```json
{"id":"1834","store":"app-store","rating":2,"title":"Timer stops early","body":"The sleep timer cuts off after 20 minutes…","author":"…","where":"US","version":"3.1","date":"2026-11-02T…","reply":{"body":"…","date":"…"}}
```

`where` is the App Store territory or the Google Play language. `reply` is missing when nobody answered. On Google Play `date` is the last-modified date, so an edited review moves to a newer month.

## 2. Read the summary first

`appeeky/reviews/summary.md` has, per store:

- last 30 days vs the 30 days before vs all pulled: count, average, share of 1–2 stars, unanswered;
- star counts, and the five most recent versions with their average and share of 1–2 stars;
- up to five 1–2 star reviews without a reply, with their ids;
- **Words users use that your listing does not**: words and two-word phrases in 3+ reviews (both stores) that appear in no listing file, with how many came from 1–3 star reviews. Stop words are English only, so other languages show more noise.

Then read the raw reviews behind anything you report:

```bash
npx @appeeky/spec reviews list --rating 1-2 --unanswered --json
npx @appeeky/spec reviews list --store google-play --rating 5 -n 50 --json
```

`--rating` takes one number or a range (`1-2`); `-n` defaults to 20. Output: `{ "reviews": [ ... ] }`, newest first. For counts over a longer period, read the monthly `.jsonl` files directly.

## 3. Find the themes

1. Group reviews by what they are about (a bug, a missing feature, pricing, a feature people love), not by single words. Count reviews per theme and note their stars and versions.
2. Check whether a theme started or stopped with a version: the version table in the summary, then `version` on the reviews.
3. Separate three kinds:
   - **Bugs and broken promises** (crash, timer stops, sync lost): product, and a reply.
   - **Requests the app already answers** (users ask for something it does): a listing gap, the feature is not visible.
   - **Requests the app cannot answer yet**: a product request. Never a listing change.
4. Every number you report names its file: "sleep timer: 7 reviews, 6 of them 1–3 stars (summary.md)". Quote users only from the files, with the review id. Never invent ids, ratings or quotes.

## 4. Decide which reviews to answer

Answer first:

1. 1–2 stars without a reply, newest first, especially ones about a bug that a version since then fixed.
2. Reviews that ask a question or report something you can act on.
3. Detailed 3-star reviews; a reply can change a rating.

Skip reviews with no content ("bad", one emoji), reviews already answered (`reply` present) unless the answer is now wrong, and ones with a draft waiting in `appeeky/reviews/replies/`.

## 5. Draft replies

```bash
npx @appeeky/spec reviews reply 1834 --text "Thanks for reporting this. 3.2 fixes the timer stopping after 20 minutes; please update and tell us if it still happens." --json
```

It writes `appeeky/reviews/replies/app-store-1834.yaml` and returns `{ "file", "issues" }`. Drafting again for the same id overwrites that file.

```yaml
id: "1834"
store: app-store
review: "2★ 2026-11-02: Timer stops early. The sleep timer cuts off…"
text: Thanks for reporting this. 3.2 fixes…
status: draft
```

Rules:

- Limits: Google Play 350 characters, App Store 5970. Keep App Store replies short too; `issues` lists a reply that is too long, and `reviews push` refuses to send any batch that has one.
- Specific to what this user said. No template that ignores the review.
- Only facts that are true today: never say a fix shipped unless that app version is out (check the app's own git history or ask), never promise dates or features.
- Write in the reviewer's language (`where` hints at it).
- No personal data, no requests to change the rating, no arguing.

## 6. Ship replies (a person decides)

1. Commit the drafts on a branch and open a pull request. `report --base main` lists the replies a merge sends out, with a warning for any that are too long.
2. When the pull request merges, the `listing-push.yml` workflow (or the Docker push routine) runs `reviews push --yes` and commits the files marked `sent`.
3. Without CI, the person runs `npx @appeeky/spec reviews push` after the merge; it shows each reply and asks before sending. Only drafts are sent.

Never run `reviews push` yourself, and never with `--yes`, unless the person explicitly told you to send. Sent replies are public. Google Play replies need `stores.googlePlay.packageName` in the spec.

## 7. Turn reviews into changes

- **A listing change**: a request the app already answers, or words users use for the app. Propose one change in `appeeky/store/**` that names the feature (a description line, a promotional text, a Google Play short description), say which theme and how many reviews it answers, and follow the appeeky-spec workflow: `status --json` first (only `free` stores may change), then `check` and `diff`. Do not mix it with another change in the same release.
- **Keywords**: phrases from "Words users use that your listing does not" are candidates, not targets. Hand them to the appeeky-spec-targets workflow to check popularity and relevance before they go in `targets`.
- **Screenshots**: a loved feature missing from the screenshots is an experiment (appeeky-spec-experiments).
- **Product requests**: list them for the person with counts and example ids, in the brief (`appeeky/briefs/<date>.md`) during the co-founder review, or in your answer. Spec does not file them anywhere.
- **Measure**: `verify` compares `rating` (average) and `low_ratings` (share of 1–2 stars) before and after each push, from the pulled files, when the window has 5+ reviews. An experiment can use `--metric rating` or `--metric low_ratings`. Judging those results is the appeeky-spec-review workflow.

## Mistakes to avoid

- Editing `summary.md` or the `.jsonl` files by hand; `reviews pull` rewrites them.
- Reporting a theme from one or two reviews as a trend, or counting the same review twice across stores.
- Treating the "words users use" table as targets without checking popularity.
- Writing a listing line for a feature the app does not have.
- Replying "fixed in the next update" when nothing shipped, or using one canned reply for every review.
- Sending replies (`reviews push`) before a person merged them.
- Quoting a review you did not read in the files.
