# TikSpyder Agent Skill

A [Claude Code](https://claude.ai/code) skill that lets you collect TikTok data through conversation — no command-line experience needed.

This skill wraps [TikSpyder](https://github.com/estebanpdl/tik-spyder), an OSINT tool for collecting TikTok videos, metadata, and media. Instead of learning CLI flags and managing configurations manually, you describe what you want in plain language and the AI agent handles the rest.

## What it does

- Sets up the TikSpyder environment automatically (clones the repo, creates a Python environment, installs dependencies)
- Configures API credentials (SerpAPI and Apify) securely
- Collects parameters through conversation — search keywords, usernames, hashtags, date ranges, download preferences
- Runs the tool and summarizes results
- Can launch the Streamlit web interface as an alternative to CLI

## Requirements

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) installed and working
- Python 3.11 or newer
- [ffmpeg](https://ffmpeg.org/download.html) (optional — needed for keyframe extraction)
- API keys (the skill will guide you through getting these):
  - [SerpAPI](https://serpapi.com/) — for keyword-based TikTok searches
  - [Apify](https://apify.com/) — for TikTok profile and hashtag scraping

## Installation

### Option 1: Clone as a plugin (recommended)

```bash
# User-level (available across all your projects)
git clone https://github.com/estebanpdl/tikspyder-agent-skill.git ~/.claude/plugins/tikspyder-agent-skill

# Or project-level (available only in one project)
git clone https://github.com/estebanpdl/tikspyder-agent-skill.git .claude/plugins/tikspyder-agent-skill
```

### Option 2: Install from a marketplace

If a marketplace has this plugin listed:

```
/plugin install tikspyder-agent-skill@<marketplace-name>
```

Restart Claude Code after installing. The skill will be available immediately.

## Usage

Just tell Claude what you want to collect. The skill triggers automatically — you don't need to reference it by name.

**Example prompts:**

```
"Collect TikTok videos about climate change from the last month"

"Scrape all recent videos from @username on TikTok"

"Search TikTok for videos with the hashtag #protest, download them too"

"I want to collect TikTok data but I'd rather use a visual interface"
```

On first run, the skill will:
1. Clone the TikSpyder tool into its own directory
2. Set up a Python environment (detects conda/mamba if available, otherwise creates a venv)
3. Ask for your API keys if not already configured
4. Guide you through search parameters
5. Run the collection and summarize what was gathered

Subsequent runs skip the setup and go straight to data collection.

## What gets collected

Depending on your search parameters, TikSpyder outputs:

| Output | Description |
|--------|-------------|
| CSV files | Structured metadata (titles, authors, dates, engagement metrics) |
| SQLite database | All collected data in queryable format |
| Videos | Downloaded .mp4 files (if requested) |
| Audio | Extracted .mp3 files (if videos are downloaded) |
| Thumbnails | Video thumbnail images |
| Keyframes | Extracted keyframes from videos (requires ffmpeg) |

## Search modes

| Mode | What it does | API needed |
|------|-------------|------------|
| Keyword search | Searches Google for TikTok content matching your query | SerpAPI |
| User profile | Scrapes videos from a specific TikTok account | SerpAPI + Apify |
| Hashtag | Scrapes videos with a specific hashtag | SerpAPI + Apify |

## License

This skill is provided as-is. TikSpyder is developed by [Esteban Ponce de Leon](https://github.com/estebanpdl).
