---
name: appeeky-spec-experiments
description: Run and measure any growth experiment outside the store listing in an Appeeky Spec repo (appeeky/experiments/), with one hypothesis, one metric, a baseline and a result. Use when asked to test a price, set up or close an experiment, pick the metric for a test, or ask which experiments are open or due. For the work inside each kind of test see appeeky-spec-screenshots, appeeky-spec-ugc, appeeky-spec-social and appeeky-spec-ads; for attribution, smart links and the app's numbers see appeeky-spec-attribution.
metadata:
  version: 1.1.0
---

# Experiments with Appeeky Spec

A listing push is already an experiment: `verify` measures it. Everything else you try to grow the app, like a new screenshot set, a creator video, an ad or a price, is tracked the same way in `appeeky/experiments/<id>.yaml`. Each one has one hypothesis, one metric, a baseline, a date to check and a result.

Run commands as `npx @appeeky/spec <command> --json`.

## 1. Know what can be measured

`experiment add --metric` takes one metric. Prefer one the spec can measure on its own. It then takes the baseline from the last days and `verify` closes the experiment when it is due:

| Needs | Metrics |
| --- | --- |
| `APPEEKY_API_KEY` with the store connected in Appeeky | `conversion`, `page_views`, `downloads`, `proceeds`, `trials` |
| RevenueCat connected in Appeeky (app-scoped project) | `revenue`, `rc_trials`, `trial_conversion` |
| Apple Search Ads connected in Appeeky, or `search-ads pull` | `ad_installs`, `cpa` |
| Meta / TikTok campaigns under `ads:` | `meta_spend`, `meta_installs`, `meta_cpi`, `tiktok_spend`, `tiktok_conversions`, `tiktok_cpa` |
| `attribution:` in the spec | `organic_installs`, `organic_revenue`, `organic_paid`, and with `--link`: `link_clicks`, `link_installs`, `link_revenue` |
| reviews pulled | `rating`, `low_ratings` |
| AI Visibility / `category pull` | `ai_visibility`, `category_rank` (see appeeky-spec-visibility) |

Without any of these, the user gives the value at the end (`experiment close --value`). Say so up front.

## 2. Attribution

Metrics that count installs and revenue by source (`organic_*`, `link_*`) need `attribution:` in the spec. Setting it up, the SDK and smart links: see appeeky-spec-attribution.

## 3. Start an experiment

```bash
npx @appeeky/spec experiment add --type ugc --persona new-parents \
  --hypothesis "A 'my baby slept in 5 minutes' hook gets more installs per view than the product demo" \
  --metric link_installs --days 14 --link tiktok
```

- `--type`: `screenshots`, `ugc`, `ads`, `pricing`, `other`.
- `--persona`: an id from `personas` in the spec. Write the hypothesis for that person.
- `--link [channel]` creates a smart link for the experiment (needs `attribution:`), so its installs and revenue are counted exactly. Channels: `tiktok`, `instagram`, `creator`, `tiktok_ads`, `meta_ads`, and others. Give each creator or ad its own experiment and link.
  - **App Store caveat:** link installs can only be counted on Google Play. For an App Store-only app, measure `link_clicks`.
- `--asset` records what ran: file paths, video links, ad ids.
- `--baseline` and `--target` when the user knows them. Otherwise the baseline is measured.

Rules:

- One change per experiment. A new video with a new price measures neither.
- Do not start a screenshot or pricing experiment on a store that has a listing release still being measured. Check `status` first; `free` stores are safe. Both would move conversion.
- Screenshots are not indexed for search. They change conversion, so measure `conversion` (or `organic_installs` with attribution).

## 4. Close it

`verify` closes due experiments on measured metrics (`closedExperiments` in its JSON) and lists the ones that still need a value (`dueExperiments`). For those:

```bash
npx @appeeky/spec experiment close <id> --value 3.1 --note "Hook B: 3.1 installs per 1k views vs 1.8" --playbook
```

`--playbook` adds the result to `appeeky/playbook.md`. A lost experiment is a result too: write it so it is not tried again.

## 5. Where the work for each kind lives

| Experiment | Skill |
| --- | --- |
| A new screenshot set | appeeky-spec-screenshots |
| A creator or UGC video | appeeky-spec-ugc (the post itself: appeeky-spec-social) |
| An ad creative or a campaign | appeeky-spec-ads |
| A price | this skill: `--type pricing --metric revenue` (or `proceeds`, `trial_conversion`), on a store with no listing release being measured |
| Reading the app's numbers to find the next test | appeeky-spec-attribution (`data pull`) |

When it is due, appeeky-spec-review closes it with the listing releases and updates the playbook.
