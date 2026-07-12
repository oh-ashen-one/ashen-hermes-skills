# STATUS — pick up here on the other machine

Date: 2026-07-12. Author: Claude (Fable 5) working with Hari.

## content-scout — DONE ✅
- Deliverable: `skills/content-scout.skill.md` (also lives at `~/content-scout.skill.md` on
  the origin Mac). Fully built + validated against live feeds (dry-run): Google News parses
  (692 items incl. the GPT-5.6 launch), HF caught real Qwen/GLM/Tencent drops, dedup works,
  over-cap items resurface next cycle, local-model enrich falls back cleanly.
- To ship: text the whole file to Midir on Telegram → Hermes installs on the Studio.
- Only blanks Hermes fills: `telegram_chat_id` + `telegram_bot_token` (reuse souls-clipper's).
- OPEN ITEM: delete the old `oh-ashen-one/ai-news-watcher` repo (the superseded Discord
  version). Blocked on gh scope — run `gh auth refresh -h github.com -s delete_repo` then
  `gh repo delete oh-ashen-one/ai-news-watcher --yes`.

## trend-scout — IN PROGRESS 🚧
Full spec in `PLAN-trend-scout.md`. Twice-daily (AM+PM) "formats to ride" Telegram digest.
Engine = **Hermes** (Hermes makes the vidIQ MCP calls; a stdlib helper does dedup/budget/send).

### Decisions locked
- Lanes: Dark Souls/soulslike, AI news/tools/models, Merl/AI-app promo.
- vidIQ = 5 credits/call, connected as your-google-account@example.com, channel UCXAN6x2NtbQ49s9ScCg6wIw.
- Hard `daily_credit_budget` = 60. Balance was ~2,533 (1,629 non-refill + 904 renewable,
  resets 2026-08-03).
- Preflight: Hermes calls `vidiq_balance` (0cr) first; if the vidIQ connector isn't present,
  STOP + tell Hari (don't spend, don't crash). Fallback engine = Claude Code scheduled agent.

### Data layer VALIDATED (already spent ~15 cr — don't re-run these to verify)
Primary call per lane = `vidiq_instagram_tiktok_outlier_search`, `embeddingType: "format"`,
`resultsPerPlatform: 5`. It returns per-item: reel/tiktok_concept, niche, hook_0_3s
(visual/text/audio), format (style/template/intended_value/is_looped), effort, execution
(pacing/text_overlays/visual_changes), audio, audience. Rich enough to extract the format +
write an angle. Exact params that worked:

- **Dark Souls** — query: `"Dark Souls Elden Ring soulslike boss fight gameplay clips with facecam reaction"`
  audienceQuery: `"Culture/Region: Global gaming; Global: true; Demographics: 16-30 male-leaning soulslike gamers;"`
  (Note: pulls broader gaming — DBD/Deltarune/CS2/Warzone — but the transferable FORMAT is
  consistent: split-screen creator+gameplay, "most broken feature ever" hook, clutch moment.
  Angle-writer maps it onto Hari's DS1 clips. Tighten the query later if you want soulslike-only.)
- **AI news/tools** — query: `"new AI model tool ChatGPT Claude local AI demo reaction explainer"`
  audienceQuery: `"Culture/Region: Global tech; Global: true; Demographics: 18-35 tech-curious creators;"`
  (Surfaced ChatCut/Codex editor, Krea realtime, HeyGen, ElevenLabs, Cursor, and a
  "Vibe coders building a SaaS with Claude Fable 5" clip @138K/17x — very on-lane.)
- **Merl/AI-app** — query: `"AI app I tried this app before after POV honest review phone screen recording"`
  audienceQuery: `"Culture/Region: Global consumer app; Global: true; Demographics: 16-30 gen-z app users;"`
  (Surfaced the alarm-app-you-play-to-turn-off @887K/156x, AI job tool, ChatGPT skits,
  face-rating apps — all directly mappable to Merl's blank-canvas→finished-cast promo.)

Optional cross-check (budget permitting, rotate ~1/run): `vidiq_outliers` YouTube,
`keyword`=lane, `contentType:"short"`, `publishedWithin:"thisWeek"`.

### Next build steps (not yet done)
1. Write `trendscout.py` (stdlib helper) with subcommands:
   - `filter-new` (stdin JSON items[] with `id` → dedup vs state/seen.jsonl, record new,
     append new to `~/content-radar/briefs/trends-<date>.jsonl`, print new items).
   - `budget check` / `budget spend N` (per-date ledger `state/budget-<date>.json`, enforce
     `daily_credit_budget`, exit 1 if over so the skill stops).
   - `send` (stdin text → Telegram Bot API sendMessage; `--dry-run` prints).
2. Write `config.json` (the 3 lanes above + telegram creds placeholders + daily_credit_budget
   60 + results_per_lane 5 + published_within "thisWeek").
3. Assemble `skills/trend-scout.skill.md` — same exportable/install pattern as content-scout:
   preflight → budget-gate → per-lane vidiq call → filter-new → Hermes writes format+angle →
   send digest. Two Hermes crons (~08:00, ~18:00).
4. Test the helper in scratchpad (dedup across runs, budget rollover by date, send dry-run,
   JSON/syntax parse). Ship the file. Hermes runs preflight + one AM dry digest for eyeball.

### Message shape (locked) — see PLAN-trend-scout.md "Message shape".

## Reuse notes
- content-scout.skill.md is the structural template (Telegram Bot API send, seen.jsonl dedup,
  config placeholders, install-instructions-to-Midir header).
- Telegram bot token: reuse souls-clipper's (`~/skills/souls-clipper/.../send_clips.py`).
- Repos rule: keep clones in `~/dev`, never `~/Documents`/`~/Desktop` (iCloud hangs).
