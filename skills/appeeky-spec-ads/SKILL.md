---
name: appeeky-spec-ads
description: Read and judge an app's paid channels in an Appeeky Spec repo, without changing a campaign. Covers the `ads` command and the `ads:` block for Meta and TikTok campaigns, Apple Search Ads on the user's own API key (`search-ads pull`, key setup, appeeky/search-ads/ files), paid search terms that should become targets, ads judged around listing pushes by `verify`, and ad creative or CPI experiments. Use when asked "how are the ads doing?", for Search Ads spend, installs or cost per install, which search terms convert, wasted keywords, to connect a Search Ads key, to add Meta or TikTok campaigns, to test an ad creative, or what to pause, bid or budget. Spec never changes bids, budgets or campaigns; those go to a person. For where installs come from (attribution, smart links) see appeeky-spec-attribution; for choosing targets see appeeky-spec-targets.
metadata:
  version: 1.0.0
---

# Paid channels with Appeeky Spec

Spec reads ad numbers into the repo and measures them next to the listing. It never creates, pauses, bids or budgets a campaign. Every spend decision is a suggestion for a person, with the numbers behind it.

Run commands as `npx @appeeky/spec <command> --json` (add `-C <dir>` when the spec is elsewhere).

## 1. See what is connected

Run `doctor --json` and read `appeeky.spec.yaml`. Three sources exist:

| Channel | Comes from | Needs |
| --- | --- | --- |
| Apple Search Ads | Appeeky (matched to the app, nothing in the spec) | `APPEEKY_API_KEY`, Search Ads connected in Appeeky |
| Apple Search Ads | the user's own key: `search-ads pull` → `appeeky/search-ads/` | a Search Ads API key (step 2); no Appeeky account |
| Meta, TikTok | Appeeky, only the campaigns listed under `ads:` | `APPEEKY_API_KEY`, the ad account connected in Appeeky |

If none is set up, say which one fits and stop; never estimate a channel that has no data.

## 2. Apple Search Ads on the user's own key

A person does the key setup; you give the steps:

1. In Apple Search Ads → Account Settings → API, invite an API user with a read role and upload its public key.
2. Save it on the App Store Connect CLI account the spec uses (`stores.appStore.account`, or the CLI's default):
   ```bash
   appstore accounts search-ads my-team --client-id SEARCHADS.… --team-id SEARCHADS.… --key-id … --private-key-path ./searchads-private-key.pem
   ```
3. In CI (no CLI account) set repository secrets `ASA_CLIENT_ID`, `ASA_TEAM_ID`, `ASA_KEY_ID`, `ASA_PRIVATE_KEY` (the PEM or its path). They win over the account. The daily workflow from `init --ci` runs `search-ads pull` when `ASA_CLIENT_ID` is set; the docker `.env` takes the same four.

Never ask for the private key in chat, and never commit the `.pem`.

Optional scoping in the spec, only when needed:

```yaml
stores:
  appStore:
    appId: "1234567890"
    searchAds:
      orgId: "1234567"        # only when the key reaches more than one organization
      campaigns: ["987654"]   # optional: only these campaign ids
```

Then `search-ads pull --json` (`--days`, default 30). Only campaigns for `stores.appStore.appId` are read. It writes `appeeky/search-ads/`:

| File | Holds |
| --- | --- |
| `daily.csv` | `date,spend,installs,taps,impressions,cpi,currency`, kept across pulls |
| `keywords.csv` | `keyword,match,campaign,impressions,taps,installs,spend,cpi` for the window |
| `search-terms.csv` | `search_term,campaign,impressions,taps,installs,spend,cpi` for the window |
| `summary.md` | totals, "Search terms that brought installs and are not targets", "Keywords that spent and brought no install" |

The JSON has `campaigns`, `currency`, `last30` and `errors`; report any `errors` (a campaign whose keywords or terms failed).

## 3. Meta and TikTok campaigns

Ad accounts usually promote several apps, so list only this app's campaigns. Ask the user which campaign ids promote this app; never guess from campaign names.

```yaml
ads:
  meta:
    adAccount: act_1234567890      # optional: defaults to the account picked in Appeeky
    campaigns: ["120210000000001"]
  tiktok:
    advertiser: "7000000000000"    # optional: defaults to the advertiser picked in Appeeky
    campaigns: ["1790000000000"]
```

Run `check --json` after editing the spec.

## 4. Read the numbers

1. `ads --json` (`--days`, default 30): spend, installs (conversions for TikTok) and cost per result per channel, plus `errors` per channel. It works with `APPEEKY_API_KEY`, or with `appeeky/search-ads/` alone.
2. `ads --save` writes `appeeky/ads/<date>.yaml` so the history sits in the repo next to the listing. Compare with earlier snapshots there.
3. For per-day detail: `data pull --json`, then `.appeeky/data/README.md`, `apple-search-ads.csv`, `meta-ads.csv`, `tiktok-ads.csv`. With `attribution:`, `attribution-keywords.csv` gives Search Ads keyword ids with the revenue they led to (ids only; the words are in the Search Ads account). See appeeky-spec-attribution.

Quote every number with its window and the file or command it came from.

## 5. What paid search teaches the listing

This is the part Spec can act on, through a pull request:

- "Search terms that brought installs and are not targets" in `summary.md`: people typed them and installed. Each is a candidate target and keyword: ranking for it organically is free. Hand it to the appeeky-spec-targets workflow (add to `targets:`), then `coverage --json` shows whether the listing indexes it; the keyword change itself follows the appeeky-spec workflow.
- Terms with many installs and a low cost per install are the strongest evidence; one install is not evidence.
- Ads that promise something the listing does not say (or the reverse) are a listing or creative finding.

## 6. Ads around a listing push

`verify` compares Search Ads installs and cost per install, and Meta and TikTok spend, installs and cost per install, in the days before and after each push, so an ad change and a listing change are judged on the same days. When the user changed bids or budgets inside a release's window, say so: the outcome may be the ads, not the listing. Say "overlaps with", not "caused".

## 7. Ad creative and cost experiments

One change per experiment (one creative, one audience, one bid strategy):

```bash
npx @appeeky/spec experiment add --type ads --metric meta_cpi \
  --hypothesis "The before/after creative gets installs cheaper than the demo" \
  --asset 120210000000099 --days 14
```

- Metrics measured for you: `ad_installs`, `cpa` (Search Ads), `meta_spend`, `meta_installs`, `meta_cpi`, `tiktok_spend`, `tiktok_conversions`, `tiktok_cpa`. The baseline comes from the last days; `verify` closes it when due.
- `--asset` records the ad or campaign ids that ran. `--link meta_ads` or `--link tiktok_ads` adds a smart link (needs `attribution:`; see appeeky-spec-attribution).
- Launching the ad is the person's job; the experiment only records and measures it. Close manual ones with `experiment close <id> --value <n> --note "…" --playbook`.

## 8. What goes to a person

Write these as recommendations (in the brief or the reply), never as actions:

```
- Search Ads "rain sounds app" (Exact): 38.40 EUR, 0 installs, 21 taps, last 30 days (appeeky/search-ads/summary.md). Suggest: pause or add as negative. Undo: re-enable the keyword.
```

One recommendation per campaign or keyword: never pause and raise the budget of the same one. Include the numbers and how to undo it. Bids, budgets, negatives, pausing and new spend are always the person's decision.

## Mistakes to avoid

- Changing, pausing or creating campaigns, or telling the user it was done. Spec has no command for it.
- Guessing which Meta or TikTok campaigns belong to the app from their names.
- Estimating spend or installs for a channel with no data, or numbers without a window and a source.
- Treating one or two installs as a winning term.
- Committing a Search Ads key or `.pem`, or pasting `.appeeky/data/` numbers into a public pull request.
- Calling an ad change the cause of a listing result (or the reverse) when they overlap in time.
