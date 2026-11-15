---
name: appeeky-spec-ugc
description: Plan, brief, generate and measure UGC creator videos with Appeeky creator personas in an Appeeky Spec repo (appeeky/ugc/). Use when asked what to film next, to write a UGC or creator video brief or script, which hooks or personas worked, to make a talking-head clip from a persona, what a clip costs, to render clips on their own fal key (FAL_KEY), to add a persona to the repo, or to track a creator video with a smart link and close the UGC experiment. Clips are made only after a person merges the brief. For posting the finished clip see appeeky-spec-social; for screenshot, ad or pricing experiments see appeeky-spec-experiments.
metadata:
  version: 1.0.0
---

# UGC videos with Appeeky Spec

A video is a brief file. You write it in `appeeky/ugc/briefs/`, a person reviews it in a pull request, and the clip is made after the merge with a creator persona: in Appeeky with media credits, or on the user's own fal account with `--local`. Each video is measured as a UGC experiment with its own smart link.

Run commands as `npx @appeeky/spec <command> --json`.

Two kinds of "persona", do not mix them:
- **Creator personas** (`appeeky/ugc/personas.yaml`, or `appeeky/ugc/personas/<name>.yaml`): who appears on screen. A brief's `persona:` names one.
- **Audience personas** (`personas:` in `appeeky.spec.yaml`, with `id`, `name`, `searches`): who the video is for. `experiment add --persona <id>` names one.

## 1. Pull and read

`ugc pull --json` (needs `APPEEKY_API_KEY`; `--limit <n>` clips, default 100) writes:

| File | Holds |
| --- | --- |
| `appeeky/ugc/personas.yaml` | creator personas made in Appeeky: `id`, `name`, `bio`, `lane`, `platforms`, `contentPillars`, `ready` (has a look, so it can make clips) |
| `appeeky/ugc/clips.jsonl` | clips made: `persona`, `status`, `script`, `caption`, `durationSec`, `aspectRatio`, `scheduledFor`, `socialPostId`, `error` (no media URLs: they expire) |

Then read: `appeeky/ugc/briefs/` (do not repeat a waiting brief), `experiment list --json` and `appeeky/experiments/*.yaml` of type `ugc` (hook, persona, channel, link, metric, result), `appeeky/playbook.md`, `appeeky/reviews/summary.md` (users' own words make strong first lines), `.appeeky/data/README.md` then `smart-links.csv` and `attribution-sources.csv` (after `data pull`), and `appeeky/social/posts.jsonl` for how posted clips did.

## 2. Pick the next video

- Start from evidence: the hook, persona and channel that won (by link installs or revenue, not views), a winning hook not tried on another channel, a persona with no video yet, or a review phrase.
- One variable per experiment: change the hook, the persona or the channel, never all three.
- A link without installs yet is "too early", not "lost". Quote results with their window and file.
- Use a creator persona with `ready: true`, or one kept in the repo with `looks:`. If none is ready, stop: a person picks a look (reference images) in Appeeky UGC, or adds a repo persona with images.

## 3. Write the brief

`appeeky/ugc/briefs/<date>-<slug>.yaml`. Strict schema; only these keys:

```yaml
persona: Mia                       # name or id from personas.yaml, or a repo persona's file name
script: "I fell asleep in five minutes with the rain mix. I'm not even kidding."
scene: "Bedroom at night, lamp on, phone in hand showing the sound mixer"
caption: "the rain mix is unreal #sleep"
title: "Rain mix hook"            # optional
durationSec: 8                     # optional, default 8 (fitted to the video model)
aspectRatio: "9:16"                # "9:16" (default) or "16:9"
why: "Hook from 6 reviews (appeeky/reviews/summary.md); test vs demo hook; link_installs, 14 days"
```

- `script` is what the persona says out loud, word for word: short and spoken, under about 25 words for 8 seconds.
- Only what the app does today; no claims the listing does not make, no invented results.
- `why` carries the hypothesis for the reviewer: which hook, for whom, measured how.
- Never set `status`, `clipId`, `credits`, `generatedAt`, `clipFile`, `videoUrl` or `costUsd`: `generate` writes them.

A persona kept in the repo, for `--local` (`looks` are image paths next to the file, or URLs; required):

```yaml
# appeeky/ugc/personas/mia.yaml
name: Mia
appearance: early 20s, dyed pink bob, oversized hoodie
personality: playful, talks fast
lane: dance trends and sound edits
looks: [mia/front.png, mia/side.png]
```

## 4. Cost first, then the pull request

Run the dry run and put the cost in the pull request:

- Appeeky credits: `ugc generate --dry-run --json` → `{ dryRun, briefs: [{ file, persona, credits, available, affordable }], credits }`. Fails on an unknown persona (`ugc pull` first) or a persona without a look.
- Own fal key: `ugc generate --local --dry-run --json` → per brief `durationSec`, `aspectRatio`, `videoModelId`, `costUsd`, and total `costUsd` (USD on the user's fal account). Works without `FAL_KEY`; the persona comes from `appeeky/ugc/personas/<name>.yaml`, else from Appeeky with `APPEEKY_API_KEY` (its cover image as the look).

Open a pull request with the brief; its check lists the clips the merge makes. On merge, the push workflow from `init --ci` (or the docker `push` routine) runs `ugc generate --yes`, adding `--local` when the `FAL_KEY` secret is set. Briefs are then marked so they are never made twice.

Never run `ugc generate` without `--dry-run`, or with `--yes`, unless the user explicitly asks in this conversation: it spends media credits or bills their fal account. `FAL_KEY` goes in the shell env or a CI secret, never in the repo or chat.

What generating leaves behind:
- Appeeky: the brief gets `status: generating`, `clipId`, `credits`. The clip renders in Appeeky; the next `ugc pull` records it in `clips.jsonl`. The repo gets no media URL, so a person posts it from Appeeky or supplies a public URL.
- `--local`: the brief gets `status: generated`, `clipFile` (`.appeeky/media/<brief>.mp4`, gitignored; CI keeps it as the `ugc-clips` run artifact), `videoUrl` and `costUsd`.

## 5. Post and measure

1. Before the clip goes out, propose the experiment and run it when the user agrees (it creates a smart link; needs `attribution:` in the spec):
   ```bash
   npx @appeeky/spec experiment add --type ugc --persona <audience id> \
     --hypothesis "Review-phrase hook gets more installs than the demo hook" \
     --metric link_installs --days 14 --link tiktok --asset appeeky/ugc/briefs/2026-11-20-rain-hook.yaml
   ```
   One experiment and link per video or creator. Channels: `tiktok`, `instagram`, `creator`, `tiktok_ads`, `meta_ads`, and others. Link installs count only on Google Play; for an App Store-only app use `--metric link_clicks`. Without attribution, say the user gives the value at the end.
2. Post it through a social draft with the clip's `videoUrl` in `media:` and the smart link in the text, with `tiktok: { disclosure: own_brand, inbox_draft: true }` and `youtube: { synthetic_media: true }` because the clip is AI-generated. Follow appeeky-spec-social.
3. When due, `verify --json` closes link-metric experiments (`closedExperiments`) and lists the rest in `dueExperiments`. For those: `experiment close <id> --value <n> --note "<what happened, window, source file>" --playbook`. A lost hook is a result: write it so it is not tried again. For the weekly review of releases and experiments, hand over to appeeky-spec-review.

## Mistakes to avoid

- A brief for a persona that is not `ready`, or a name that is not in `personas.yaml` or `appeeky/ugc/personas/`.
- Putting an audience persona id in a brief, or a creator persona name in `experiment add --persona`.
- A script that reads like ad copy or is too long to say in the clip's length.
- Two variables in one video; two videos on one smart link.
- Skipping the dry run, or generating before the merge.
- Calling a hook a winner on views or clicks when installs were the metric.
- Committing `.appeeky/media/` clips or a `FAL_KEY`.
