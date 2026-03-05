# /viral:discover — Multi-Platform Topic Discovery

You are running the Viral Command discovery engine. Your primary job is to analyze competitor content performance, then optionally discover trending topics — all scored against the creator's ICP.

**Arguments:** $ARGUMENTS

---

## Phase A: Load Agent Brain

Read the agent brain to understand who the creator is:

```
@data/agent-brain.json
```

**Extract these fields:**
- `identity` — who the creator is, niche, tone
- `competitors[]` — competitor list with platform + handle
- `icp.segments[]`, `icp.pain_points[]`, `icp.goals[]` — scoring context
- `pillars[]` — content themes + keywords
- `learning_weights` — scoring multipliers
- `platforms.api_keys_configured` — available APIs

**If the brain is empty or `identity.name` is blank:**
Stop and say: *"Your agent brain isn't set up yet. Run `/viral:onboard` first to tell me who you are and what you create."*

---

## Phase 1: Competitor Analysis (Core)

This is the main event. Pull and rank competitor **longform** content from the last 30 days.

**IMPORTANT: Longform only.** Filter out YouTube Shorts and any video under 5 minutes. We only care about longform content (5+ minutes) for competitor analysis. Shorts have different dynamics and aren't useful for longform content strategy.

### Step 1: Pull YouTube Competitor Content

For each competitor where `platform == "YouTube"`:

1. **Get channel ID from handle** using YouTube Data API:
   ```bash
   # Load API key from .env
   source "$(pwd)/.env" 2>/dev/null || source "$(dirname "$(pwd)")/.env" 2>/dev/null

   # Search for channel by handle
   curl -s "https://www.googleapis.com/youtube/v3/search?part=snippet&q=${HANDLE}&type=channel&key=${YOUTUBE_API_KEY}"
   ```
   Extract the `channelId` from the response.

2. **Pull recent videos** (last 30 days):
   ```bash
   # Get date 30 days ago in RFC3339 format
   PUBLISHED_AFTER=$(date -u -v-30d '+%Y-%m-%dT%H:%M:%SZ' 2>/dev/null || date -u -d '30 days ago' '+%Y-%m-%dT%H:%M:%SZ')

   curl -s "https://www.googleapis.com/youtube/v3/search?part=snippet&channelId=${CHANNEL_ID}&order=date&publishedAfter=${PUBLISHED_AFTER}&maxResults=50&type=video&key=${YOUTUBE_API_KEY}"
   ```

3. **Get video statistics + duration** (views, likes, comments, contentDetails):
   ```bash
   # Comma-separated video IDs from step 2
   curl -s "https://www.googleapis.com/youtube/v3/videos?part=statistics,contentDetails&id=${VIDEO_IDS}&key=${YOUTUBE_API_KEY}"
   ```

4. **Filter to longform only** — parse `contentDetails.duration` (ISO 8601 format like `PT11M8S`) and **discard any video under 5 minutes**. YouTube Shorts and short clips are noise for this analysis.

5. **Calculate engagement rate** for each remaining video:
   ```
   engagement_rate = (likes + comments) / views × 100
   ```

### Step 2: Pull Instagram Competitor Content

For each competitor where `platform == "Instagram"`:

1. **Scrape recent posts/reels** using instaloader:
   ```bash
   # Create temp directory for scrape
   mkdir -p /tmp/viral-discover-ig

   # Pull posts from last 30 days (metadata only — no media download)
   # IMPORTANT: Use python3 -m instaloader (not bare instaloader) for PATH compatibility
   python3 -m instaloader --no-pictures --no-videos --no-video-thumbnails --post-metadata-txt="" --dirname-pattern="/tmp/viral-discover-ig/{profile}" -- ${HANDLE}
   ```

   **Note:** If instaloader hits 403 rate limits, it auto-retries — that's normal. If it fully fails or requires login, inform the user and skip that competitor. Don't block the whole discovery on Instagram failures.

2. **Parse the scraped metadata** for each post:
   - Caption text (this is the "title" equivalent)
   - Likes count
   - Comments count
   - Post date
   - Post type (reel, carousel, single image)
   - **URL** — build from shortcode: `https://www.instagram.com/reel/{shortcode}/` for reels, `https://www.instagram.com/p/{shortcode}/` for posts. The shortcode is in the JSON filename or metadata.

3. **Calculate engagement rate:**
   ```
   engagement_rate = (likes + comments) / followers × 100
   ```
   (If follower count unavailable, use likes + comments as raw engagement)

### Step 3: Rank All Content

Combine YouTube and Instagram results into a single ranked list:

1. **Normalize engagement** across platforms (YouTube views-based, Instagram followers-based)
2. **Sort by raw engagement** (total likes + comments + views) as primary sort
3. **Display the TOP 10 performing pieces** across all competitors:

```
═══════════════════════════════════════════════════
COMPETITOR CONTENT — TOP 10 (Last 30 Days)
═══════════════════════════════════════════════════

 #  │ Platform  │ Competitor       │ Title/Caption                    │ Views/Likes  │ Eng%
────┼───────────┼──────────────────┼──────────────────────────────────┼──────────────┼──────
 1  │ YouTube   │ Chase Hannegan   │ {title}                          │ {views}      │ {rate}%
    │           │                  │ https://youtube.com/watch?v=...  │              │
────┼───────────┼──────────────────┼──────────────────────────────────┼──────────────┼──────
 2  │ Instagram │ Cooper Simson    │ {caption first 50 chars}...      │ {likes}      │ {rate}%
    │           │                  │ https://instagram.com/reel/...   │              │
────┼───────────┼──────────────────┼──────────────────────────────────┼──────────────┼──────
...

═══════════════════════════════════════════════════
```

**Every entry must include the direct URL** — YouTube: `https://youtube.com/watch?v={VIDEO_ID}`, Instagram: `https://www.instagram.com/reel/{SHORTCODE}/` or `https://www.instagram.com/p/{SHORTCODE}/`

### Step 4: Interactive Transcription

After showing the top 10, ask the user:

```
Want to transcribe any of these for deeper analysis?
Enter numbers (e.g., "1,3,5") or "skip" to continue without transcription:
```

**For each selected video/reel:**

**YouTube transcription:**
```bash
# Download audio
yt-dlp -x --audio-format mp3 -o "/tmp/viral-discover-yt/%(id)s.%(ext)s" "https://youtube.com/watch?v=${VIDEO_ID}"

# Transcribe with OpenAI Whisper API (preferred — faster, more reliable)
curl -s https://api.openai.com/v1/audio/transcriptions \
  -H "Authorization: Bearer ${OPENAI_API_KEY}" \
  -F file="@/tmp/viral-discover-yt/${VIDEO_ID}.mp3" \
  -F model="whisper-1" \
  -F response_format="text"
```

If OpenAI API unavailable, fall back to local whisper:
```bash
whisper "/tmp/viral-discover-yt/${VIDEO_ID}.mp3" --model base --output_format txt --output_dir /tmp/viral-discover-yt/
```

**Instagram transcription:**
```bash
# Download reel video
python3 -m instaloader --dirname-pattern="/tmp/viral-discover-ig/{profile}" -- -${SHORTCODE}

# Extract audio and transcribe (same as YouTube)
ffmpeg -i "/tmp/viral-discover-ig/${HANDLE}/${POST_FILE}" -vn -acodec libmp3lame "/tmp/viral-discover-ig/${HANDLE}/${SHORTCODE}.mp3"

# Then use OpenAI Whisper API or local whisper (same as above)
```

### Step 5: Dissect Transcripts

For each transcribed piece, produce a breakdown:

```
═══════════════════════════════════════════════════
CONTENT BREAKDOWN: {title}
By: {competitor} ({platform})
Views: {views} | Engagement: {rate}%
═══════════════════════════════════════════════════

HOOK (first 3 seconds / first line):
"{exact opening words}"
→ Hook type: {contradiction/specificity/pattern interrupt/etc.}
→ Why it works: {1-2 sentences}

STRUCTURE:
1. Hook (0:00-0:03): {what happens}
2. Setup (0:03-0:15): {what happens}
3. Body (0:15-0:45): {what happens}
4. CTA (0:45-end): {what happens}

ANGLES USED:
- {angle 1}: {how it's applied}
- {angle 2}: {how it's applied}

WHY IT WORKED:
- {engagement signal 1}
- {engagement signal 2}
- {what Charles could learn from this}

═══════════════════════════════════════════════════
```

---

## Phase 2: Trend Discovery (Optional)

After the competitor report, ask:

```
Competitor analysis complete. Want to also run trend discovery queries?
These will be based on patterns from competitor content — not generic pillar searches.

(yes/no):
```

**If no:** Skip to Phase 3 (scoring + save).

**If yes:**

### Step 1: Suggest Queries Based on Competitor Patterns

Analyze the competitor content from Phase 1 to identify trending themes, then suggest 2-3 search queries:

```
Based on competitor content, here are suggested trend queries:
═══════════════════════════════════════════════════
1. "{query}" — because {competitors} are getting {views} on {theme}
2. "{query}" — because {pattern observed across competitors}
3. "{query}" — gap spotted: {competitors NOT covering this ICP-relevant topic}
═══════════════════════════════════════════════════

Edit, remove, or approve these queries (or type your own):
```

**Wait for user confirmation before running any queries.** The user can edit, remove, or add queries.

### Step 2: Run Approved Queries

For each approved query, execute the last30days research script:

```bash
python3 skills/last30days/scripts/last30days.py "{query}" --emit=compact --quick
```

**Rules:**
- Maximum 3 queries per run
- Run each in foreground with 5-minute timeout
- If `--deep` flag was passed in arguments: use `--deep` instead of `--quick`
- If a query fails or times out, log the error and continue

### Step 3: Capture Trend Topics

From last30days results, identify unique topics worth covering. Each topic should be a coherent content idea, not just a keyword.

---

## Phase 3: Score All Topics

Score ALL topics — both competitor-sourced and trend-discovered — against the ICP.

### Topic Extraction

**From competitor content (Phase 1):**
- Each high-performing competitor piece = a potential topic
- Extract the core content idea (not just the title)
- Note the competitor validation (who made it, how it performed)

**From trend discovery (Phase 2, if run):**
- Each unique trend = a potential topic
- Note the source platform and engagement signals

### Scoring Criteria (each 1-10):

| Criterion | What to evaluate | Weight key |
|-----------|-----------------|------------|
| **icp_relevance** | Does this directly address the ICP's pain points, goals, or interests? | `learning_weights.icp_relevance` |
| **timeliness** | Is this trending NOW? New release, breaking news, viral moment? | `learning_weights.timeliness` |
| **content_gap** | Has Charles covered this? Is there room for a fresh angle? | `learning_weights.content_gap` |
| **proof_potential** | Can Charles SHOW something on screen? Demo, tutorial, walkthrough? | `learning_weights.proof_potential` |

### Calculate scores:

```
weighted_total = (icp_relevance × learning_weights.icp_relevance) +
                 (timeliness × learning_weights.timeliness) +
                 (content_gap × learning_weights.content_gap) +
                 (proof_potential × learning_weights.proof_potential)
```

### Competitor validation bonuses:
- Topics validated by competitor performance (>50K views) get **+2 content_gap** bonus
- Topics from top 3 competitor pieces get **+1 proof_potential** bonus

### Selection:
- Deduplicate: merge same topic from multiple sources
- Minimum threshold: weighted_total ≥ 20 (out of 40)
- If fewer than 5 topics meet threshold, lower to 15 and note it
- Target: 10-20 scored topics per run

---

## Phase 4: Save to JSONL

Write discovered topics to a date-stamped JSONL file:

**File path:** `data/topics/{YYYY-MM-DD}-topics.jsonl`

**One JSON object per line:**

```json
{
  "id": "topic_{YYYYMMDD}_{NNN}",
  "title": "Topic title — clear, specific, content-ready",
  "description": "Why this topic matters and why it's trending",
  "source": {
    "platform": "youtube|instagram|reddit|x|web|hackernews",
    "url": "https://source-url",
    "author": "Original author / competitor name",
    "engagement_signals": "500K views, 15K likes, 3.2% engagement"
  },
  "discovered_at": "2026-03-05T12:00:00Z",
  "scoring": {
    "icp_relevance": 8,
    "timeliness": 9,
    "content_gap": 7,
    "proof_potential": 8,
    "total": 32,
    "weighted_total": 32.0
  },
  "pillars": ["AI Automation"],
  "competitor_coverage": [
    {"name": "Chase Hannegan", "platform": "YouTube", "views": 150000, "url": "https://..."}
  ],
  "status": "new",
  "notes": ""
}
```

**Rules:**
- IDs: `topic_{YYYYMMDD}_{NNN}` with zero-padded 3-digit sequence
- Set `status: "new"` for all freshly discovered topics
- Set `discovered_at` to current ISO timestamp
- If file exists for today, **append** (no duplicates — check existing IDs)
- `competitor_coverage` should be populated for competitor-sourced topics

---

## Phase 5: Output Summary

```
═══════════════════════════════════════════════════
DISCOVERY COMPLETE
═══════════════════════════════════════════════════

Competitors analyzed: {N}
  YouTube: {list handles} — {N} videos pulled
  Instagram: {list handles} — {N} posts pulled
Transcriptions done: {N}
Trend queries run: {N} (or "skipped")
Total topics scored: {N} → {N saved} (after dedup + threshold)
Saved to: data/topics/{YYYY-MM-DD}-topics.jsonl

TOP TOPICS BY SCORE
───────────────────────────────────────────────────

 #  │ Score │ Topic                                    │ Source
────┼───────┼──────────────────────────────────────────┼──────────────
 1  │ 35.0  │ {topic title}                            │ {competitor or trend}
    │       │ ICP: 9  Time: 9  Gap: 8  Proof: 9       │
    │       │ Why: {one line on why this scored high}   │
────┼───────┼──────────────────────────────────────────┼──────────────
 2  │ 33.0  │ {topic title}                            │ {source}
    │       │ ICP: 8  Time: 9  Gap: 8  Proof: 8       │
    │       │ Why: {one line}                           │
────┼───────┼──────────────────────────────────────────┼──────────────
...

COMPETITOR INSIGHTS
───────────────────────────────────────────────────
What's working for competitors right now:
- {competitor}: {pattern/theme} ({engagement stats})
- {competitor}: {pattern/theme} ({engagement stats})

Content gaps (competitors NOT covering):
- {gap 1} — relevant to Charles's ICP because {reason}
- {gap 2} — {reason}
───────────────────────────────────────────────────

RECOMMENDED NEXT STEPS
───────────────────────────────────────────────────

Develop these topics next with /viral:angle:

1. "{top topic}" — {reason it's the best pick}
2. "{second topic}" — {reason}
3. "{third topic}" — {reason}

Run: /viral:angle "{topic title}" to develop angles.

═══════════════════════════════════════════════════
```

Show top 10 topics maximum in the summary.

---

## Caching

To avoid re-scraping within the same day:

- **Cache directory:** `data/recon/cache/`
- **Cache files:** `{handle}_{YYYY-MM-DD}.json` for each competitor
- Before pulling competitor data, check if today's cache exists
- If cache hit: load from cache instead of API calls
- Cache includes: video/post list, statistics, engagement rates

---

## Important Rules

- **NEVER modify `data/agent-brain.json`** — discover is read-only on the brain
- **Competitor analysis is the core** — trend discovery is optional and secondary
- **No hardcoded queries** — all trend queries are suggested based on competitor patterns, user approves before running
- **Interactive transcription** — always ask which videos to transcribe, never auto-transcribe all
- **Always score against ICP** — topics without ICP relevance scoring are useless
- **Save before displaying** — write JSONL first, then show summary
- **Idempotent for same day** — running twice appends without duplicates (check existing IDs)
- **Graceful degradation** — if Instagram scraping fails, continue with YouTube only. If no API keys, explain what's missing.
- **Clean up temp files** after transcription (`/tmp/viral-discover-*`)
- **Maximum 3 trend queries** per run (API rate limits)
