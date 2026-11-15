---
name: appeeky-spec-seo
description: Improve and measure the app's own website in Google from an Appeeky Spec repo (web.url in appeeky.spec.yaml, appeeky/seo/). Use when asked to audit or check the website's SEO, fix title, meta description, sitemap, robots.txt, store links, Smart App Banner or structured data, let AI crawlers read the site, write llms.txt, set up or pull Google Search Console (GSC_CREDENTIALS), find queries close to page one, check Google ranks, AI Overview citations, keyword volume or competing sites on DataForSEO, or measure a website change. Website only. Whether ChatGPT, Claude, Gemini or Perplexity name the app (intents, prompts, visibility scans) is appeeky-spec-visibility; store keywords are appeeky-spec-targets.
metadata:
  version: 1.0.0
---

# The app's website in Google with Appeeky Spec

This skill is about the website people reach from Google: is it readable by crawlers, what it ranks for, and whether a change helped. Whether AI assistants recommend the app is measured separately by `visibility` (appeeky-spec-visibility); this skill only makes the site readable and citable for them.

Run commands as `npx @appeeky/spec <command> --json`.

## 1. Point the spec at the site

```yaml
web:
  url: https://sleepsounds.app/
  searchConsole: sc-domain:sleepsounds.app   # optional; default sc-domain:<host without www>
  keywords:                                  # optional; default: the store targets
    us: [rain sounds, white noise app, sleep sounds for baby]
  countries: [us]                            # optional; default: the first country under keywords or targets
  device: mobile                             # default; desktop reads the desktop results page
```

Only one country is watched unless `web.countries` lists more. `searchConsole` can also be a URL-prefix property (`https://example.com/`).

## 2. Check the site (free)

```bash
npx @appeeky/spec seo check --json            # web.url
npx @appeeky/spec seo check https://sleepsounds.app/rain --json
```

It fetches the page, robots.txt, the sitemap and /llms.txt like a crawler that does not run JavaScript. Output: `{"url": "...", "checks": [{"area": "search|app|ai", "level": "ok|warn|fail", "message": "...", "fix": "..."}]}`. Exit code 1 when any check is `fail`; warnings do not fail it.

| Area | Checks (fail ✗ / warn !) |
| --- | --- |
| `search` | HTTPS ✗, title ✗ (! over 65 chars), meta description ✗ (! outside 70–170), noindex ✗, robots.txt blocking Googlebot ✗, canonical !, `<html lang>` !, exactly one h1 !, Open Graph !, sitemap ! |
| `app` | App Store link, Smart App Banner (`apple-itunes-app` with the app id), Google Play link, `SoftwareApplication`/`MobileApplication` JSON-LD, the app's name in the text (all !) |
| `ai` | robots.txt blocking GPTBot, OAI-SearchBot, ChatGPT-User, ClaudeBot, Claude-SearchBot, PerplexityBot, Google-Extended or Applebot-Extended ✗; llms.txt !; under 150 words readable without JS !; under 30% of targets on the page !; no FAQPage !|

Each non-ok item has a `fix`. Turn them into a short list for the website's repo, `fail` first. Do not edit the website repo unless the person asked you to work there.

## 3. llms.txt (free)

```bash
npx @appeeky/spec seo llms                       # print it
npx @appeeky/spec seo llms --out public/llms.txt # write it
```

Built from the listing: name, subtitle, the first paragraph of the description, store, website and support links, up to 12 targets of the first storefront ("Good for"), and the full description. Improve the listing first if it reads badly; llms.txt repeats it. The file belongs at the site root, so it goes into the website's repo, proposed in a pull request there.

## 4. Search Console (free, the person's own Google account)

Setup, done by the person: create a service account in Google Cloud, enable the Search Console API, add the service account's email as a user of the property, and set `GSC_CREDENTIALS` to the JSON key or its path (a CI secret, never a committed file). `GOOGLE_APPLICATION_CREDENTIALS` also works, and on a laptop a gcloud login does:

```bash
gcloud auth application-default login --scopes=https://www.googleapis.com/auth/webmasters.readonly,https://www.googleapis.com/auth/cloud-platform
```

```bash
npx @appeeky/spec seo pull --json
# {"property": "sc-domain:sleepsounds.app", "days": 58, "last28": {"clicks": 412, "impressions": 18300}, "previous28": {...}, "queries": 250, "pages": 41, "files": [...]}
```

Files in `appeeky/seo/` (data ends two days ago; Search Console lags):

- `daily.csv`: `date,clicks,impressions,ctr_pct,position`, the last 60 days merged into what is there (history is kept).
- `queries.csv`, `pages.csv`: the last 28 days, up to 250 queries and 100 pages.
- `summary.md`: clicks and impressions vs the 28 days before, top queries, **close to the top** (position over 3 and up to 15, 20+ impressions), and queries containing a store target.

A 403 means the account is not a user of the property. The daily workflow runs `seo pull` when the `GSC_CREDENTIALS` secret is set.

## 5. DataForSEO (paid, the person's own account)

Needs `DATAFORSEO_LOGIN` and `DATAFORSEO_PASSWORD`. Keywords are `web.keywords.<country>`, else the store targets.

| Command | Writes | Rough cost |
| --- | --- | --- |
| `seo ranks` | `ranks.csv` (`date,country,keyword,position,url,store_position,ai_overview,ai_cites_you`, one snapshot a day), `ranks.md`, `changes.md` (into/out of top 10, AI Overview citation gained/lost) | ~$0.02 a keyword |
| `seo keywords` | `keywords.csv` (`country,keyword,volume,difficulty,cpc,intent,tracked`; `tracked: no` = related ideas) | ~$0.01 + ~$0.0135 for each of up to 3 seed keywords, per country |
| `seo domain` | `domain.md`, `ranked-keywords.csv`: what the site ranks for, estimated visits, competing domains | ~$0.05 |

`position` is the site's best organic position (empty = not in the top 100); `store_position` is where the app's App Store or Google Play page ranks on Google.

Spending rule: always run `--dry-run` first and show the person the keywords and the quote. Run with `--yes` only after they say yes. Without `--yes`, a non-interactive run spends nothing. With `--json --dry-run` a quote line is printed before the JSON. The daily workflow already runs `seo ranks --every 7 --yes` when `DATAFORSEO_LOGIN` is set (`SEO_RANKS_EVERY` changes the days), so a manual ranks check is rarely needed.

## 6. Decide what to change

1. Fix every `fail` from `seo check` first: a noindex or a blocked crawler outweighs any copy change.
2. Queries close to the top in `summary.md` are the cheapest wins: a page title, description or section that uses the query's words. Name the query, its position and impressions, quoted from the file.
3. `keywords.csv` ideas with high volume and low difficulty: candidates for `web.keywords`; if people would also search the store for them, propose them as store targets through appeeky-spec-targets.
4. `ranks.md`: a keyword where the store page outranks the site may not need a site page. Where `ai_overview` is yes but the site is not cited, the cited domains show the kind of page that wins.
5. `domain.md` competitors: sites that rank for the same searches. Look at what their pages cover.

A website change is the person's change in their website repo, not a listing push. If the listing copy should change too, that goes through appeeky-spec.

## 7. Measure it

`verify` reads, from files in the repo:

- `search_clicks`, `search_impressions`: daily averages from `daily.csv` (needs 3+ days in a window).
- `web_rank`: average site position over `ranks.csv` rows, 101 for not ranked; lower is better.
- `ai_overview_citations`: share of `ranks.csv` rows where the AI Overview cites the site.

They appear in a release's day-7 and day-14 metric changes; a change under 5% counts as flat. They never decide a listing release's verdict (ranks, then organic installs, conversion, downloads and category rank do). To judge a website change on its own, track it as an experiment:

```bash
npx @appeeky/spec experiment add --type other --hypothesis "A /rain-sounds page lifts Google clicks" --metric search_clicks --days 28
```

`experiment add` does not measure a baseline for these metrics: pass `--baseline` from `daily.csv` (the average of the last 28 days; quote the rows). When the experiment is due, `verify` reads the value from the same files and closes it; if it stays in `dueExperiments`, close it with `experiment close <id> --value <n> --playbook`.

## Mistakes to avoid

- Running `seo ranks|keywords|domain` with `--yes` without the person's go-ahead.
- Committing the Google service account JSON or DataForSEO credentials.
- Treating a `warn` like a `fail`, or a Search Console dip in the last two days as real.
- Reading `web_rank` as "higher is better".
- Using this skill to answer "does ChatGPT recommend us": that is `visibility` (appeeky-spec-visibility).
- Quoting clicks, positions or volumes not in `appeeky/seo/` files.
