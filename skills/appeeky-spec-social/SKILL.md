---
name: appeeky-spec-social
description: Draft, check and measure organic social posts, and handle comments, in an Appeeky Spec repo (appeeky/social/). Use when asked what to post this week, to announce a release or feature on X, Instagram, TikTok, LinkedIn, Pinterest, Reddit, YouTube, Threads, Facebook or Bluesky, to write or fix a post draft, why `social check` fails, how past posts did, which hooks work, to schedule posts, or to answer, hide, like or delete comments. Posts go out only after a person merges them. For making a creator video see appeeky-spec-ugc; for the store listing see appeeky-spec.
metadata:
  version: 1.0.0
---

# Organic social with Appeeky Spec

Posts are files. You draft them in `appeeky/social/drafts/`, a person reviews them in a pull request, and they go out after the merge through the accounts connected in Appeeky. Needs `APPEEKY_API_KEY` and the Startup plan or above in Appeeky; accounts are connected in Appeeky, not in the repo.

Run commands as `npx @appeeky/spec <command> --json`.

## 1. Pull and read

`social pull --json` (`--limit <n>` recent posts, default 100) writes `appeeky/social/`:

| File | Holds |
| --- | --- |
| `accounts.yaml` | connected accounts: `id`, `platform`, `name`, `active`. Draft only for these |
| `posts.jsonl` | posts made through Appeeky: `status`, `text`, `scheduledFor`, per platform `url`, `publishedAt`, `error`, `draft` (in the TikTok inbox) and `metrics` (views, impressions, reach, likes, comments, shares, reposts, saves, clicks) |
| `native-posts.jsonl` | posts made in each platform's own app, same metrics. Often most of an account's history |
| `comments.jsonl` | recent comments (`own: true` is the account itself, `repliedAt` means answered) |
| `insights.yaml` | followers and growth per account |
| `summary.md` | last 30 days per platform and account, the posts that did best, what sits in the TikTok inbox, what is waiting to go out |

The JSON output has counts and `warnings`. A file the API could not fill this time is kept from the last pull. If `appeeky/social/` is missing, the key or the connected accounts are missing: say so, do not guess handles.

Also read before drafting: existing drafts (do not repeat one), `whatsNew` in `appeeky/store/**`, `status --json` and `log --json` (what shipped and won), `appeeky/reviews/summary.md` (what users love, in their words), `.appeeky/data/README.md` and the attribution files it lists (which channels and smart links bring installs that pay), and UGC experiments in `appeeky/experiments/` so posts and creator videos do not repeat each other.

## 2. Decide what to post

- One reason per post: a new version or feature, a result worth saying plainly (a rank won, a rating that rose), something users praise that posts never mention, a question commenters keep asking.
- Copy the hooks (first lines) that did best on that account in `summary.md`, `posts.jsonl` and `native-posts.jsonl`. Each account has its own audience: write for its theme.
- Only what the app does today. No invented numbers, dates or quotes; quote users only from `appeeky/reviews/`, without names, and cite the file.

## 3. Write the draft

One file per post: `appeeky/social/drafts/<date>-<slug>.yaml`. The schema is strict; only these keys exist: `accounts`, `text`, `when`, `media`, `tiktok`, `youtube`, `reddit`, `why` (plus `status`, `postId`, `sentAt`, which `push` writes; never set them).

```yaml
accounts: [twitter, tiktok:@sleepysounds]
text: "Fan sounds are here. Pick a fan, set a timer, sleep."
when: "2026-11-20T16:00:00Z"     # optional ISO time; without it, it goes out when sent (on merge)
media: ["https://cdn.example.com/fan.mp4"]   # public URLs; type is read from the extension
tiktok: { disclosure: own_brand, inbox_draft: true }
why: "Fan sounds shipped in 3.2; 9 reviews asked for it (appeeky/reviews/summary.md)."
```

`accounts` entries: `twitter` / `x` / `instagram` … (only when one account is connected on that platform), `tiktok:all` (every account on it), `@handle` or `u/name`, `tiktok:@handle`, or an `id` from `accounts.yaml`. With several accounts on a platform, name the handle.

What each platform takes (text counted in characters):

| Platform | Text | Media |
| --- | --- | --- |
| X | 280 | optional |
| Threads | 500 | optional: up to 20 images and 10 videos |
| Bluesky | 300 | optional: up to 4 images or one video, not both |
| Facebook | 63,206 | optional: up to 10 images or one video, not both |
| LinkedIn | 3,000 | optional |
| Reddit | 40,000 (body) | optional; `reddit:` block required |
| YouTube | 5,000 (description) | exactly one video, no images |
| Instagram / TikTok / Pinterest | 2,200 / 2,200 / 500 | at least one image or video |

Platform blocks (every key optional unless marked):

- `tiktok:` `privacy` (`public` default, `friends`, `followers`, `only_me`), `disclosure` (`none` default, `own_brand`, `paid_partnership`), `inbox_draft`, `comments` (false turns off), photo posts only: `title` (≤ 90), `auto_music`, `cover_index`; video posts only: `cover_time_ms`, `cover_image_url` (https). At most 35 images; images and a video never mix.
- `youtube:` `title` (≤ 100, default the first line of the text), `visibility` (`public`, `unlisted`, `private`), `made_for_kids`, `synthetic_media`.
- `reddit:` `subreddit` (**required**, with or without `r/`), `title` (≤ 300, default the first line), `link` (makes it a link post, text as the body), `flair_id` (when the subreddit requires flair).

Disclosure rules to apply every time:

- A post that promotes the user's own app is branded content: `tiktok: { disclosure: own_brand }`.
- AI-generated or AI-altered media (UGC clips, generated images): on TikTok set `inbox_draft: true` so the creator adds TikTok's AI-generated label in the app before posting (Spec cannot set it); on YouTube set `synthetic_media: true`. `made_for_kids: true` only when the app is for children.
- Reddit: a subreddit whose rules allow it, written for that community, saying plainly that the poster made the app. Never the same text in several subreddits.

Media must be a public URL that exists. For a clip rendered with `ugc generate --local`, use the `videoUrl` the brief recorded. If there is none, write the draft and say a person has to add the media.

Put a smart link in the text when `attribution:` is set up, one per post or channel, so the post is measured: `experiment add --type other --link <channel> --metric link_clicks --hypothesis "…"` prints it. It creates the link in the attribution service, so run it only when the user agrees. Use `link_installs` for an app on Google Play; App Store link installs cannot be counted.

## 4. Check

`social check --json` checks every draft still marked `draft` against `accounts.yaml` and exits 1 on any issue (`{ drafts: [{ file, issues }] }`). Fix and run again until `issues` is empty everywhere. The reasons it refuses:

- `no connected account for …`; `… matches N … accounts (…): name the ones to post on … or write <platform>:all`
- `<platform>: <n>/<limit> characters`
- `<platform> needs an image or video (media:)`, `youtube needs a video`, `youtube: needs exactly one video (media:), and no images`
- `a post is either images or one video, not both`, `<n> images, at most <m>`, `<n> videos, at most <m>`
- TikTok: `disclosure … can't be privacy only_me` (unless `inbox_draft: true`), title over 90, photo keys on a video or video keys on images, `cover_index` past the last image
- YouTube title over 100 (or no title and a first line over 100); Reddit missing `subreddit`, title over 300
- `<block>: options set, but the draft goes to no <platform> account`
- `when … is not in the future`. Checked again at send time: schedule far enough ahead for the review, or leave `when` out.

Without `accounts.yaml`, every account is "not connected": run `social pull` first.

## 5. Send: after the merge only

Open a pull request with the drafts. Its check (`report`) lists the posts the merge sends and any issues. On merge, the push workflow from `init --ci` (or the docker `push` routine) runs `social push --yes`; each draft gets `status: sent`, `postId` and `sentAt` and is committed, so it never goes out twice. `push` refuses everything if any draft has issues.

Never run `social push` yourself unless the user explicitly asks for it in this conversation, and never with `--yes` on your own. Posts are public at once.

## 6. Measure

After the next `social pull` (daily in `listing-daily.yml`), find the sent post in `posts.jsonl` by the draft's `postId` and read each platform's `metrics`. Compare with that account's earlier posts, not other platforms. A TikTok inbox draft has `draft: true` and no URL until the creator posts it in the app. Installs come from the smart link (`data pull` → `smart-links.csv`, or the experiment closed by `verify`); views alone do not show installs. Write what worked to `appeeky/playbook.md` (with `experiment close <id> --value … --playbook` when a link experiment exists).

## 7. Comments

1. `social comments --unanswered --json` (`--account tiktok:@handle`, `--post <appeeky post id>`, `--limit` up to 200). Each comment has `id`, `platform`, `account`, `author`, `text`, `postUrl` and `canReply`, `canHide`, `canLike`, `canDelete`.
2. Draft a reply for questions, bug reports and praise worth thanking; skip spam and `canReply: false`. Short, specific to what they wrote, in the account's voice, no promises, no links unless asked. Reply limits: X 280, Bluesky 300, TikTok 150, Threads 500, LinkedIn 1,250, Instagram 2,200, Facebook 8,000.
3. Show each as the exact command for the user to approve:
   `npx @appeeky/spec social reply <id> "<reply>"`. Run it only when the user approves that reply. It checks the limit first and uses an idempotency key, so a retry posts once.
4. Hide, like or delete only the comment the user named:
   - `social hide <id>` (`--unhide` to undo): Facebook, Instagram, Threads, X, TikTok.
   - `social like <id>`: Facebook, X, Bluesky, Reddit, LinkedIn, TikTok.
   - `social delete-comment <id>`: cannot be undone; outside a terminal it needs `--yes`, which only the user's explicit request justifies. Threads, TikTok, Bluesky, Reddit and X only allow deleting the account's own comments.

## Mistakes to avoid

- Drafting for a handle that is not in `accounts.yaml`, or a bare platform name when it has several accounts.
- One text pasted to every platform: an X post over 280, a Reddit post with no subreddit, Instagram/TikTok/YouTube without media.
- Promoting the app on TikTok without `disclosure: own_brand`, or posting an AI clip without `inbox_draft: true` / `synthetic_media: true`.
- A `when` that passes before the pull request is merged.
- Media URLs without a file extension: they count as neither image nor video and fail the media checks.
- Editing a draft marked `sent`: it will not go out again; write a new one.
- Sending, replying, hiding, liking or deleting without the user's go-ahead; scheduled routines deny these commands to the agent anyway.
- Judging a post on likes when the question was installs: use its smart link.
