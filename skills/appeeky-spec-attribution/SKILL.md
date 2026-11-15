---
name: appeeky-spec-attribution
description: Find out where an app's installs and revenue come from in an Appeeky Spec repo, with Appeeky Analytics. Covers the `attribution:` block, self-hosted (own Cloudflare server, ATTRIBUTION_TOKEN) vs hosted (APPEEKY_API_KEY), `attribution init`, adding the SDK to the app and tying purchases from the paywall to installs through RevenueCat, smart links for experiments (`experiment add --link`), the attribution CSVs from `data pull`, and how `verify` judges listing pushes on organic installs and revenue. Use when asked to set up attribution or install tracking, "where do our installs come from?", "which source pays?", organic vs paid installs, revenue per source, Search Ads keywords by revenue, to make a tracked link for a creator, TikTok or post, or to pull the numbers for analysis. For ad spend and campaigns see appeeky-spec-ads; for designing an experiment see appeeky-spec-experiments.
metadata:
  version: 1.0.0
---

# Attribution with Appeeky Spec

With Appeeky Analytics in the app, the spec knows each install's source (organic store search, an Apple Search Ads keyword, a smart link, the Google Play referrer) and what it paid. Listing pushes are then judged on organic installs and revenue, not only ranks.

Run commands as `npx @appeeky/spec <command> --json` (add `-C <dir>` when the spec is elsewhere).

## 1. Check what exists

1. Read `appeeky.spec.yaml`. An `attribution:` block means it is set up.
2. `doctor --json`, the `attribution` line: not set up (a warning, it is optional), `via: self` without `ATTRIBUTION_TOKEN`, `via: hosted` without `APPEEKY_API_KEY` (both errors), or ok.
3. Search the app's code for `Appeeky.init` to see whether the SDK is already in.

## 2. Choose hosted or self-hosted

Ask the user; do not pick for them.

| | hosted | self |
| --- | --- | --- |
| Server | Appeeky's | the user's own Appeeky Analytics on Cloudflare |
| Secret | `APPEEKY_API_KEY` | `ATTRIBUTION_TOKEN` = the server's `ADMIN_TOKEN` |
| Purchases arrive | RevenueCat connected in Appeeky | a RevenueCat webhook to the server |

Running the server itself is outside Spec; point the user to https://appeeky.com/attribution. Secrets go in the shell env, CI secrets or the docker `.env`, never in the repo or the chat.

## 3. Set it up

The app id or package must be under `stores:` first.

```bash
npx @appeeky/spec attribution init --json                                       # hosted
npx @appeeky/spec attribution init --self https://analytics.example.workers.dev --json   # self
```

Hosted turns attribution on for each store's app in Appeeky. Self finds the app on the server by its store id, or creates it. Both write the block and return `apps` (each with `store`, `id`, `publishableKey`, `endpoint`, `created`):

```yaml
attribution:
  via: self                                   # your own server: ATTRIBUTION_TOKEN
  url: https://analytics.sleepsounds.workers.dev
  app:                                        # one id per store; a string when there is one
    app-store: app_ios123
    google-play: app_and456
```

An app in Appeeky Analytics is one platform, so an app on both stores has one id per store. Each release is measured with its own store's app.

## 4. The SDK and purchases (a code change: propose it)

Without `--json`, `attribution init` prints what to add. For React Native and Expo (Swift, Kotlin and Flutter work the same way):

```ts
Appeeky.init({ key: Platform.OS === "ios" ? "<ios publishableKey>" : "<android publishableKey>" });
// self-hosted adds: endpoint: "<url>/v1/ingest"
Appeeky.identify(await Purchases.getAppUserID());
```

- `identify` with the RevenueCat app user id is what ties a paywall purchase to the install that led to it. Call it once RevenueCat is configured.
- Self-hosted: in RevenueCat, Integrations → Webhooks, URL `<url>/v1/webhooks/revenuecat`, the Authorization header set to the server's `REVENUECAT_WEBHOOK_SECRET`, environment production. Hosted: connect RevenueCat in Appeeky.
- The publishable key is meant for the app. The RevenueCat dashboard, the webhook and the app release are the person's work: show the diff and the steps, do not change settings silently, and say numbers only start with the first build that ships the SDK.

## 5. Read where installs come from

`data pull --json` (`--days`, default 90) writes these next to the other data files. It works with `attribution:` alone (self), without an Appeeky key:

| File | Holds |
| --- | --- |
| `attribution-sources.csv` | installs, trials, payers and revenue by source and campaign (installs first seen in the window) |
| `attribution-keywords.csv` | Apple Search Ads keyword ids with installs, payers and revenue, last 30 days. Apple reports ids; the words are in the Search Ads account (`appeeky/search-ads/keywords.csv` when pulled) |
| `attribution-campaigns.csv` | by campaign; `(none)` is organic and other installs without one |
| `attribution-countries.csv` | by country |
| `smart-links.csv` | clicks, installs and revenue per link, last 30 days |

With both stores, each file gets a `-app-store` / `-google-play` suffix. Read `.appeeky/data/README.md` first. The files are private (`.appeeky/data/`, gitignored, unless `data: { commit: true }`): never paste them into a public pull request.

Answer with numbers, windows and the file: "organic: 412 installs, 9.1% paid, $1,240 in the last 90 days (attribution-sources.csv)". Revenue covers what those installs paid so far, so recent installs look poorer than old ones.

## 6. Smart links

`experiment add --link [channel]` creates one link that sends each visitor to the right store and counts its installs and revenue to the experiment:

```bash
npx @appeeky/spec experiment add --type ugc --persona new-parents --link tiktok \
  --hypothesis "Before/after hooks get installs" --metric link_installs --target 50
# Started 2026-11-04-before-after-hooks-get-installs
#   link https://analytics.sleepsounds.workers.dev/l/ab12xy
```

- Channels include `tiktok`, `instagram`, `creator`, `tiktok_ads`, `meta_ads`. One link per creator, post or ad, each its own experiment.
- Metrics: `link_clicks`, `link_installs`, `link_revenue`. `verify` closes it from the link when due.
- iOS: the App Store passes nothing from a click to a new install, so link installs are in practice Google Play installs (the SDK reads the link on first launch). For an App Store only app use `link_clicks` or `organic_installs`; the CLI warns.
- Experiment design and closing: appeeky-spec-experiments.

## 7. How verify uses it

For a listing push where no target moved, the outcome decides in this order: organic installs and their revenue (with `attribution:`), then store conversion, then downloads, then category rank. `verify` reports organic installs per day, organic revenue per day and the share of organic installs that paid, and calls a change inside the metric's normal range flat. Experiments can use `organic_installs`, `organic_revenue` or `organic_paid` as their metric. Reading verdicts: appeeky-spec-review.

## Mistakes to avoid

- Committing `ATTRIBUTION_TOKEN`, `APPEEKY_API_KEY` or the RevenueCat webhook secret.
- Adding the SDK or changing RevenueCat without telling the user; it ships in an app release.
- Skipping `identify`: installs then never connect to purchases.
- Measuring `link_installs` on an App Store only app.
- Reusing one smart link across creators or ads: their results mix.
- Reading Search Ads keyword ids in `attribution-keywords.csv` as words, or inventing a source's numbers when a file is missing.
