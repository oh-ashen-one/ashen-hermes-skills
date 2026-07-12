# ashen-hermes-skills

Autonomous "textable" skills for the Firelink Shrine operation — self-contained files you
text to Midir (Hermes) on Telegram, and Hermes installs them on the Studio.

Same DNA throughout: cheap/local-model-powered, dedup + state in plain code, delivers to
Telegram, feeds the content machine, never auto-posts.

## Skills
- **`skills/content-scout.skill.md`** — ✅ DONE. Always-on AI-news → content-idea firehose.
  Polls Google News + r/LocalLLaMA + r/singularity + HuggingFace (trending + papers) +
  Hacker News every ~12 min; DMs a link + gist + TikTok hook + long-form angle. Hybrid
  delivery (instant BREAKING + rolled-up RADAR). Enrichment on local GLM-5.2 (free);
  falls back to link+headline if it's down. Validated end-to-end against live feeds.
  Text this whole file to Midir to install.

- **Trend/Format Scout** — 🚧 IN PROGRESS. See `PLAN-trend-scout.md` + `STATUS.md`. Twice-
  daily "formats to ride" digest from vidIQ (Dark Souls / AI / Merl lanes). Data layer
  validated; helper + exportable file not yet written.

## Continue here
Read `STATUS.md` — it has exactly where things stand, the validated vidIQ queries (so no
credits get re-spent), and the next build steps.
