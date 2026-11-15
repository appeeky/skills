# Appeeky skills

Agent skills for [Appeeky Spec](https://spec.appeeky.com): they teach your coding agent (Claude Code, Codex, Cursor, Gemini CLI, opencode and the rest) the listing-as-code workflow.

```bash
npx skills add appeeky/skills
```

Then ask your agent: "Set up Appeeky Spec for this app."

| Skill | What it is for |
| --- | --- |
| `appeeky-spec` | The whole workflow: set up, check, diff, push, verify |
| `appeeky-spec-targets` | Choose the searches the app should be found for |
| `appeeky-spec-localize` | Add or improve a language or country |
| `appeeky-spec-review` | Judge past listing changes, keep or revert, plan the next |
| `appeeky-spec-experiments` | Screenshots, UGC, ads and pricing experiments |
| `appeeky-spec-visibility` | AI answers, the website in Google, the category chart |

Install one with `npx skills add appeeky/skills --skill appeeky-spec-targets`.

These files are copied here automatically from each Appeeky Spec release, so they always match the CLI on npm (`@appeeky/spec`). Edits made here are overwritten by the next release.
