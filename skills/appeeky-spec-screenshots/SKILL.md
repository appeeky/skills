---
name: appeeky-spec-screenshots
description: Work on an app's App Store screenshots in an Appeeky Spec repo (appeeky/screenshots/). Use when asked to download or pull the live screenshots, check screenshot sizes or counts ("wrong size", "too many screenshots", a failing screenshot check in CI), decide what the screenshots should say or in what order (captions, first screenshot, which feature leads), prepare a new screenshot set, or test and measure whether new screenshots lift conversion. Not for listing text such as name, subtitle or keywords (use appeeky-spec), and not for UGC, ads or pricing experiments (use appeeky-spec-experiments).
metadata:
  version: 1.0.0
---

# App Store screenshots with Appeeky Spec

Screenshots are not indexed for search. They decide whether someone who opened the product page installs, so a new set is judged on conversion, never on ranks.

Run commands as `npx @appeeky/spec <command> --json`.

What Spec does and does not do with screenshots:

- It downloads the App Store set (`screenshots pull`) and checks sizes and counts (`screenshots check`, also part of `check` and `report`).
- It does **not** upload them. `push` sends listing text only; when screenshot errors exist it prints them and goes on ("Screenshots are not uploaded by push"). The person uploads a new set in App Store Connect.
- Google Play screenshots are not handled.

## 1. Read first

- `appeeky.spec.yaml`: `personas` (who the screenshots speak to) and `targets` (the words people search).
- `appeeky/reviews/summary.md`: what users praise and ask for. A feature people look for and cannot see is a screenshot caption.
- `appeeky/playbook.md` and `appeeky/experiments/*.yaml` with `type: screenshots`: what was tried and how it did. Do not re-run a lost idea.
- `status --json`: open loops and `free` (stores with no listing release being measured).

## 2. Pull the live set

```bash
npx @appeeky/spec screenshots pull --json            # every locale → {"downloaded": 24}
npx @appeeky/spec screenshots pull --locale en-US de-DE
```

It uses the same App Store connection as `pull` and reads the version in Prepare for Submission when there is one, otherwise the live version. Files land in:

```
appeeky/screenshots/app-store/<locale>/<DISPLAY_TYPE>/01.png, 02.png, …
```

`<DISPLAY_TYPE>` is App Store Connect's name, e.g. `APP_IPHONE_67`, `APP_IPAD_PRO_3GEN_129`. The number is the position in the set. Pulling overwrites files with the same name but never deletes extra ones, so clear a display type's folder before pulling it again if the live set got shorter.

Look at the images (they are PNGs you can open) before proposing anything.

## 3. Check sizes and counts

```bash
npx @appeeky/spec screenshots check --json
```

Output: `{"issues": [{"severity": "error", "rule": "screenshots", "store": "app-store", "locale": "en-US", "field": "APP_IPHONE_67", "message": "03.png is 1284×2778; APP_IPHONE_67 takes 1290×2796, 1320×2868, 1260×2736."}]}`. Exit code 1 on any error.

The rules, from the source:

| Display type | Accepted portrait sizes (landscape: swapped) |
| --- | --- |
| `APP_IPHONE_67` | 1290×2796, 1320×2868, 1260×2736 |
| `APP_IPHONE_65` | 1242×2688, 1284×2778 |
| `APP_IPHONE_61` | 1179×2556, 1206×2622, 1170×2532 |
| `APP_IPHONE_58` | 1125×2436, 1080×2340 |
| `APP_IPHONE_55` | 1242×2208 |
| `APP_IPAD_PRO_3GEN_129` | 2048×2732, 2064×2752 |
| `APP_IPAD_PRO_129` | 2048×2732 |
| `APP_IPAD_PRO_3GEN_11` | 1668×2388, 1640×2360 |

- At most 10 screenshots per display type and locale (error above that).
- Only `.png`, `.jpg` and `.jpeg` are read. Other display type folders get the count check only, no size check.
- There is no minimum-count check and no check of the text in the image.

`check` and `report` include these issues, and the GitHub Action fails the pull request on screenshot errors. Fix the image size; do not rename the folder to dodge the check.

## 4. Decide what the set says

The first two or three screenshots are what most people see in search results and on the page without scrolling. Decide in this order:

1. **One persona.** Pick the persona from `personas` the set is for. A set that talks to everyone lands with no one.
2. **The first screenshot carries the main promise** in that persona's words, ideally the phrasing of a top target (`rain sounds for sleep`, not `immersive audio`). The caption is text in the image: short (a few words), readable at thumbnail size.
3. **Order by what users value**, taken from `reviews/summary.md` and the review counts behind it, not from the order features were built. Cite the source when you propose the order.
4. **One idea per screenshot**: a caption plus the screen that proves it.
5. **Match the listing.** Captions should agree with the subtitle and the first lines of the description; a promise the app does not keep shows up in reviews.
6. **Per locale:** captions are written for that market (see appeeky-spec-localize), not machine-translated.

Write the proposal as a list a designer or the person can act on: position, caption, which screen, why (with the file it came from). Spec does not render images. If new images are supplied, place them in the folder layout above and run `screenshots check`.

## 5. Test it as an experiment

A new set is an experiment, tracked in `appeeky/experiments/<id>.yaml`:

```bash
npx @appeeky/spec experiment add --type screenshots --persona new-parents --store app-store --country us \
  --hypothesis "A first screenshot with a sleeping baby lifts conversion for new parents" \
  --metric conversion --days 14 \
  --asset appeeky/screenshots/app-store/en-US/APP_IPHONE_67/01.png
```

Rules:

- **Metric:** `conversion` (downloads ÷ product page views, needs `APPEEKY_API_KEY` with the store connected) or `organic_installs` when `attribution:` is set. Without either, the person gives the value at the end. Say so up front.
- **Start date = the day the new set goes live**, not the day it was designed. Run `experiment add` once the person confirms it is live.
- **Never overlap** with a listing release on the same store: check `status --json` and start only when `free` includes `app-store`. Both move conversion and neither could be read.
- **One change at a time.** Changing captions and order and the first screen together tells you the set won, not why.
- **Baseline:** measured from the same number of days before the start when the API key is set; otherwise pass `--baseline`. Pass `--target` only when the person names a value that counts as a win.

How it is measured: `verify` (run daily by the workflow) compares the metric over the `--days` window after the start with the window before. With no `--target`, a change under 5% either way is `inconclusive`; better is `won`, worse is `lost`. Conversion needs at least 50 product page views in a window, otherwise there is no value and the experiment shows in `dueExperiments`. Close those by hand:

```bash
npx @appeeky/spec experiment close <id> --value 0.036 --note "Baby-first set: 3.6% vs 3.1%" --playbook
```

`--playbook` writes the result to `appeeky/playbook.md`. A lost set is a result too. For results of past experiments and releases together, hand over to appeeky-spec-review.

## Needs a person

- Uploading screenshots to App Store Connect, and saying when they went live.
- Designing or rendering the images.
- Approving the experiment's start in a pull request.

## Mistakes to avoid

- Expecting `push` to upload screenshots, or judging a set by keyword ranks.
- Starting a screenshot experiment while a listing release on the App Store is still measured.
- Starting the experiment before the set is live, or two experiments on the same page at once.
- Putting keyword lists into captions for search: captions are not indexed.
- Inventing review counts or conversion numbers; quote the file (`reviews/summary.md`, the experiment YAML, `verify` output).
- Leaving stale `07.png`–`10.png` from an older, longer set after a pull.
