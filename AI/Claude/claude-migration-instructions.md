# Migrating the `/meeting-notes` Skill to Another PC

A step-by-step guide for moving the meeting-notes extraction skill (and its runtime dependencies) from one Windows PC to another.

> **Prerequisite:** Claude Code is already installed on the new PC. If it isn't, install it from <https://docs.claude.com/en/docs/claude-code> first.

---

## Overview

The `/meeting-notes` skill is **not a single file** — it is a folder containing:
- A skill definition (e.g. `SKILL.md`) that Claude Code reads.
- Any helper scripts/assets the skill references.

The skill itself is **just instructions**. At runtime it shells out to external tools (`ffmpeg`, `python`, `faster-whisper`), so those must also be installed on the new PC.

The 6 migration steps below cover:
1. (Done) Install Claude Code
2. Locate the skill on the source PC
3. Copy the skill folder to the target PC
4. Install runtime dependencies (`ffmpeg`, Python, `faster-whisper`)
5. Verify the skill appears and works
6. (Optional) Pre-populate the Whisper model cache for offline use

---

## Step 2: Locate the skill on the source PC

The skill lives under your Claude user config directory. On Windows:

```
%USERPROFILE%\.claude\skills\meeting-notes\
```

In a real path that typically resolves to:

```
C:\Users\<your-username>\.claude\skills\meeting-notes\
```

List the contents to confirm what needs to be copied. Open **PowerShell** or **Git Bash** and run:

```bash
dir "%USERPROFILE%\.claude\skills\meeting-notes"
```

or in PowerShell:

```powershell
Get-ChildItem "$env:USERPROFILE\.claude\skills\meeting-notes"
```

You should see something like:

```
SKILL.md
README.md
... (any other helper files)
```

> 💡 **Tip:** If you have multiple skills, the parent folder `%USERPROFILE%\.claude\skills\` will contain one subfolder per skill. You can copy the whole `skills\` directory if you want to migrate everything at once.

---

## Step 3: Copy the skill folder to the target PC

Pick **one** of the two methods below.

### Option A — USB drive or cloud storage (simplest)

1. **On the source PC**, zip the skill folder:
   ```powershell
   Compress-Archive -Path "$env:USERPROFILE\.claude\skills\meeting-notes" -DestinationPath "$env:USERPROFILE\Desktop\meeting-notes.zip"
   ```
2. Transfer `meeting-notes.zip` to the new PC (USB stick, OneDrive, Google Drive, etc.).
3. **On the target PC**, ensure the destination folder exists, then extract:
   ```powershell
   New-Item -ItemType Directory -Force -Path "$env:USERPROFILE\.claude\skills"
   Expand-Archive -Path "<path-to-downloads>\meeting-notes.zip" -DestinationPath "$env:USERPROFILE\.claude\skills\"
   ```
4. Verify:
   ```powershell
   Get-ChildItem "$env:USERPROFILE\.claude\skills\meeting-notes"
   ```

### Option B — Git (recommended if you keep a dotfiles/skills repo)

1. **On the source PC**, turn the skill folder into its own git repo (or a subtree of a personal dotfiles repo):
   ```bash
   cd "$USERPROFILE/.claude/skills/meeting-notes"
   git init
   git add .
   git commit -m "Import meeting-notes skill"
   git remote add origin git@github.com:<your-username>/claude-skills.git
   git push -u origin main
   ```
2. **On the target PC**, clone it into the right place:
   ```bash
   mkdir -p "$USERPROFILE/.claude/skills"
   cd "$USERPROFILE/.claude/skills"
   git clone git@github.com:<your-username>/claude-skills.git meeting-notes
   ```

   > If the folder name is part of a larger repo, adjust accordingly (e.g. sparse-checkout, subtree split, or just copy the subfolder out).

---

## Step 4: Install the runtime dependencies

The skill calls out to three external tools. Install each on the target PC.

| Tool | Why it is needed | Install command (Windows) |
|------|------------------|---------------------------|
| **ffmpeg** (includes **ffprobe**) | Audio extraction, audio chunking, video frame extraction | `winget install Gyan.FFmpeg` |
| **Python 3.10+** | Runs the `faster-whisper` transcription script | `winget install Python.Python.3.12` |
| **faster-whisper** | The actual speech-to-text model loader | `pip install faster-whisper` |

### 4.1 — Install ffmpeg

Easiest: use Windows Package Manager (`winget`, built into Windows 11).

```powershell
winget install --id Gyan.FFmpeg -e
```

Alternative: download a static build from <https://www.gyan.dev/ffmpeg/builds/> and add the `bin\` folder to your `PATH`.

### 4.2 — Install Python

```powershell
winget install --id Python.Python.3.12 -e
```

> ⚠️ **Important:** On the Python installer / first-run prompt, make sure the **“Add Python to PATH”** checkbox is ticked. Restart PowerShell afterwards so `python` resolves.

### 4.3 — Install faster-whisper

```powershell
pip install faster-whisper
```

(If you have multiple Python versions, use `py -3.12 -m pip install faster-whisper`.)

### 4.4 — Verify everything

Run each command and confirm a version / “ok” message:

```powershell
ffmpeg -version
ffprobe -version
python --version
python -c "import faster_whisper; print('faster-whisper ok')"
```

All four should print version info. If any command is “not recognized,” close and reopen the terminal (so it picks up the new `PATH`).

---

## Step 5: Verify the skill works

1. **Restart Claude Code** on the target PC. Claude Code scans the skills folder at startup; it won’t see a skill added mid-session.
2. Type `/` in the prompt. `meeting-notes` should appear in the slash-command list. If it does not:
   - Confirm the folder path is **exactly** `%USERPROFILE%\.claude\skills\meeting-notes\` (no extra nesting).
   - Confirm the skill definition file (usually `SKILL.md`) exists inside.
   - Check Claude Code logs / run `claude --debug` for skill-loading errors.
3. **Smoke-test** the skill on a short audio or video file (a 1–2 minute clip is plenty):
   - Run `/meeting-notes` and pick **Single file**.
   - Point it at a short test file in your `C:\Recordings\` directory.
   - Confirm it produces: `audio.wav` → `chunks/` → `transcript.txt` → `frames/` → `<basename>-meeting-notes.html`.
4. Open the generated HTML in a browser and check that:
   - The header, executive summary, topics, and visual corrections log all render.
   - `[AUDIO]` / `[VISUAL]` / `[UNCERTAIN]` tags are present.

If all four sections render and the HTML opens cleanly, the migration is complete.

---

## Step 6 (Optional): Pre-populate the Whisper model cache

The first time the skill runs on a new PC, `faster-whisper` downloads the `tiny` model (~75 MB) into Hugging Face’s cache directory. If you want the new PC to work **offline from day one**, copy the cache from the source PC.

### 6.1 — Find the cache on the source PC

The default Hugging Face cache path on Windows is:

```
%USERPROFILE%\.cache\huggingface\
```

(For the specific `faster-whisper` model files, look under `%USERPROFILE%\.cache\huggingface\hub\`.)

### 6.2 — Copy the cache

1. **On the source PC**, zip the cache:
   ```powershell
   Compress-Archive -Path "$env:USERPROFILE\.cache\huggingface" -DestinationPath "$env:USERPROFILE\Desktop\huggingface-cache.zip"
   ```
2. Transfer the zip to the target PC.
3. **On the target PC**, extract it to the matching path:
   ```powershell
   New-Item -ItemType Directory -Force -Path "$env:USERPROFILE\.cache"
   Expand-Archive -Path "<path-to-downloads>\huggingface-cache.zip" -DestinationPath "$env:USERPROFILE\.cache\"
   ```

### 6.3 — Verify

```powershell
Get-ChildItem "$env:USERPROFILE\.cache\huggingface\hub"
```

You should see the model snapshot folders. The first `/meeting-notes` run on the new PC will now skip the download and go straight to transcription.

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| `/` doesn’t show `meeting-notes` | Wrong folder path or skill definition filename | Confirm path is `%USERPROFILE%\.claude\skills\meeting-notes\` and `SKILL.md` exists |
| `ffmpeg` not recognized | `PATH` not refreshed | Close & reopen terminal; or re-login to Windows |
| `python` not recognized | Not added to `PATH` at install | Reinstall Python with “Add to PATH” checked, or use `py` launcher |
| `faster-whisper` import fails | Installed against a different Python than the one being run | Use `python -m pip install faster-whisper` to target the active interpreter |
| Skill runs but produces no transcript | Audio extraction failed silently | Run `ffmpeg -i input.mp4 -vn audio.wav` manually to see errors |
| Whisper download is slow / blocked | Network/proxy | Pre-populate the cache (Step 6), or set `HF_HUB_OFFLINE=1` once cached |

---

## One-shot setup script (PowerShell)

If you want to automate steps 3–5 on a fresh PC, save the following as `setup-meeting-notes.ps1` and run it as Administrator:

```powershell
# setup-meeting-notes.ps1
# Run as: powershell -ExecutionPolicy Bypass -File .\setup-meeting-notes.ps1

$ErrorActionPreference = "Stop"

Write-Host "==> Installing ffmpeg..." -ForegroundColor Cyan
winget install --id Gyan.FFmpeg -e --accept-source-agreements --accept-package-agreements

Write-Host "==> Installing Python 3.12..." -ForegroundColor Cyan
winget install --id Python.Python.3.12 -e --accept-source-agreements --accept-package-agreements

Write-Host "==> Refreshing PATH for current session..." -ForegroundColor Cyan
$env:Path = [System.Environment]::GetEnvironmentVariable("Path","Machine") + ";" + [System.Environment]::GetEnvironmentVariable("Path","User")

Write-Host "==> Installing faster-whisper..." -ForegroundColor Cyan
python -m pip install --upgrade pip
python -m pip install faster-whisper

Write-Host "==> Creating skills folder..." -ForegroundColor Cyan
New-Item -ItemType Directory -Force -Path "$env:USERPROFILE\.claude\skills"

Write-Host ""
Write-Host "==> Next steps:" -ForegroundColor Green
Write-Host "    1. Copy your meeting-notes skill folder into: $env:USERPROFILE\.claude\skills\"
Write-Host "    2. Restart Claude Code."
Write-Host "    3. Type / and confirm meeting-notes appears."
Write-Host "    4. Run a smoke test on a short audio/video file."
```

---

## Reference — what gets installed where

```
C:\Users\<user>\.claude\skills\meeting-notes\   <- the skill (this is what you migrate)
C:\Users\<user>\.cache\huggingface\              <- whisper model cache (optional copy)
C:\ProgramData\chocolatey\lib\ffmpeg\...         <- ffmpeg (or wherever winget puts it)
C:\Users\<user>\AppData\Local\Programs\Python\   <- python install
site-packages\faster_whisper\                    <- pip package
```

---

## Related

- Claude Code skills overview: <https://docs.claude.com/en/docs/claude-code/skills>
- The companion guide `claude_cli_skills_implementation_guide.md` in this folder walks through authoring a new skill from scratch.
- `uninstall_claude_clean.md` (same folder) covers the reverse — a clean uninstall of Claude Code and its skill folder.
