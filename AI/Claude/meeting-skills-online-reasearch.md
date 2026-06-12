# Claude Skills for Video / Meeting Transcription — Community Research

> Research compiled 2026-06-12. There is **no single canonical "official" Claude skill** for transcribing video conferences — but there is a healthy ecosystem of community-maintained skills, organized in three layers: **raw transcription**, **frame-aware video analysis**, and **transcript → structured minutes**.

---

## TL;DR — What to use when

| If you want… | Use |
|---|---|
| Pure audio/video → text (with optional jargon grounding) | [`SpillwaveSolutions/whisper-transcribe`](https://github.com/spillwavesolutions/whisper-transcribe) |
| `/transcribe` slash command with critical analysis (whisper.cpp, YouTube support) | [`jftuga/transcript-critic`](https://github.com/jftuga/transcript-critic) ⭐ 19 |
| "Claude watches a video" (frames + audio, 7+ platforms) | [`Newuxtreme/watch-video-skill`](https://github.com/Newuxtreme/watch-video-skill) ⭐ 18 |
| Frame + transcript correlation in one bash script | [`msadig` Video Input Skill (gist)](https://gist.github.com/msadig/b109ff286929b79c14a8480e9b848651) |
| Multi-speaker diarization, 99 languages, long recordings | [`zxkane/audio-transcriber-funasr`](https://github.com/zxkane/meeting-transcriber) |
| A community index of Claude skills | [`majiayu000/claude-skill-registry`](https://github.com/majiayu000/claude-skill-registry) ⭐ 212 |
| Transcript → structured meeting minutes | [`minutes` (seabbs, skills.rest)](https://skills.rest/skill/minutes) |
| Batch transcript → Obsidian knowledge graph | [`summarize-meetings` (2389 Research)](https://skills.2389.ai/plugins/summarize-meetings/) |
| Minutes from a meeting transcript (general purpose) | [`phuryn/pm-skills` — "Summarize Meeting"](https://mcpmarket.com/tools/skills/meeting-summary-minutes) ⭐ 9,437 on parent |

---

## Layer 1 — Raw transcription (audio/video → text)

### 1.1 [`SpillwaveSolutions/whisper-transcribe`](https://github.com/spillwavesolutions/whisper-transcribe)

A dedicated Claude Code skill for transcribing audio/video files using OpenAI's Whisper.

- **Key feature:** **Context grounding** — reads `.md` files in the same directory to improve accuracy for technical terms, names, and jargon.
- **Supported formats:** mp3, wav, m4a, mp4, webm.
- **Install:** `npx skilz install SpillwaveSolutions_whisper-transcribe/whisper-transcribe`
- **Triggers:** `whisper`, `transcribe`, `audio to text`, `video to text`, `speech to text`.

### 1.2 [`jftuga/transcript-critic`](https://github.com/jftuga/transcript-critic) ⭐ 19

Combines **whisper.cpp** transcription with structured critical analysis.

- **Supports:** Local audio files, `.vtt` files, and YouTube/yt-dlp URLs.
- **Output:** Markdown analysis with timestamped summaries, evidence notes, logical fallacies, and underdeveloped areas.
- **Usage:** `/transcribe recording.m4a` or `/transcribe https://youtube.com/watch?v=...`.

### 1.3 [`Newuxtreme/watch-video-skill`](https://github.com/Newuxtreme/watch-video-skill) ⭐ 18

Lets Claude "watch" videos by combining transcription with frame extraction.

- **v2 features:** Auto-scaled frames (cap of 100), cloud Whisper via **Groq/OpenAI API**, focused mode (`--start` / `--end`).
- **Supports:** YouTube, Vimeo, TikTok, X, Twitch, Loom, Instagram, and local files.
- **Usage:** `/watch-video <URL-or-path>`.
- **Output:** Structured markdown notes with TL;DR, timeline, key quotes, and visual notes.

### 1.4 [`msadig`'s Video Input Skill (gist)](https://gist.github.com/msadig/b109ff286929b79c14a8480e9b848651)

A gist-based skill that extracts frames + transcription and matches them temporally.

- **Workflow:** Frame extraction (ffmpeg) → Audio transcription (whisper) → Frame-to-transcript correlation.
- **Output:** Comprehensive markdown with a frame-transcription timeline table.
- **Includes:** Robust `analyze-video.sh` bash script with error handling.

### 1.5 [`zxkane/audio-transcriber-funasr`](https://github.com/zxkane/meeting-transcriber)

Full agent skill + Claude Code plugin for multi-speaker meeting & podcast transcription.

- **99 languages** supported (zh/en/ja/ko/yue + Whisper).
- **Speaker diarization** with real name mapping.
- **GPU & CPU** support.
- **LLM cleanup** via Bedrock Claude.
- **Long recordings:** handles 4+ hours.
- **Install:**
  ```bash
  npx skills add zxkane/audio-transcriber-funasr
  # or
  /plugin marketplace add zxkane/audio-transcriber-funasr
  /plugin install funasr-transcriber@zxkane-audio-transcriber-funasr
  ```

### 1.6 [`majiayu000/claude-skill-registry`](https://github.com/majiayu000/claude-skill-registry) ⭐ 212

The most comprehensive Claude Code skills registry. **Not a skill itself** — a curated index that includes the `whisper-transcribe` skill definition and many others. Closest thing to an "official community directory" today.

---

## Layer 2 — Transcript → structured meeting minutes

### 2.1 [`minutes` (seabbs)](https://skills.rest/skill/minutes)

- **Purpose:** Converts raw transcripts → structured meeting minutes.
- **Extracts:** decisions, action items, attendees, key topics.
- **Listed at:** <https://skills.rest/skill/minutes>

### 2.2 [`summarize-meetings` (2389 Research)](https://skills.2389.ai/plugins/summarize-meetings/)

- **Purpose:** Batch-processes Obsidian transcripts → knowledge graph.
- **Marketplace:** vercel-labs/skills + native Claude Code.
- **Install:** `npx skills add 2389-research/summarize-meetings`.

### 2.3 [`Summarize Meeting` (ClaudSkills)](https://claudskills.com/skills/summarize-meeting/)

- **Purpose:** Structured notes with date, participants, decisions, action items.

### 2.4 [`Meeting Summary & Minutes` (phuryn / pm-skills)](https://mcpmarket.com/tools/skills/meeting-summary-minutes)

- **GitHub:** `phuryn/pm-skills` (⭐ 9,437 on parent repo).
- **Install:** `npx skillfish add phuryn/pm-skills summarize-meeting`.

---

## Common prerequisites

Across nearly all skills above, you will need:

| Tool | Why | Install (Windows) |
|------|-----|-------------------|
| **ffmpeg** (includes `ffprobe`) | Audio extraction, chunking, frame extraction | `winget install Gyan.FFmpeg` |
| **Python 3.9+** | Runs Whisper scripts | `winget install Python.Python.3.12` |
| **Whisper** (one of) | Transcription | `pip install faster-whisper` *or* `pip install openai-whisper` *or* `brew install openai-whisper` |
| **yt-dlp** (optional) | YouTube/URL support | `winget install yt-dlp.yt-dlp` |

> **Tip:** If you're already running the `/meeting-notes` skill on this PC, ffmpeg + Python + `faster-whisper` are likely already installed. Adding a new skill is usually just a folder copy.

---

## End-to-end recommended stack

For a **complete video conference → structured minutes** pipeline, combine:

1. **[`zxkane/audio-transcriber-funasr`](https://github.com/zxkane/meeting-transcriber)** — for the raw audio → text + speaker ID stage.
2. **[`minutes` (seabbs)](https://skills.rest/skill/minutes)** *or* **[`summarize-meetings` (2389 Research)](https://skills.2389.ai/plugins/summarize-meetings/)** — for the transcript → minutes stage.

For a **complete video → understanding** pipeline (audio + frames):

1. **[`Newuxtreme/watch-video-skill`](https://github.com/Newuxtreme/watch-video-skill)** *or* **[`msadig`'s Video Input Skill](https://gist.github.com/msadig/b109ff286929b79c14a8480e9b848651)** — for frames + audio correlation.
2. Add **Layer 2** skills on top if you also want structured minutes.

---

## What the "famous" demo is

If you've seen a single skill widely demo'd on YouTube or X, it's likely **Matthew Berman demonstrating a "meeting notes" skill** that drops an audio file into `~/.claude/meetings/` and auto-processes it. That demo is the same *shape* as the `/meeting-notes` skill — the underlying idea is widespread; **no single skill has yet won a "standard" position** in the ecosystem.

---

## Cross-references

- [Claude Code skills overview (official docs)](https://docs.claude.com/en/docs/claude-code/skills)
- [`claude_cli_skills_implementation_guide.md`](./claude_cli_skills_implementation_guide.md) — authoring a new skill from scratch.
- [`claude-migration-instructions.md`](./claude-migration-instructions.md) — moving a skill to another PC.
- [`uninstall_claude_clean.md`](./uninstall_claude_clean.md) — clean uninstall of Claude Code + skills.
- [`claude-code-repos.md`](./claude-code-repos.md) — index of Claude-related repos.
