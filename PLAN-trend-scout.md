# Trend/Format Scout — twice-daily "formats to ride" digest for Hermes/Telegram

## Context
Second skill in the series (after `content-scout`). content-scout answers *what happened*
in AI; Trend Scout answers *what to make* — it surfaces the viral short-form FORMATS
popping right now in Hari's lanes and hands him a ready angle for each. Delivered to the
same Telegram, twice daily (AM + PM).

**What makes this different from content-scout (drives the whole design):**
- vidIQ is an **authenticated MCP connector, 5 credits/call** — NOT a keyless feed. So it
  **cannot be a bare Python cron**; it must run inside an agent that holds the vidIQ
  connection. Hari chose the **Hermes engine** (text-to-Midir), so Hermes itself makes the
  vidIQ calls. **Top risk:** this only works if Midir/Hermes actually has the vidIQ
  connector. Mitigation = a preflight check (below) that fails safe.
- At 5 cr/call it must be a **scheduled digest with a hard credit budget**, not a firehose.
  Twice daily, ~3–4 vidIQ calls/run → ~30–40 cr/day, hard-capped at 60. Balance is ~2,533
  (1,629 non-refill + 904 renewable, resets Aug 3) → months of runway.

**Locked choices:** engine = Hermes skill; lanes = Dark Souls/soulslike, AI news/tools/models,
Merl/AI-app promo; cadence = twice daily (AM+PM). vidIQ connected as your-google-account@example.com,
channel `UCXAN6x2NtbQ49s9ScCg6wIw`. 87 trend-format categories available via
`vidiq_trend_categories` (speedrun, reaction, tier_list, everything_wrong, how_to,
honest_review, i_tried, pov…).

## Deliverable
One exportable file `~/trend-scout.skill.md` (text it to Midir, same as content-scout),
containing a SKILL.md procedure + two support files Hermes writes to
`~/.hermes/skills/trend-scout/`:
- `SKILL.md` — the twice-daily procedure Hermes follows (preflight → per-lane vidIQ calls →
  dedup/budget via helper → write angles → send digest).
- `trendscout.py` — stdlib helper for the deterministic parts: dedup ledger, daily credit
  budget ledger, Telegram send, digest formatting. (Hermes does the vidIQ MCP calls + writes
  the angles; the helper never touches vidIQ.)
- `config.json` — lanes (query + audienceQuery + angle hint per lane), Telegram creds
  (reuse souls-clipper token), `daily_credit_budget`, `results_per_lane`, `published_within`.

## Architecture (per scheduled run)
1. **Preflight (safety):** Hermes calls `vidiq_balance` (0 cr). If the tool is unavailable →
   STOP, Telegram Hari: "Trend Scout can't run — I (Midir) don't have the vidIQ connector.
   Wire it, or switch this to the Claude Code engine." This is the guard for the chosen-engine risk.
2. **Budget gate:** `python3 trendscout.py budget --check` → remaining credits today. If below
   one lane's cost, skip to step 5 with whatever's affordable.
3. **Per lane** (Dark Souls, AI, Merl), cheapest-useful call set:
   - Primary: `vidiq_instagram_tiktok_outlier_search` (embeddingType `format`, the lane's
     `query` + `audienceQuery`, `resultsPerPlatform` from config) — TikTok/IG is where Hari
     posts, so this is the core surface.
   - Optional (budget permitting, ~1/run rotated across lanes): `vidiq_outliers` (YouTube,
     `keyword`=lane, `contentType:short`, `publishedWithin:thisWeek`) for cross-platform confirmation.
   - After each call: `trendscout.py budget --spend 5` records the debit.
4. **Dedup + angle:** pipe the surfaced items' IDs through `trendscout.py filter-new`
   (drops anything already sent on a prior day, records the rest). For each *new* format,
   Hermes (GPT-5.6, already in-loop) writes: the FORMAT/pattern name, why it's popping
   (outlier score + views), the example link, and **Hari's angle** — how to apply that format
   to the lane, using the lane's `angle_hint` (e.g. "cut an 'Everything Wrong With Ornstein &
   Smough' from the DS1 VOD bank"; "point this POV format at Merl").
5. **Deliver:** `trendscout.py send` posts the formatted digest to Telegram (grouped by lane;
   header notes AM/PM + credits spent/remaining). Nothing if no new formats.
6. Ideas also appended to `~/content-radar/briefs/trends-<date>.jsonl` so the `/news-video`
   + phone-farm pipeline can pick a format to actually produce.

## Message shape (locked)
```
🎯 Trend Scout — AM · 2026-07-12   (spent 20cr · 40 left today)
Formats popping in your lanes — ride these:

🗡️ Dark Souls / soulslike
• "Everything Wrong With [boss]" — 8.2x outlier, 1.2M views · <link>
  ↳ Angle: cut "Everything wrong with Ornstein & Smough" from your VOD bank
• rage cold-open → silent clutch → cam pop — 5.1x · <link>
  ↳ Angle: lead your next death-clip with the cam reaction, not the death

🤖 AI news / tools
• "I replaced my job with [tool]" POV — 6.4x · <link>
  ↳ Angle: "I replaced my content team with a Mac Studio" (your local stack)

📱 Merl / AI-app promo
• before/after screen-record + trend audio — 4.7x · <link>
  ↳ Angle: Merl blank-canvas → finished cast, 7s, trend audio
```

## Key files / tools to reuse
- `~/content-scout.skill.md` — same install/exportable pattern, Telegram Bot API send, seen.jsonl
  dedup, config-placeholder style. Reuse verbatim structure.
- vidIQ MCP: `vidiq_instagram_tiktok_outlier_search`, `vidiq_outliers`, `vidiq_trending_videos`,
  `vidiq_trend_categories` (0cr), `vidiq_balance` (0cr), `vidiq_user_channels` (0cr).
- souls-clipper `send_clips.py` — existing Telegram bot token to reuse.
- `~/content-radar/briefs/` — downstream handoff target.

## Verification (on approval, before shipping)
1. Live vidIQ query test from THIS session (costs ~10–15 cr, within the twice-daily budget
   Hari approved): run `vidiq_instagram_tiktok_outlier_search` for each of the 3 lanes; confirm
   they return real, on-lane outliers with usable format signals. Tune the `query`/`audienceQuery`
   until the results are genuinely relevant (esp. Dark Souls). This validates the core data layer.
2. Helper unit test in scratchpad: `trendscout.py filter-new` dedups across runs;
   `budget --spend/--check` enforces the daily cap and rolls over by date; `send` dry-run
   prints the formatted digest without posting. Confirm syntax + JSON configs parse.
3. Ship `~/trend-scout.skill.md`; on install Hermes runs the preflight and (if vidIQ present)
   an AM dry digest for Hari to eyeball before the cron goes live.

## Guardrails
- Hard `daily_credit_budget` (60) enforced by the helper ledger — the skill must budget-gate
  every vidIQ call so a loop can't drain the balance. Log spend each run.
- Preflight fails safe if Hermes lacks vidIQ (don't silently spend / don't crash the cron).
- Feeds Hari only; never auto-posts. Secrets in config.json, not committed.
- Two Hermes crons (~8:00, ~18:00) like warm-saga/warmup-overseer; local-only, no Discord.
- If the connector turns out not to be in Hermes, fallback is the Claude Code scheduled engine
  (same helper + message format, different runner) — noted so the pivot is cheap.
```
```
