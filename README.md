# Appeeky skills

Agent skills for [Appeeky Spec](https://spec.appeeky.com): they teach your coding agent (Claude Code, Codex, Cursor, Gemini CLI, opencode and the rest) how to grow an app from its repo.

```bash
npx skills add appeeky/skills
```

Then ask your agent: "Set up Appeeky Spec for this app."

| Skill | Use it for |
| --- | --- |
| `appeeky-spec` | Set up the spec and `doctor`, find gaps, edit, check and push listing changes |
| `appeeky-spec-targets` | Personas, keyword research and which searches to target |
| `appeeky-spec-localize` | A new language or country, and spare keyword space in secondary locales |
| `appeeky-spec-review` | Day-7/14 results, keep or revert, playbook, closing experiments |
| `appeeky-spec-cofounder` | The weekly review in any agent: read everything, one listing change, a brief and a pull request |
| `appeeky-spec-reviews` | Review themes with counts, replies drafted within each store's limit, sent after the merge |
| `appeeky-spec-competitors` | Finding competitors, reading their changes, what to do when one moves |
| `appeeky-spec-screenshots` | App Store screenshot sets, their sizes and counts, screenshot tests |
| `appeeky-spec-social` | Posts as files for every platform, `social check`, comments and replies |
| `appeeky-spec-ugc` | Creator personas, video briefs, cost first, which hooks worked |
| `appeeky-spec-ads` | Apple Search Ads on your key, Meta and TikTok, paid searches that should be targets |
| `appeeky-spec-attribution` | Where installs and revenue come from: the SDK, smart links, `data pull` |
| `appeeky-spec-experiments` | Any experiment's hypothesis, metric and result; pricing tests |
| `appeeky-spec-visibility` | Whether AI assistants name your app and how to get named; the category chart rank |
| `appeeky-spec-seo` | The app's website: `seo check`, `llms.txt`, Search Console, Google ranks |
| `appeeky-spec-automation` | The GitHub workflows and their secrets, other CI, cron, the weekly routine, failed runs |
| `appeeky-spec-server` | Running it on a VM or container host: Docker, the `.env`, logs, updates |

Install one with `npx skills add appeeky/skills --skill appeeky-spec-server`.

These files are copied here automatically from Appeeky Spec, so they always match the CLI on npm (`@appeeky/spec`). Edits made here are overwritten by the next copy.
