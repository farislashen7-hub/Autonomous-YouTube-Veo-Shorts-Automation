
# YouTube Shorts Autopilot

Automated pipeline that finds trending Shorts, generates new video ideas, produces each video with Google Veo, writes the SEO metadata, and uploads it to YouTube. A Google Sheet is the control panel: ideas land in it, rows marked `approved` get produced, rows marked `uploaded` are done.

Current niche: wordless Pixar-style 3D family comedy shorts (dad + two toddlers), Arabic titles and descriptions for an Egyptian audience.

## Features

**1. Idea generation (manual run)**
- Searches the top Shorts of the last 30 days for the configured niche (YouTube Data API)
- Uses those titles for title patterns only, and copies the formula and style of a reference story, never its plot
- LLM returns structured ideas: title, hook, angle, 8 to 10 scenes with timing, camera, and SFX, style prompt, character sheet, loop ending, thumbnail idea, tags, viral score
- Code validates each idea (6 to 12 scenes, total length within 2 s of the target) and drops bad ones
- Ideas scoring at or above `approve_score` get `approved`, the rest `new`, then rows are appended to the sheet

**2. Video production and upload (daily 09:00)**
- Picks the first `approved` row
- Splits the scenes into 4 segments: Veo generates the first clip, then extends it 3 times for about 29 s, 9:16
- Polls Veo status every 45 s per segment
- Writes Arabic title, description, hashtags, and tags (`#Shorts` first), validated by a structured output parser with auto-fix
- Downloads the final video and uploads it to YouTube as **private**, then marks the row `uploaded`

**3. Legacy pipeline (ASMR glass-cutting)**
- Older flow that turns a row into a single Veo prompt (colored glass cut by a steel knife), generates one clip, writes English SEO, and uploads
- Still wired to its own 09:00 trigger, see Known issues

## Tech stack

| Layer | Tools |
|---|---|
| Automation | n8n (schedule, AI Agent, chain, structured parsers) |
| Video | Google Veo 3 Fast via kie.ai (generate, extend, status) |
| LLM | OpenRouter (ideas), Ollama `qwen2.5:3b` (SEO, JSON repair, legacy prompt) |
| Research | YouTube Data API v3 (`search`) |
| Control panel | Google Sheets |
| Publishing | YouTube upload (OAuth2) |

## Architecture

```
Manual ─► Config ─► YouTube top Shorts (30d) ─► titles ─► LLM ideas ─► validate
                                                                         │
                                                         score >= 8 ─► approved
                                                         otherwise  ─► new
                                                                         ▼
                                                                  Google Sheet
                                                                         │
Daily 09:00 ─► rows where status = approved ─► first row                 │
        ┌──────────────────────────────────────────────────────◄────────┘
        ▼
  Build 4 segments ─► Veo generate ─► wait 45s ─► status ─► Veo extend ×3
        (pending ─► wait again, failed ─► stop)
        ▼
  Arabic SEO agent ─► download video ─► YouTube upload (private) ─► status = uploaded
```

## Getting started

### 1. Import

Import the workflow JSON into n8n and re-link every credential.

### 2. Credentials

| Credential | Type | Used by |
|---|---|---|
| Header Auth (kie.ai) | `httpHeaderAuth` | Veo generate, extend, status, download. Header `Authorization: Bearer <key>` |
| Query Auth (YouTube) | `httpQueryAuth` | Trend search. Param `key` = Data API key |
| YouTube OAuth2 | `youTubeOAuth2Api` | Upload |
| Google Sheets | `googleSheetsOAuth2Api` | Read, append, update rows |
| OpenRouter | `openRouterApi` | Idea generation |
| Ollama | `ollamaApi` | SEO agent, parser auto-fix |

### 3. Sheet

One sheet with these columns: `title`, `hook`, `angle`, `format`, `duration_sec`, `characters`, `style_prompt`, `scenes`, `loop_ending`, `thumbnail_idea`, `tags`, `score`, `status`, `date`.
Status values: `new`, `approved`, `uploaded`.

### 4. Configure the niche

Edit the `Config2` node:

| Field | Meaning |
|---|---|
| `niche` | Search query, several terms separated by `|` |
| `language` | Title and hook language (`ar`) |
| `count` | Ideas per run |
| `approve_score` | Minimum score for auto-approval |
| `duration_sec` | Target length (30) |
| `reference_story`, `reference_style` | The formula the ideas must copy |

### 5. Run

1. Run the manual trigger to fill the sheet with ideas.
2. Review the sheet. Set `status` to `approved` for the ideas you want produced.
3. Activate the workflow. At 09:00 the next approved row is produced and uploaded as private.
4. Review the video in YouTube Studio, then publish.

## Known issues

| Issue | Fix |
|---|---|
| **Two daily triggers** (`Daily 09:00` legacy, `Daily 09:`) both read the same `approved` rows. Both generate a video for the same idea, double Veo cost and double upload | Delete or disable the legacy branch (everything from `Daily 09:00` to `Sheet: Mark Uploaded`) |
| Veo wait loops (`pending` returns to `Wait`) have no attempt cap. A stuck task loops forever | Add a counter, fail after about 20 polls |
| Failed runs leave the row `approved`, so the next day it is regenerated and paid for again | Set `processing` before generating and `failed` on error, only pick `approved` |
| Upload failure after a paid generation loses the video | Store the final video URL in the sheet before uploading |
| Sheet append maps `format ` and `characters ` with trailing spaces in the column names, and `date ` is never written | Rename the sheet headers without spaces and map `date` |
| Aspect ratio key is `aspect_ratio` in the new flow and `aspectRatio` in the legacy one. One is wrong | Check the kie.ai Veo docs and use one key |
| Extend flow takes the last `resultUrls[0]` as the final video and ignores the collected `urls` | Verify that an extended task returns the full merged video, otherwise concatenate |
| `OpenRouter Chat Model4` has no model set, so it uses the node default | Pin the model explicitly |
| `qwen2.5:3b` writes weak Arabic and unreliable JSON | Use a stronger model for the SEO agent |
| Ideas are auto-approved from the LLM's own score | Raise `approve_score` or approve by hand |

## Security notes

- Keep all keys in n8n credentials. Never inline them in nodes or prompts.
- Uploads are set to `private`. Keep it that way until a human has watched the video. Videos from unaudited API projects are forced private by YouTube anyway.
- Veo calls cost money per generation. Fix the duplicate trigger and add retry caps before leaving this unattended.
- Check platform rules on AI-generated content and use the disclosure option when uploading.
- Do not commit exported workflow JSON with credential IDs or sheet IDs to a public repo.

## Project structure

```
workflow.json     exported n8n workflow
README.md
```

## Author

**Faris Ataa Lashen**, AI Automation Engineer (n8n, AI Agents, RAG, LLMs)
Portfolio: https://farislashen7-hub.github.io/portfolio · LinkedIn: https://linkedin.com/in/farislashen
