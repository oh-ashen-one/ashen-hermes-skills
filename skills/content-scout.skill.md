---
name: content-scout
description: Always-on AI-news → content-idea firehose. Polls Google News, r/LocalLLaMA, r/singularity, HuggingFace trending+papers, and Hacker News every ~12 min; DMs Hari on Telegram a link + gist + TikTok hook + long-form angle for anything that breaks. BREAKING drops ping instantly; the rest arrive as a rolled-up radar batch.
---

# Content Scout — INSTALL INSTRUCTIONS FOR MIDIR (Hermes)

Midir: when Hari texts you this file, INSTALL IT AS A SKILL on the Studio. Do exactly this:

1. `mkdir -p ~/.hermes/skills/content-scout/state`
2. Save the three files below verbatim into `~/.hermes/skills/content-scout/`:
   `scout.py`, `sources.json`, `config.json`.
3. In `config.json`, set `telegram_chat_id` to Hari's chat id and `telegram_bot_token` to
   the souls-clipper bot token (reuse it) — ask Hari only if you can't find either.
   Leave `enrich_endpoint` pointed at the local LM Studio unless Hari says otherwise.
4. Prime + test: `cd ~/.hermes/skills/content-scout && python3 scout.py --dry-run`
   then a real prime run `python3 scout.py`. Text Hari the "online" confirmation.
5. Register a cron every 12 minutes:
   `cd ~/.hermes/skills/content-scout && python3 scout.py`
   (use the same cron mechanism as warm-saga / warmup-overseer). Local-only, no Discord.
6. Remove the retired predecessor if present: delete `~/.hermes/skills/ai-news-watcher`
   and any `news-watcher` cron.
7. Ongoing: this is a background news feed for HARI ONLY. It never auto-posts to socials.

## config.json
```json
{
  "telegram_bot_token": "REUSE_SOULS_CLIPPER_BOT_TOKEN",
  "telegram_chat_id": "HARI_TELEGRAM_CHAT_ID",
  "enrich_endpoint": "http://localhost:1234/v1/chat/completions",
  "enrich_model": "glm-5.2",
  "enrich_api_key": "lm-studio",
  "cadence_minutes": 12,
  "recency_hours": 4,
  "hn_min_points": 40,
  "content_radar_briefs": "~/content-radar/briefs",
  "max_breaking_per_cycle": 4,
  "max_radar_per_cycle": 8
}
```

## sources.json
```json
{
  "google_news_keywords": [
    "OpenAI GPT", "Anthropic Claude", "Google Gemini", "Meta Llama", "xAI Grok",
    "Mistral AI", "DeepSeek", "Qwen Alibaba", "Moonshot Kimi", "Zhipu GLM",
    "MiniMax AI", "Baidu Ernie", "ByteDance Doubao", "Tencent Hunyuan", "01.AI Yi",
    "open source LLM", "local LLM", "AI model release", "AI funding round",
    "Cursor AI", "Claude Code", "AI agent", "open weights model"
  ],
  "reddit": ["LocalLLaMA", "singularity"],
  "hn_terms": ["ai","llm","gpt","claude","gemini","llama","model","openai","anthropic","deepseek","qwen","mistral","agent","inference","open source","fine-tune"],
  "signal_words": ["release","released","launch","launches","drops","ships","unveils","introduces","announces","open weights","open-source","open source","weights","benchmark","sota","state-of-the-art","beats","surpasses","raises","funding","valuation","acquires","acquisition","api","fine-tune","quantized","gguf","llama.cpp","context window","multimodal","reasoning","outage","banned","lawsuit","partnership","available"],
  "breaking_words": ["release","released","launch","launches","drops","ships","unveils","introduces","open weights","raises","funding","acquires","acquisition","sota","beats","surpasses"],
  "lab_names": ["openai","anthropic","google","deepmind","meta","xai","mistral","deepseek","qwen","alibaba","moonshot","kimi","zhipu","glm","minimax","baidu","ernie","bytedance","doubao","tencent","hunyuan","microsoft","nvidia","01.ai","yi"]
}
```

## scout.py
```python
#!/usr/bin/env python3
"""Content Scout: always-on AI-news -> content-idea firehose for Telegram.
Stdlib only. Poll -> dedup -> classify BREAKING/RADAR -> enrich (local model) -> Telegram.
Usage: python3 scout.py [--dry-run] [--loop]"""
import json, os, re, sys, time, hashlib, urllib.request, urllib.parse
from datetime import datetime, timezone

HERE = os.path.dirname(os.path.abspath(__file__))
def load_json(n): return json.load(open(os.path.join(HERE, n), encoding="utf-8"))
CFG = load_json("config.json"); SRC = load_json("sources.json")
STATE = os.path.join(HERE, "state"); os.makedirs(STATE, exist_ok=True)
SEEN_FILE = os.path.join(STATE, "seen.jsonl"); LOG_FILE = os.path.join(STATE, "scout.log")
DRY = "--dry-run" in sys.argv
UA = {"User-Agent": "Mozilla/5.0 (compatible; content-scout/1.0)"}

def log(m):
    line = datetime.now().isoformat(timespec="seconds") + " " + m
    print(line)
    try: open(LOG_FILE, "a", encoding="utf-8").write(line + "\n")
    except Exception: pass

def http(url, timeout=15, data=None, headers=None):
    h = dict(UA)
    if headers: h.update(headers)
    req = urllib.request.Request(url, data=data, headers=h)
    with urllib.request.urlopen(req, timeout=timeout) as r:
        return r.read().decode("utf-8", "replace")

def clean(s):
    s = re.sub(r"<!\[CDATA\[|\]\]>", "", s or ""); s = re.sub(r"<[^>]+>", "", s)
    for a, b in (("&amp;","&"),("&#39;","'"),("&#x27;","'"),("&quot;",'"'),("&lt;","<"),("&gt;",">")):
        s = s.replace(a, b)
    return re.sub(r"\s+", " ", s).strip()

def sid(url, title): return hashlib.sha1((url or title).encode("utf-8")).hexdigest()[:16]

def load_seen():
    ids = set()
    if os.path.exists(SEEN_FILE):
        for ln in open(SEEN_FILE, encoding="utf-8"):
            ln = ln.strip()
            if ln:
                try: ids.add(json.loads(ln)["id"])
                except Exception: pass
    return ids

def save_seen(ids):
    with open(SEEN_FILE, "a", encoding="utf-8") as f:
        for i in ids: f.write(json.dumps({"id": i, "t": int(time.time())}) + "\n")
    try:
        lines = open(SEEN_FILE, encoding="utf-8").read().splitlines()
        if len(lines) > 3000:
            open(SEEN_FILE, "w", encoding="utf-8").write("\n".join(lines[-3000:]) + "\n")
    except Exception: pass

def age_hours(pub):
    if not pub: return 0.0
    pub = pub.strip()
    for fmt in ("%a, %d %b %Y %H:%M:%S %z","%a, %d %b %Y %H:%M:%S %Z","%Y-%m-%dT%H:%M:%S%z","%Y-%m-%dT%H:%M:%SZ"):
        try:
            dt = datetime.strptime(pub, fmt)
            if dt.tzinfo is None: dt = dt.replace(tzinfo=timezone.utc)
            return (datetime.now(timezone.utc) - dt).total_seconds() / 3600
        except Exception: pass
    return 0.0

def parse_rss(xml, source):
    out = []; atom = False
    blocks = re.findall(r"<item\b[^>]*>(.*?)</item>", xml, re.S)
    if not blocks:
        blocks = re.findall(r"<entry\b[^>]*>(.*?)</entry>", xml, re.S); atom = True
    for b in blocks:
        tm = re.search(r"<title[^>]*>(.*?)</title>", b, re.S)
        title = clean(tm.group(1)) if tm else ""
        if not title: continue
        if atom:
            lm = re.search(r'<link[^>]*href="([^"]+)"', b)
            url = lm.group(1) if lm else ""
            pm = re.search(r"<(?:updated|published)>(.*?)</", b, re.S)
        else:
            lm = re.search(r"<link[^>]*>(.*?)</link>", b, re.S)
            url = clean(lm.group(1)) if lm else ""
            pm = re.search(r"<pubDate>(.*?)</pubDate>", b, re.S)
        pub = pm.group(1).strip() if pm else ""
        if source == "GoogleNews":
            title = re.sub(r"\s+-\s+[^-]+$", "", title).strip()
        out.append({"title": title, "url": url, "source": source, "pub": pub})
    return out

def fetch_google():
    items = []
    for kw in SRC.get("google_news_keywords", []):
        try:
            u = "https://news.google.com/rss/search?q=%s&hl=en-US&gl=US&ceid=US:en" % urllib.parse.quote(kw)
            items += parse_rss(http(u), "GoogleNews")
        except Exception as e: log("google fail %s: %s" % (kw, e))
        time.sleep(0.25)
    return items

def fetch_reddit():
    items = []
    for sub in SRC.get("reddit", []):
        for attempt in (1, 2):
            try:
                items += parse_rss(http("https://www.reddit.com/r/%s/new/.rss?limit=25" % sub), "r/" + sub)
                break
            except Exception as e:
                if attempt == 2: log("reddit fail %s: %s" % (sub, e))
                else: time.sleep(2.5)
        time.sleep(0.5)
    return items

def fetch_hn():
    items = []
    try:
        cutoff = int(time.time()) - CFG.get("recency_hours", 4) * 3600
        pts = CFG.get("hn_min_points", 40)
        u = "https://hn.algolia.com/api/v1/search_by_date?tags=story&numericFilters=points>%d,created_at_i>%d&hitsPerPage=50" % (pts, cutoff)
        terms = [t.lower() for t in SRC.get("hn_terms", [])]
        for h in json.loads(http(u)).get("hits", []):
            title = h.get("title") or ""
            if not title or not any(t in title.lower() for t in terms): continue
            url = h.get("url") or ("https://news.ycombinator.com/item?id=%s" % h.get("objectID"))
            items.append({"title": clean(title), "url": url, "source": "HN(%s)" % h.get("points"),
                          "pub": "", "_fresh": True, "_score": h.get("points", 0)})
    except Exception as e: log("hn fail: %s" % e)
    return items

def fetch_hf():
    items = []
    try:
        for m in json.loads(http("https://huggingface.co/api/models?sort=trendingScore&direction=-1&limit=30")):
            mid = m.get("id") or m.get("modelId") or ""
            if mid:
                items.append({"title": "Trending HF model: " + mid, "url": "https://huggingface.co/" + mid,
                              "source": "HuggingFace", "pub": "", "_fresh": True, "_hf": True})
    except Exception as e: log("hf models fail: %s" % e)
    try:
        for p in json.loads(http("https://huggingface.co/api/daily_papers"))[:15]:
            pp = p.get("paper", {}) or {}; t = pp.get("title") or p.get("title") or ""; pid = pp.get("id") or ""
            if t: items.append({"title": "Paper: " + clean(t), "url": "https://huggingface.co/papers/" + pid,
                                "source": "HFpapers", "pub": "", "_fresh": True})
    except Exception as e: log("hf papers fail: %s" % e)
    return items

def has_signal(title):
    low = title.lower(); return any(w in low for w in SRC.get("signal_words", []))

def fresh(it):
    if it.get("_fresh"): return True
    a = age_hours(it.get("pub", ""))
    return a <= CFG.get("recency_hours", 4) if a > 0 else True

def is_breaking(it):
    if it.get("_hf"): return True
    if it.get("_score", 0) >= CFG.get("hn_min_points", 40) * 2: return True
    low = it["title"].lower()
    if any(w in low for w in SRC.get("breaking_words", [])) and any(l in low for l in SRC.get("lab_names", [])):
        return True
    if re.search(r"\b(gpt|claude|gemini|llama|qwen|deepseek|kimi|glm|grok|mistral|yi|ernie|doubao|hunyuan)[- ]?\d", low):
        return True
    return False

def enrich(items):
    payload = [{"i": n, "title": it["title"], "source": it["source"]} for n, it in enumerate(items)]
    sys_p = ("You are a content scout for a creator who makes TikToks and long-form YouTube videos about AI. "
             "For each item return JSON with fields i, gist (<=18 words, concrete what-happened), "
             "tiktok (one punchy true hook line), longform (a specific video concept). "
             "Return ONLY a JSON array, no prose.")
    body = json.dumps({"model": CFG.get("enrich_model"), "temperature": 0.6,
                       "messages": [{"role": "system", "content": sys_p},
                                    {"role": "user", "content": json.dumps(payload)}]}).encode("utf-8")
    try:
        headers = {"Content-Type": "application/json"}
        if CFG.get("enrich_api_key"): headers["Authorization"] = "Bearer " + CFG["enrich_api_key"]
        raw = http(CFG.get("enrich_endpoint"), timeout=90, data=body, headers=headers)
        content = json.loads(raw)["choices"][0]["message"]["content"]
        content = re.sub(r"^```(?:json)?|```$", "", content.strip(), flags=re.M).strip()
        m = re.search(r"\[.*\]", content, re.S)
        arr = json.loads(m.group(0) if m else content)
        by = {int(x.get("i", n)): x for n, x in enumerate(arr)}
        for n, it in enumerate(items):
            x = by.get(n, {})
            it["gist"] = x.get("gist") or it["title"]; it["tiktok"] = x.get("tiktok") or ""; it["longform"] = x.get("longform") or ""
    except Exception as e:
        log("enrich fail (fallback to headline): %s" % e)
        for it in items: it["gist"] = it["title"]; it["tiktok"] = ""; it["longform"] = ""
    return items

def tg_send(text):
    if DRY: log("[DRY] would send:\n" + text + "\n"); return
    u = "https://api.telegram.org/bot%s/sendMessage" % CFG.get("telegram_bot_token")
    body = urllib.parse.urlencode({"chat_id": CFG.get("telegram_chat_id"), "text": text,
                                   "disable_web_page_preview": "false"}).encode()
    try: http(u, timeout=20, data=body, headers={"Content-Type": "application/x-www-form-urlencoded"})
    except Exception as e: log("telegram fail: %s" % e)

def fmt_breaking(it):
    lines = ["🚨 BREAKING", it["title"], it["url"], "Gist: " + it.get("gist", "")]
    if it.get("tiktok"): lines.append("🎬 TikTok: " + it["tiktok"])
    if it.get("longform"): lines.append("📹 Long-form: " + it["longform"])
    lines.append("↩︎ reply 🎬 to spin into a video")
    return "\n".join(lines)

def fmt_radar(items):
    rows = ["📡 AI Radar — " + datetime.now().strftime("%I:%M%p").lstrip("0")]
    for it in items:
        rows.append("• " + it["title"] + "\n" + it["url"] + "\n  ↳ " + (it.get("tiktok") or it.get("gist", "")))
    return "\n".join(rows)

def save_briefs(items):
    d = os.path.expanduser(CFG.get("content_radar_briefs", "") or "")
    if not d: return
    try:
        os.makedirs(d, exist_ok=True)
        f = os.path.join(d, "scout-%s.jsonl" % datetime.now().strftime("%Y-%m-%d"))
        with open(f, "a", encoding="utf-8") as fh:
            for it in items:
                fh.write(json.dumps({"ts": int(time.time()), "title": it["title"], "url": it["url"],
                    "source": it["source"], "tier": it["tier"], "gist": it.get("gist", ""),
                    "tiktok": it.get("tiktok", ""), "longform": it.get("longform", "")}) + "\n")
    except Exception as e: log("briefs fail: %s" % e)

def run_once():
    first_run = not os.path.exists(SEEN_FILE)
    seen = load_seen()
    raw = fetch_google() + fetch_reddit() + fetch_hn() + fetch_hf()
    new, ids = [], set()
    for it in raw:
        i = sid(it["url"], it["title"])
        if i in seen or i in ids: continue
        if not (it.get("_fresh") or fresh(it)): continue
        if not (it.get("_hf") or has_signal(it["title"])): continue
        it["id"] = i; ids.add(i); new.append(it)
    if first_run:
        save_seen([sid(x["url"], x["title"]) for x in raw])
        tg_send("🛰️ Content Scout online — primed %d stories, now watching AI news 24/7. "
                "You'll get 🚨 breaking pings + 📡 radar batches with TikTok + long-form angles." % len(raw))
        log("primed %d, no alerts on first run" % len(raw)); return
    if not new:
        log("no new items, skipped enrich"); return
    for it in new: it["tier"] = "BREAKING" if is_breaking(it) else "RADAR"
    breaking = [x for x in new if x["tier"] == "BREAKING"][:CFG.get("max_breaking_per_cycle", 4)]
    radar = [x for x in new if x["tier"] == "RADAR"][:CFG.get("max_radar_per_cycle", 8)]
    enrich(breaking + radar)
    for it in breaking: tg_send(fmt_breaking(it))
    if radar: tg_send(fmt_radar(radar))
    save_briefs(breaking + radar)
    # Only mark as seen what we actually delivered — anything over the per-cycle cap
    # stays unseen and surfaces next cycle, so nothing is silently dropped.
    save_seen([x["id"] for x in breaking + radar])
    log("sent %d breaking, %d radar (%d filtered this cycle)" % (len(breaking), len(radar), len(new)))

if __name__ == "__main__":
    if "--loop" in sys.argv:
        iv = CFG.get("cadence_minutes", 12) * 60
        while True:
            try: run_once()
            except Exception as e: log("cycle error: %s" % e)
            time.sleep(iv)
    else:
        run_once()
```

## What it does (for Hari)
- Every ~12 min: pulls Google News (all major US + Chinese labs), r/LocalLLaMA + r/singularity
  (where Chinese/local weights drop first), HuggingFace trending models + daily papers, and
  Hacker News stories over the points threshold.
- Plain-code dedup (`state/seen.jsonl`) + signal filter → **quiet cycles cost $0** (no model call).
- Only genuinely new items get one batched enrichment call to your **local GLM-5.2** (free);
  if it's down, you still get link + headline.
- 🚨 BREAKING (new model/version, big funding/acquisition, HF drop, hot HN) = instant ping.
  📡 Everything else = one rolled-up radar batch. Each carries a TikTok hook + long-form idea.
- Reply 🎬 to any item to hand it to `/news-video`. Ideas also saved to `~/content-radar/briefs/`.
- Tune keywords/sources in `sources.json`; cadence + thresholds in `config.json`. Add X/Twitter
  later by extending `sources.json` + a fetcher.
```
```
