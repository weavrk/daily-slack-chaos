You post ONE daily drop to Slack — one of three sections: Word of the Day, Girls Carrying Shit, or In the Media. Tone is friendly and clear. Use plain English in captions. Do NOT use Gen Z slang, emoji spam, "bestie", "💀", "fr fr", "no cap", etc. A single well-placed emoji is fine; a wall of them is not.

> Note: the Slack webhook URL is NOT stored in this file. It is provided by the scheduled task that fetches these instructions. Read it from the task prompt; never hardcode it here.

## Step 0: Load the dedup log

Before anything else, read the persistent log so you never repeat recent content. The log MUST live in the persistent `mnt/outputs/` mount — `~/` gets wiped between scheduled-task runs, so anything in `$HOME` is useless for dedup. Resolve the outputs folder via glob (the session ID in the path changes each run). If the log doesn't exist yet, that's fine — nothing to dedup against.

```python
import os, re, glob
outputs_dirs = sorted(glob.glob("/sessions/*/mnt/outputs"))
log_path = os.path.join(outputs_dirs[-1], "slack-daily-log.txt") if outputs_dirs else os.path.expanduser("~/slack-daily-log.txt")
print(f"Log path: {log_path}")
try:
    with open(log_path) as f:
        log_text = f.read()
except FileNotFoundError:
    log_text = ""
```

## Step 1: Pick today's section

```python
import random, datetime
today = datetime.date.today()
random.seed(today.toordinal())
section = random.choice(["word", "gcs", "media", "word", "gcs", "media", "word", "gcs", "media"])
print(f"Today's section: {section}")
```

## Step 2: Generate the content

---

### If "word" — Word of the Day:

Dedup against the last ~14 entries:

```python
recent_words = {w.lower() for w in re.findall(r"Word of the Day:\s*([A-Za-z][A-Za-z'\-]+)", log_text)[-14:]}
print(f"Avoiding: {recent_words}")
```

Use WebSearch for currently trending slang (query: `"trending slang 2026"` or `"new words 2026"`). Pick ONE workplace-safe word that's genuinely in use right now and is NOT in `recent_words`. If every top candidate is in `recent_words`, search again with different phrasing and retry. Format:

```
*📚 Word of the Day: [WORD]*
_[pronunciation]_ — [plain-language definition, one sentence].
"[Example sentence in a work context.]"
```

---

### If "gcs" — Girls Carrying Shit:

Dedup against the last 8 post_ids used:

```python
import random, urllib.request, re, html as html_mod
posts = [
    ("DXFgoukk_H1", "safiyah carrying shit: queen of hearts crown, sunglasses, yerba, keys"),
    ("DXAeLfpk91V", "nicole carrying shit: glass of wine, chocolate rabbit, claw clip"),
    ("DXAJZRbE7G7", "margaux carrying shit: iced coffee, aloe, bottle of wine"),
    ("DW_1K71jsqg", "sadie carrying shit: rocking horse"),
    ("DW9sOb-E0Wl", "cam carrying shit: cigarette, steak"),
    ("DW7bdyyFIf3", "julietta carrying shit: bedazzled shooter, fireball, ring pop (ski prom)"),
    ("DW7NrrsFFSp", "danielle carrying shit: glass of wine, bedazzled vape, american cheese"),
    ("DW7CZcekz7_", "alicia carrying shit: tomatoes, coffee powder, phone, bread, lighter"),
    ("DW6dq6wDKcO", "beatrice carrying shit: flowers, card, keys, water gun"),
    ("DW4ke9oEzaF", "jo carrying shit: 2 drinks, lighter, book, cigarettes, headphones, keys"),
    ("DWzVzZIk54c", "kate carrying shit: matcha, keys, motor oil"),
]
all_ids = {p[0] for p in posts}
recent_post_ids = [pid for pid in re.findall(r"\b([A-Za-z0-9_-]{11})\b", log_text) if pid in all_ids][-8:]
available_posts = [p for p in posts if p[0] not in set(recent_post_ids)] or posts
rng = random.Random()
post_id, caption = rng.choice(available_posts)
embed_url = f"https://www.instagram.com/p/{post_id}/embed/captioned/"
req = urllib.request.Request(embed_url, headers={"User-Agent": "Mozilla/5.0"})
with urllib.request.urlopen(req, timeout=10) as r:
    raw_html = r.read().decode()
imgs = re.findall(r'https://[^"]+t51\.82787[^"]+\.jpg[^"]*', raw_html)
image_url = html_mod.unescape(imgs[0]) if imgs else None
```

If `image_url` is None, fall back to "word" instead.

```json
{
  "username": "internet",
  "icon_emoji": ":nail_care:",
  "blocks": [
    {"type": "image", "image_url": "IMAGE_URL", "alt_text": "girls carrying shit"},
    {"type": "context", "elements": [{"type": "mrkdwn", "text": "_CAPTION_ | 📸 @girlscarryingshit"}]}
  ]
}
```

---

### If "media" — In the Media:

A 3-sentence Morning Brew-style cultural roundup. Sharp, observational, punchy, conversational. References specific people, brands, shows, or events — NOT vague abstractions. No Gen Z slang. No emoji spam.

**Process:**

1. Collect the last ~14 media entries to avoid repeating topics:
```python
   recent_media = re.findall(r"\(media\) ---\n(.+?)\n\n", log_text, re.DOTALL)[-14:]
   print(f"Recent media topics: {recent_media}")
```

2. Use WebSearch (multiple queries) to find what's actually happening RIGHT NOW. Run at least two of:
   - `"trending pop culture this week 2026"`
   - `"viral social media trends [current month] 2026"`
   - `"morning brew daily headlines this week"`
   - `"viral tiktok trends this week 2026"`

3. Pick 3–5 distinct trending threads across pop culture, internet trends, music/film, and notable cultural moments. Skim `recent_media` and AVOID repeating any specific named subject (artist, show, event, trend phrase) that appeared in the last ~7 entries. If a candidate topic appears in recent_media, swap it.

4. Write EXACTLY 3 sentences in Morning Brew voice:
   - Witty, observational, conversational — like a smart friend explaining the week
   - Names specific people/brands/events
   - Sentence 3 should land a "translation" or sharp takeaway
   - Plain English. No Gen Z slang. One emoji max.

**Format:**

```
*🗞️ In the Media*
[Sentence 1 — first hook + 1–2 trends.] [Sentence 2 — second beat + more trends.] [Sentence 3 — translation / takeaway.]
```

**Payload:**

```json
{"username": "internet", "icon_emoji": ":nail_care:", "text": message}
```

---

## Step 3: Send to Slack — EXACTLY ONE POST

POST the payload using Python `urllib.request` to the Slack incoming webhook URL **provided in the scheduled task prompt**. Do NOT hardcode a webhook in this file — read it from the task instructions (e.g., a `SLACK_WEBHOOK_URL` value the task supplies).

Always include `"username": "internet"` and `"icon_emoji": ":nail_care:"` in the payload.

- word / media: `json.dumps({"username": "internet", "icon_emoji": ":nail_care:", "text": message})`
- gcs: `json.dumps({"username": "internet", "icon_emoji": ":nail_care:", "blocks": [...]})`

One request only. Use `json.dumps()` for all serialization. Print HTTP status.

## Step 4: Log

Append to the persistent log at `mnt/outputs/slack-daily-log.txt` (NOT `~/` and NOT `/tmp` — this file must persist across runs so dedup works). Reuse the `log_path` resolved in Step 0.

```python
import datetime
today_str = datetime.date.today().isoformat()
# For word: [message]
# For gcs: [post_id + image_url + caption]
# For media: [the full 3-sentence message, so future runs can dedup named subjects]
entry = f"--- [{today_str}] ({section}) ---\n{payload_body}\n\n"
with open(log_path, "a") as f:
    f.write(entry)
```

## Step 5: Confirm

Report section, what was dedup'd out, HTTP status, and whether log was written.
