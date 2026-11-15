---
name: appeeky-spec-competitors
description: Track competing apps in an Appeeky Spec repo (competitors in appeeky.spec.yaml, appeeky/competitors/). Use when asked who the competitors are, to find or add competitors, to pull their listings, what a competitor changed (title, version, description, ranks on our searches), whether a competitor's move explains a rank change, or what users dislike about competing apps. Not for deciding which keywords to target after a move (appeeky-spec-targets), our own reviews (appeeky-spec-reviews) or AI assistants naming a competitor instead (appeeky-spec-visibility).
metadata:
  version: 1.0.0
---

# Competitors with Appeeky Spec

Each competitor is a file in the repo. Pulling again records what it changed, so its moves sit next to your own releases and their results.

Run commands as `npx @appeeky/spec <command> --json`. `competitors discover` and `competitors pull` need `APPEEKY_API_KEY`.

| Path | What it holds |
| --- | --- |
| `competitors:` in `appeeky.spec.yaml` | the apps you compete with |
| `appeeky/competitors/<name>.yaml` | one competitor, rewritten on every pull |
| `appeeky/competitors/changes.md` | what each competitor changed between pulls, newest first, appended by `pull` |

## 1. Choose competitors

```bash
npx @appeeky/spec competitors discover --json                 # apps like yours on the first store with an id
npx @appeeky/spec competitors discover --store google-play --json
npx @appeeky/spec competitors discover --add 3 --json         # also writes the first 3 new ones into the spec
```

Output: `{ "store", "country", "apps": [{ "id", "title", "rating", "ratings" }], "added" }`. The country is the first storefront under `targets`. `--add` skips apps already listed.

Prefer adding by hand after reading the list: keep the 3–6 apps people actually choose between, not the category giants or apps for a different need. Show the list to the user before adding many.

```yaml
competitors:
  - name: Calm Night
    appStore: "1234567891"      # numeric App Store id, quoted
    googlePlay: com.calmnight.app
```

Each entry needs `appStore` or `googlePlay`. Run `check --json` after editing the spec.

## 2. Pull

```bash
npx @appeeky/spec competitors pull --json
```

Output: `{ "results": [{ "file", "errors", "changes" }] }`. A store in `errors` was not fetched; that competitor's file is kept as it was when nothing could be fetched. The daily workflow (`listing-daily.yml`) already runs it, so read the files when they are recent.

Everything is measured in **one country: the first storefront under `targets`**, against that storefront's targets. Say so when the user asks about another market.

```yaml
name: Calm Night
pulledAt: 2026-11-01
country: us
stores:
  app-store:
    id: "1234567891"
    title: "Calm Night: Rain & Baby Sleep"
    developer: …
    rating: 4.6
    ratings: 18234
    version: "5.2"
    updated: 2026-10-18
    releaseNotes: …              # first 300 characters
    description: …               # first 600 characters
    ranks:                       # the 25 searches it ranks best for
      - { keyword: rain sounds, rank: 3, popularity: 54 }
    yourTargets:                 # where it ranks for your own targets
      - { keyword: baby sleep, rank: 6 }
    recentReviews:               # first page of recent public reviews
      count: …                   # reviews on that page
      average: 4.1
      low:                       # up to 5 at 1–2 stars, 160 characters each
        - { rating: 1, text: "Ads every few minutes…" }
```

The App Store subtitle and keyword field are not in the file; only the title is.

## 3. Read what changed

`changes.md` lines look like:

```
- 2026-11-01 · Calm Night · App Store: title "Calm Night: Sleep" → "Calm Night: Rain & Baby Sleep"
- 2026-11-01 · Calm Night · App Store: now #6 for "baby sleep" (was #12)
- 2026-10-18 · Calm Night · App Store: version 5.2: Fan and AC sounds, a new bedtime timer
```

Four kinds are recorded: a new title, a new version (with its notes), a rewritten description, and a climb into the top 10 for one of your targets. Rank drops and changes to ranks outside your targets are not; read the git diff of the competitor file (`git log -p appeeky/competitors/<name>.yaml`) for those.

For each change, answer:

1. **Does it touch one of our targets?** A title that gained or lost one of our target words, or a climb on a target.
2. **Did our rank move?** Compare with the rank baselines and day-7/14 checks in `appeeky/releases/*.yaml` (`log --json`). The competitor file does not hold our own ranks.
3. **Does a new version add a feature our users ask for?** Check against `appeeky/reviews/summary.md` (appeeky-spec-reviews).

## 4. What to do when a competitor moves

- **Keyword decisions go to appeeky-spec-targets**: whether a word the competitor dropped is now worth taking, or whether to defend a target it now attacks. Report the move with its line from `changes.md`; do not change `targets` here.
- **A listing change** only for a target that matters (popular and relevant) and only on a `free` store from `status --json`, through the appeeky-spec workflow (`check`, `diff`, push only with consent).
- **Never** use a competitor's name or brand as a keyword, in a target or in listing copy.
- **Do nothing** when the move does not touch your targets or your rank did not move. Most moves need nothing.
- **Product**: a competitor shipping a feature your users ask for is a product finding for the person, not a listing line.

## 5. Competitors' low ratings

`recentReviews.low` shows what users dislike about each alternative. When several competitors share a complaint (ads, subscriptions, a missing feature) and your app does better, that is a listing angle: a description line, promotional text or screenshot caption that says what your app does, without naming them. Quote only the stored text and name the file. Only claim what your app does today; check your own reviews for the same complaint first.

## 6. Status warns about overlaps

`status --json` (and `appeeky/loops.md`) adds a warning to every open release when a competitor changed something on the same store within 14 days before or after the push:

```
Calm Night changed on 2026-11-01, inside this window: App Store: title "…" → "…"
```

When you see it:

- Do not credit or blame the release alone for rank moves on the searches the competitor touched; say so in the verdict (appeeky-spec-review).
- Prefer the day-14 check and the control storefronts (`control.net` in the check) over raw rank moves.
- Do not revert or follow up on that store because of the competitor's move until the release is measured.

## Mistakes to avoid

- Adding every similar app from `discover`; too many competitors make `changes.md` noise.
- Reading ranks or reviews as global when they come from one country.
- Concluding the competitor did not touch its subtitle or keywords because `changes.md` is silent: those are not tracked.
- Changing targets or the listing in reaction to every move; hand keyword choices to appeeky-spec-targets.
- Using competitor names or trademarks anywhere in the listing.
- Inventing competitor ranks, ratings or quotes instead of citing `appeeky/competitors/<name>.yaml`.
