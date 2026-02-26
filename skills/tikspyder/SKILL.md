---
name: tikspyder
description: Run the TikSpyder OSINT tool to collect TikTok data — search by keyword, username, or hashtag, download videos, extract keyframes, and export structured data. Use this skill whenever the user wants to collect TikTok data, scrape TikTok profiles or hashtags, search TikTok videos, download TikTok content, run tikspyder, or launch the TikSpyder Streamlit interface. Also trigger when the user mentions OSINT data collection from TikTok, even if they don't say "tikspyder" by name.
---

# TikSpyder — Guided TikTok Data Collection

You are guiding a user (who may not be technical) through collecting TikTok data using TikSpyder, a command-line OSINT tool. Your job is to set up the environment, configure credentials, gather search parameters conversationally, run the tool, and summarize results.

Throughout this process, communicate clearly and explain what you're doing at each step. If something fails, explain the error in plain language and suggest a fix.

---

## Phase 1: Locate TikSpyder & Environment Check

Before anything else, you need to find (or install) TikSpyder and verify the environment is ready. Work through these steps in order and track two key variables:

- **TIKSPYDER_DIR** — the root of the TikSpyder source repo (contains `main.py`, `config/`, etc.)
- **SKILL_DIR** — the directory where this skill file lives

### 1.1 Find the skill directory

The skill directory contains this SKILL.md file. Search for it:

```bash
# Check project-level and user-level skill locations
for candidate in .claude/skills/tikspyder "$HOME/.claude/skills/tikspyder"; do
  if [ -f "$candidate/SKILL.md" ]; then
    echo "SKILL_DIR=$candidate"
    break
  fi
done
```

Store this as SKILL_DIR for later use.

### 1.2 Check if tikspyder is already installed

Try these checks in order. Stop at the first one that works.

**Step A — Check current environment:**

```bash
pip show tikspyder 2>&1
```

If found, extract `Location:` and verify `main.py` exists there. Set TIKSPYDER_DIR accordingly.

**Step B — Check for conda/mamba environment named `tikspyder`:**

```bash
conda env list 2>/dev/null | grep tikspyder
```

If a `tikspyder` environment exists, activate it and check again. **Important:** In non-interactive shells (which is what you're running), `conda activate` won't work directly — you must initialize the shell hook first:

```bash
eval "$(conda shell.bash hook)" && conda activate tikspyder && pip show tikspyder 2>&1
```

Always use `eval "$(conda shell.bash hook)" && conda activate tikspyder` as the activation pattern — never bare `conda activate`. This applies everywhere in this skill: Phase 1, Phase 4, Phase 5.

If you find tikspyder in a conda env, use that environment for all subsequent commands. Remember to activate it (with the shell hook) before running tikspyder in Phase 4.

**Step C — Check if repo is already cloned in the skill directory:**

```bash
ls "$SKILL_DIR/tik-spyder/main.py" 2>/dev/null
```

If found, set `TIKSPYDER_DIR=$SKILL_DIR/tik-spyder`.

### 1.3 Clone and install if needed

If TikSpyder isn't found by any of the checks above, clone it from GitHub into the skill directory:

```bash
cd "$SKILL_DIR"
git clone https://github.com/estebanpdl/tik-spyder.git
```

Then set `TIKSPYDER_DIR=$SKILL_DIR/tik-spyder`.

### 1.4 Python version

```bash
python --version 2>&1 || python3 --version 2>&1
```

TikSpyder needs Python 3.11 or newer. If the version is too old, tell the user and stop.

### 1.5 Create environment and install (only if tikspyder wasn't found in 1.2)

If tikspyder is already installed (found in Step A or B), skip this entirely.

Otherwise, create an environment and install. First check whether conda/mamba is available:

```bash
conda --version 2>/dev/null || mamba --version 2>/dev/null
```

**If conda/mamba IS available**, use it:

```bash
eval "$(conda shell.bash hook)" && conda create -n tikspyder python=3.11 -y && conda activate tikspyder
cd "$TIKSPYDER_DIR" && pip install -e .
```

**If conda/mamba is NOT available**, fall back to Python's built-in venv. This is perfectly fine — venv ships with Python and needs no extra installation:

```bash
cd "$TIKSPYDER_DIR"
python -m venv .venv
```

Activate — the path differs by platform:

```bash
if [ -f "$TIKSPYDER_DIR/.venv/Scripts/activate" ]; then
  source "$TIKSPYDER_DIR/.venv/Scripts/activate"
else
  source "$TIKSPYDER_DIR/.venv/bin/activate"
fi
```

Then install:

```bash
cd "$TIKSPYDER_DIR" && pip install -e .
```

Verify it works regardless of which environment type was created:

```bash
tikspyder --help
```

Tell the user which environment type was set up (conda or venv) so they know for future reference.

### 1.6 Check ffmpeg

```bash
ffmpeg -version 2>&1 | head -1
```

ffmpeg is required for video processing and keyframe extraction. If missing, warn the user: "ffmpeg is not installed. Video downloads will still work, but keyframe extraction will fail. You can install it from https://ffmpeg.org/download.html."

### 1.7 Summary to user

Tell the user what you found — Python version, ffmpeg status, tikspyder installation status, where the repo lives — in a short summary. If everything is ready, move to Phase 2.

---

## Phase 2: API Key Configuration

TikSpyder uses two external APIs. Both are optional depending on what the user wants to do:

- **SerpAPI** — powers keyword searches (Google-based TikTok search). Needed for `--q` searches.
- **Apify** — powers direct TikTok profile and hashtag scraping. Needed for `--user` and `--tag` modes.

### 2.1 Check existing keys

Read the config file at `$TIKSPYDER_DIR/config/config.ini` and check whether both `api_key` and `apify_token` have non-empty values (not just the placeholder text).

**Security rules:**
- NEVER print, display, or echo API keys back to the user
- When reporting status, say "SerpAPI key: configured" or "Apify token: not configured" — never show the actual values
- When writing keys to the config file, use the Write tool directly — do not use echo/cat in bash where the key would appear in the command

### 2.2 Ask for missing keys

If either key is missing or empty, ask the user to provide it. Explain what each API is for:

- "**SerpAPI key** — This lets TikSpyder search Google for TikTok content. You can get one at https://serpapi.com/ (they have a free tier). You need this for keyword searches."
- "**Apify token** — This lets TikSpyder scrape TikTok profiles and hashtags directly. You can get one at https://apify.com/ (they have a free tier). You need this for user profile and hashtag searches."

If the user only plans to do keyword searches, they only need SerpAPI. If they only want profile/hashtag scraping, they only need Apify (though SerpAPI is still used for the Google search component).

### 2.3 Save keys

Write the config file at `$TIKSPYDER_DIR/config/config.ini` using this exact format:

```ini
[SerpAPI Key]
api_key = <the_key>

[Apify Token]
apify_token = <the_token>
```

Preserve any existing valid key if the user only provides one of the two.

---

## Phase 3: Collect Search Parameters

Ask the user what they want to collect. Use AskUserQuestion to make this conversational. Here's the decision tree:

### 3.1 Search mode (required)

Ask: "What would you like to search for?"

| Mode | CLI flag | Notes |
|------|----------|-------|
| Keyword search | `--q "term"` | Searches Google for TikTok results matching the term |
| User profile | `--user username` | Scrapes a specific TikTok user's videos (requires `--apify`) |
| Hashtag | `--tag hashtag` | Scrapes videos with a specific hashtag (requires `--apify`) |

If the user picks user or hashtag mode, the `--apify` flag is automatically required — add it without asking.

Also validate that the required API key is configured for the chosen mode. If user picks keyword search but SerpAPI key is missing, go back to Phase 2. Same for Apify with user/hashtag modes.

### 3.2 Additional parameters

After knowing the search mode, ask about these options. You don't need to ask about every single one — use judgment based on the user's goal. Present the most relevant options:

**For keyword searches:**
- Country (`--gl`, e.g., `us`, `gb`, `mx`) — "Which country should Google search from?"
- Language (`--hl`, e.g., `en`, `es`, `fr`) — "What language?"
- Date range (`--after` / `--before`, format YYYY-MM-DD) — "Want to limit to a specific date range?"
- Search depth (`--depth`, default 3) — "How deep should related content search go? Default is 3 levels."

**For user/hashtag searches (Apify):**
- Date filters (`--oldest-post-date` / `--newest-post-date`, format YYYY-MM-DD)
- Number of results (`--number-of-results`, default 25)

**For all modes:**
- Download videos? (`--download`) — "Do you want to download the actual video files?"
- Output directory (`--output`) — "Where should I save the results? Default creates a timestamped folder in `./tikspyder-data/`."
- Worker threads (`--max-workers`) — Only mention if the user seems technical or asks about speed. Default is 5 for downloads, 3 for keyframes.
- Tor (`--use-tor`) — Only mention if privacy is relevant. Requires Tor service running locally.

### 3.3 Date synchronization (critical)

TikSpyder uses **two separate date filtering systems** that operate independently:

- **SerpAPI dates** (`--after` / `--before`) — filter the Google search results
- **Apify dates** (`--oldest-post-date` / `--newest-post-date`) — filter the Apify scraper results

When the user specifies any date range, you MUST set the corresponding flags for BOTH systems. Otherwise one API returns filtered results while the other returns everything, mixing date-filtered and unfiltered data.

**Mapping:**
| User says | SerpAPI flag | Apify flag |
|-----------|-------------|------------|
| "after [date]" / "since [date]" / "from [date]" | `--after [date]` | `--oldest-post-date [date]` |
| "before [date]" / "until [date]" | `--before [date]` | `--newest-post-date [date]` |

**Example:** If the user says "videos after January 2026", the command needs BOTH:
```
--after 2026-01-01 --oldest-post-date 2026-01-01
```

This applies to all search modes — keyword, user, and hashtag. Even though `--after`/`--before` are SerpAPI flags and `--oldest-post-date`/`--newest-post-date` are Apify flags, always include both when dates are specified.

### 3.4 Confirm before running

Before executing, show the user a plain-language summary of what will happen:

```
Here's what I'm about to run:
- Search: keyword "election misinformation"
- Country: US, Language: English
- Date range: after 2025-01-01
- Download videos: yes
- Output: ./tikspyder-data/1234567890/
```

Ask for confirmation before proceeding.

---

## Phase 4: Execute

### 4.1 Activate environment if needed

Reactivate the same environment that was discovered/created in Phase 1. Use whichever applies:

**conda/mamba** (always use the shell hook pattern):
```bash
eval "$(conda shell.bash hook)" && conda activate tikspyder
```

**venv:**
```bash
if [ -f "$TIKSPYDER_DIR/.venv/Scripts/activate" ]; then
  source "$TIKSPYDER_DIR/.venv/Scripts/activate"
elif [ -f "$TIKSPYDER_DIR/.venv/bin/activate" ]; then
  source "$TIKSPYDER_DIR/.venv/bin/activate"
fi
```

Remember which environment type was used in Phase 1 and reuse the same activation here.

### 4.2 Build and run the command

Construct the tikspyder CLI command from the collected parameters. Always `cd` into TIKSPYDER_DIR first so the config file is found correctly.

Example commands (showing conda activation — adapt if using venv):

```bash
# Keyword search with date filter
eval "$(conda shell.bash hook)" && conda activate tikspyder && \
  cd "$TIKSPYDER_DIR" && tikspyder --q "search term" --gl us --hl en \
  --after 2025-01-01 --before 2025-06-01 --output ./data/ --download

# User profile with date filter (note: BOTH --after AND --oldest-post-date)
eval "$(conda shell.bash hook)" && conda activate tikspyder && \
  cd "$TIKSPYDER_DIR" && tikspyder --user username --apify \
  --after 2025-01-01 --oldest-post-date 2025-01-01 --output ./data/

# Hashtag with date filter
eval "$(conda shell.bash hook)" && conda activate tikspyder && \
  cd "$TIKSPYDER_DIR" && tikspyder --tag hashtag --apify \
  --after 2025-01-01 --oldest-post-date 2025-01-01 \
  --number-of-results 50 --output ./data/ --download
```

Run the command and let the user see the output. Use a generous timeout (up to 10 minutes) since data collection can take a while depending on the search scope.

### 4.3 Error handling

Common issues and what to tell the user:

| Error | Likely cause | Fix |
|-------|-------------|-----|
| `ValueError: Either --user, --q or --tag must be provided` | Missing search term | Ask what they want to search |
| `serpapi` authentication error | Bad SerpAPI key | Ask user to check their key at https://serpapi.com/dashboard |
| `apify_client` error | Bad Apify token or insufficient credits | Check token at https://console.apify.com/account |
| `RuntimeError` with asyncio | Event loop conflict | Run `cd "$TIKSPYDER_DIR" && git pull` to get the latest fix |
| Connection/timeout errors | Network issues | Suggest checking internet connection, or trying with fewer results |

### 4.4 Post-run summary

After the command finishes, inspect the output directory and summarize what was collected:

```bash
# Count output files
find <output_dir> -type f | head -50
ls -la <output_dir>/*.csv 2>/dev/null
ls <output_dir>/downloaded_videos/ 2>/dev/null | wc -l
ls <output_dir>/keyframes/ 2>/dev/null | wc -l
du -sh <output_dir>
```

Report to the user:
- Number of CSV data files generated
- Number of videos downloaded (if applicable)
- Number of keyframes extracted (if applicable)
- Total size of the output directory
- Path to the main CSV file(s) they can open in Excel or Google Sheets

---

## Phase 5: Streamlit App (Alternative)

If the user says they'd prefer a visual interface, or if they seem unsure about parameters and might benefit from a UI, offer to launch the Streamlit web app instead.

Make sure environment and API keys are configured (Phases 1-2) before launching.

```bash
cd "$TIKSPYDER_DIR" && tikspyder --app
```

This starts a local web server at `http://localhost:8501`. Tell the user:
- "I've launched the TikSpyder web interface. It should open in your browser at http://localhost:8501"
- "The web interface lets you configure searches, set download options, and track progress visually"
- "Press Ctrl+C in the terminal when you're done to stop the server"

The Streamlit app runs as a blocking process. To keep the conversation going while it runs, launch it in the background:

```bash
cd "$TIKSPYDER_DIR" && nohup tikspyder --app > /dev/null 2>&1 &
echo "Streamlit PID: $!"
```

Tell the user the PID so they (or you) can stop it later.

**Stopping Streamlit:** `tikspyder --app` spawns Streamlit as a child process, so killing just the parent PID won't stop the server. You need to kill the whole process tree. Use this approach:

```bash
# Kill the parent and all child processes
kill $PID 2>/dev/null
# Also find and kill any remaining streamlit processes on port 8501
lsof -ti:8501 2>/dev/null | xargs kill 2>/dev/null || \
  netstat -tlnp 2>/dev/null | grep 8501 | awk '{print $7}' | cut -d'/' -f1 | xargs kill 2>/dev/null
echo "Streamlit server stopped"
```

On Windows (Git Bash), `lsof` may not be available. Fall back to:
```bash
netstat -ano | grep 8501 | awk '{print $5}' | head -1 | xargs taskkill //PID //F 2>/dev/null
```

---

## Quick Reference: Full CLI Flags

| Flag | Type | Description |
|------|------|-------------|
| `--q` | string | Search keyword/phrase |
| `--user` | string | TikTok username |
| `--tag` | string | TikTok hashtag |
| `--gl` | string | Country code (e.g., `us`) |
| `--hl` | string | Language code (e.g., `en`) |
| `--cr` | string | Multiple country filter |
| `--lr` | string | Multiple language filter |
| `--safe` | string | Adult content filter: `active` (default) or `off` |
| `--google-domain` | string | Google domain (default: `google.com`) |
| `--depth` | int | Related content depth (default: 3) |
| `--before` | string | Date upper bound (YYYY-MM-DD) |
| `--after` | string | Date lower bound (YYYY-MM-DD) |
| `--apify` | flag | Enable Apify integration |
| `--oldest-post-date` | string | Apify: oldest post date (YYYY-MM-DD) |
| `--newest-post-date` | string | Apify: newest post date (YYYY-MM-DD) |
| `--number-of-results` | int | Apify: max results (default: 25) |
| `--use-tor` | flag | Route downloads through Tor |
| `-d, --download` | flag | Download video files |
| `-w, --max-workers` | int | Thread count for downloads |
| `-o, --output` | string | Output directory path |
| `--app` | flag | Launch Streamlit web UI |
