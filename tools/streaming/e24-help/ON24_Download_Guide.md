# ON24 Recording Download Guide for Windows 10

> **Author:** Vaibhav A  
> **Last Updated:** 2026-06-01  
> **Target Platform:** Windows 10 (with Git Bash, Docker Desktop, WSL installed)  
> **Purpose:** Download recorded live event streams from ON24 platform (e.g., O'Reilly Media)

---

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Tool Installation](#tool-installation)
    - [1. Install ffmpeg](#1-install-ffmpeg)
    - [2. Install yt-dlp](#2-install-yt-dlp)
    - [3. Install Browser Cookie Exporter](#3-install-browser-cookie-exporter)
3. [Method 1: yt-dlp (Recommended)](#method-1-yt-dlp-recommended)
    - [Step 5: Download Highest Quality](#step-5-download-highest-quality)
    - [Using youtube-dl (Alternative)](#using-youtube-dl-alternative)
4. [Method 2: Browser Developer Tools + ffmpeg](#method-2-browser-developer-tools--ffmpeg)
5. [Method 3: Download Slides + Audio (Bonus)](#method-3-download-slides--audio-bonus)
6. [Method 4: Screen Recording (Last Resort)](#method-4-screen-recording-last-resort)
7. [Troubleshooting](#troubleshooting)
8. [Understanding the ON24 URL](#understanding-the-on24-url)
9. [References](#references)

---

## Prerequisites

Before starting, ensure you have:

| Tool | Purpose | Required? |
|------|---------|-----------|
| **Google Chrome / Firefox / Edge** | Browser to watch & export cookies | ✅ Required |
| **Git Bash** (already installed) | Running shell commands | ✅ Required |
| **Docker Desktop** (already installed) | Alternative environment if needed | ❌ Optional |
| **WSL** (already installed) | Linux subsystem alternative | ❌ Optional |
| **O'Reilly Subscription** | Active paid subscription | ✅ Required |

---

## Tool Installation

### 1. Install ffmpeg

`ffmpeg` is the industry-standard tool for video/audio processing. It can download HLS streams, remux containers, and combine audio with images.

#### Option A: Winget (Recommended - Easiest)

Open **PowerShell as Administrator** and run:

```powershell
winget install ffmpeg
```

After installation, **close and reopen your terminal**, then verify:

```powershell
ffmpeg -version
```

Expected output: `ffmpeg version ...` with configuration details.

#### Option B: Manual Download

1. Go to https://ffmpeg.org/download.html
2. Under "Windows" → click "Windows builds from gyan.dev"
3. Download `ffmpeg-release-full.7z` (the "full" build)
4. Extract with **7-Zip** (install from https://7-zip.org if needed)
5. Copy the `bin` folder contents (ffmpeg.exe, ffprobe.exe, ffplay.exe) to a permanent location, e.g., `C:\Tools\ffmpeg\`
6. Add to PATH:
    - Press **Win + X** → **System** → **Advanced system settings**
    - Click **Environment Variables**
    - Under **System variables**, find `Path` → **Edit**
    - Add `C:\Tools\ffmpeg\` → **OK**
7. Verify: Open a **new** PowerShell/Git Bash window and run `ffmpeg -version`

#### Option C: Chocolatey

```powershell
choco install ffmpeg
```

---

### 2. Install yt-dlp

`yt-dlp` is a feature-rich command-line video downloader with **native ON24 support** (added March 2025).

> **Note:** ON24 support requires yt-dlp **version 2025.03.31 or newer**.

#### Option A: Winget (Recommended)

Open **PowerShell** and run:

```powershell
winget install yt-dlp.yt-dlp
```

#### Option B: pip / pipx (Requires Python)

If you already have Python installed, this is the fastest method:

```powershell
# Using pip
pip install yt-dlp

# Or using pipx (isolated environment, no dependency conflicts)
pipx install yt-dlp
```

> **Note:** If you have both Python 2 and Python 3, use `pip3 install yt-dlp` to ensure it installs under Python 3.

To upgrade later:

```powershell
pip install --upgrade yt-dlp
# Or with pipx:
pipx upgrade yt-dlp
```

#### Option C: Manual Download

```powershell
# Download the exe
curl -L https://github.com/yt-dlp/yt-dlp/releases/latest/download/yt-dlp.exe -o "$env:USERPROFILE\Downloads\yt-dlp.exe"

# Move to a permanent location in PATH
mkdir C:\Tools\yt-dlp -Force
Move-Item "$env:USERPROFILE\Downloads\yt-dlp.exe" C:\Tools\yt-dlp\

# Add C:\Tools\yt-dlp to your PATH (same steps as ffmpeg PATH setup above)
```

#### Verify Installation

Open a **new** terminal and run:

```powershell
yt-dlp --version
```

Expected: `2025.03.31` or higher.

#### Update yt-dlp (if already installed)

```powershell
yt-dlp -U
```

---

### 3. Install Browser Cookie Exporter

yt-dlp needs your browser cookies to authenticate with ON24 through your O'Reilly account.

#### For Google Chrome / Microsoft Edge / Brave

1. Open Chrome/Edge/Brave
2. Go to: https://chromewebstore.google.com/detail/get-cookiestxt-locally/cclelndahbckbenkjhflpdbgdldlbecc
3. Click **Add to Chrome** (or **Get** for Edge)
4. Pin it to your toolbar for easy access

#### For Firefox

1. Open Firefox
2. Go to: https://addons.mozilla.org/en-US/firefox/addon/get-cookies-txt-locally/
3. Click **Add to Firefox**

> **Alternative:** Skip the extension and use yt-dlp's built-in browser cookie extraction: `--cookies-from-browser chrome` (supports chrome, firefox, edge, brave, opera)

---

## Method 1: yt-dlp (Recommended)

This is the **easiest and most reliable method**. yt-dlp has a built-in extractor for ON24 that directly finds and downloads the video stream.

### Step-by-Step

#### Step 1: Log into O'Reilly

1. Open your browser (Chrome/Firefox/Edge)
2. Go to https://learning.oreilly.com/
3. **Sign in** with your credentials
4. Navigate to the ON24 recording you want to download
5. **Start playing the video** to ensure your session is active

#### Step 2: Copy the ON24 URL

Right-click on the video page and copy the URL. It will look like:

```
https://event.on24.com/eventRegistration/console/apollox/mainEvent?&eventid=5270653&sessionid=1&username=&partnerref=&format=fhvideo1&mobile=&flashsupportedmobiledevice=&helpcenter=&key=CAEFA4522A10C3E58E300BA9A1347EDF&newConsole=true&nxChe=true&newTabCon=true&consoleEarEventConsole=true&consoleEarCloudApi=false&text_language_id=en&playerwidth=748&playerheight=526&referrer=...&eventuserid=829567966&contenttype=A&mediametricsessionid=714599988&mediametricid=7385373&usercd=829567966&mode=launch
```

The **critical parts** are:
- `eventid=5270653` — the event ID
- `key=CAEFA4522A10C3E58E300BA9A1347EDF` — the authentication key

> **Note:** The `key` parameter is tied to your session. If it expires, you'll need to revisit O'Reilly to generate a fresh URL.

#### Step 3: Download Video (Method A — Direct Browser Cookies)

Open **Git Bash** or **PowerShell** and run:

```bash
# Download best quality video
yt-dlp --cookies-from-browser chrome -o "lecture.mp4" "PASTE_YOUR_ON24_URL_HERE"
```

Replace `chrome` with your browser: `firefox`, `edge`, `brave`, `opera`.

> ⚠️ **Important:** Close your browser before running this command, or yt-dlp may fail to read the cookie database.

#### Step 3 (Alternative): Download Video (Method B — Cookie File)

If the direct browser method fails:

1. **Keep your browser open** with O'Reilly logged in
2. Click the **Get cookies.txt LOCALLY** extension icon in your toolbar
3. Click **Export** → save as `cookies.txt` (e.g., in `C:\Users\vibs2\Downloads\cookies.txt`)
4. Run:

```bash
yt-dlp --cookies "C:\Users\vibs2\Downloads\cookies.txt" -o "lecture.mp4" "PASTE_YOUR_ON24_URL_HERE"
```

#### Step 4: Check Available Formats

To see what formats are available before downloading:

```bash
yt-dlp --cookies-from-browser chrome -F "YOUR_ON24_URL"
```

Expected output:

```
ID      EXT  RESOLUTION  PROTO  VCODEC      ACODEC
video   mp4  unknown     https  avc1.640020 mp4a.40.2
```

ON24 typically provides a single MP4 video stream and an audio-only WAV stream.

#### Step 5: Download Highest Quality

`yt-dlp` defaults to downloading the best available quality, but you can be explicit with format selectors:

```bash
# Highest quality video+audio (recommended for most use cases)
yt-dlp --cookies-from-browser chrome -f "bestvideo+bestaudio/best" -o "lecture.mp4" "URL"

# Highest quality video+audio merged into MP4 container
yt-dlp --cookies-from-browser chrome -f "bestvideo+bestaudio" --merge-output-format mp4 -o "lecture.mp4" "URL"

# Best single pre-merged file (no remuxing needed)
yt-dlp --cookies-from-browser chrome -f "best" -o "lecture.mp4" "URL"

# Highest quality video only (no audio)
yt-dlp --cookies-from-browser chrome -f "bestvideo" -o "lecture_video.mp4" "URL"

# Highest quality audio only
yt-dlp --cookies-from-browser chrome -f "bestaudio" -o "lecture_audio.wav" "URL"

# Download a specific format (use format IDs from -F output)
yt-dlp --cookies-from-browser chrome -f "video" -o "lecture.mp4" "URL"
```

> **Tip:** For ON24 streams, the format IDs shown by `-F` are typically `video` and `audio`. Since ON24 usually offers a single video stream, the default behavior (`-f "bestvideo+bestaudio/best"`) already selects the highest quality. Use `-F` first to see what's available before choosing a specific format.

#### Using youtube-dl (Alternative)

`youtube-dl` is the predecessor of `yt-dlp`. It is no longer actively maintained but may still be installed on some systems. Its highest-quality flags are slightly different:

```powershell
# Install (pip only — no winget package)
pip install youtube-dl
```

```bash
# Highest quality video+audio
youtube-dl -f "bestvideo+bestaudio/best" -o "lecture.mp4" "URL"

# Best single pre-merged file
youtube-dl -f "best" -o "lecture.mp4" "URL"

# Highest quality video only (no audio)
youtube-dl -f "bestvideo" -o "lecture_video.mp4" "URL"

# Highest quality audio only
youtube-dl -f "bestaudio" -o "lecture_audio.wav" "URL"
```

> **⚠️ Warning:** `youtube-dl` does **not** have a native ON24 extractor. It will not work for ON24 streams unless the direct stream URL is provided. For ON24, always prefer `yt-dlp`.

#### Full Command Reference

```bash
# Basic download
yt-dlp --cookies-from-browser chrome -o "output.mp4" "URL"

# With retries for poor connections
yt-dlp --cookies-from-browser chrome -o "output.mp4" --retries 10 --fragment-retries 10 "URL"

# Resume partial downloads
yt-dlp --cookies-from-browser chrome -o "output.mp4" --continue "URL"

# Limit download speed (useful if throttled)
yt-dlp --cookies-from-browser chrome -o "output.mp4" --limit-rate 2M "URL"

# Download audio only
yt-dlp --cookies-from-browser chrome -f audio -o "lecture.wav" "URL"
```

---

## Method 2: Browser Developer Tools + ffmpeg

If yt-dlp doesn't work for some reason, you can manually find and download the stream.

### Step-by-Step

#### Step 1: Open Developer Tools

1. Navigate to the ON24 video page and **start playing**
2. Press **F12** to open DevTools
3. Click the **Network** tab
4. Click the filter bar and type: `m3u8` OR `mp4`

#### Step 2: Find the Stream URL

1. **Clear** the existing entries (click 🚫 icon)
2. **Refresh the page** and start playing again
3. Look for entries with:
    - Type: `media` or `xhr`
    - Domain: `event.on24.com` or a CDN domain
    - File: ending in `.mp4`, `.m3u8`, or `.ts`

4. **Right-click** the media entry → **Copy** → **Copy URL**

The URL will look something like:

```
https://event.on24.com/media/news/corporatevideo/events/5270653/.../video.mp4
```

#### Step 3: Download with ffmpeg

```bash
ffmpeg -i "PASTED_STREAM_URL" -c copy output.mp4
```

The `-c copy` flag means **no re-encoding** — it just saves the stream as-is, preserving original quality.

For HLS streams (`.m3u8`):

```bash
ffmpeg -i "https://...stream.m3u8" -c copy output.mp4
```

ffmpeg will download all `.ts` segments and combine them into a single MP4.

#### Step 4: If Stream Requires Referrer

Some streams check the `Referer` header. Add it:

```bash
ffmpeg -headers "Referer: https://event.on24.com/" -i "STREAM_URL" -c copy output.mp4
```

---

## Method 3: Download Slides + Audio (Bonus)

ON24 stores slides as images with timestamps. You can extract these and combine them with the audio to create a video. This is useful if you primarily need the slides in high quality.

### Step 1: Get Slide Data from ON24 API

The ON24 API exposes slide metadata at this endpoint:

```
https://event.on24.com/apic/utilApp/EventConsoleCachedServlet?eventId=EVENT_ID&displayProfile=player&key=YOUR_KEY&contentType=A
```

Replace `EVENT_ID` and `YOUR_KEY` with values from your URL.

Open this URL in your browser to see the JSON response.

### Step 2: Understand the JSON Response

```json
{
  "presentationLogInfo": {
    "presentationLog": [
      {
        "ondemandoffset": 1234,  // timestamp in seconds
        "slideurl": "slide_001.jpg"
      }
    ]
  },
  "mediaUrlInfo": {
    "background": "background.png",
    "slides": [
      { "url": "slide_001.jpg" },
      { "url": "slide_002.jpg" }
    ]
  }
}
```

- `presentationLog`: Maps each slide to a timestamp (`ondemandoffset` in seconds)
- `mediaUrlInfo.background`: The background image
- `mediaUrlInfo.slides`: All slide images

Full slide URLs are constructed as:

```
https://event.on24.com/media/news/corporatevideo/events/EVENT_ID/SLIDE_URL
```

### Step 3: Extract Slides + Timestamps with a Script

Save this as `extract_slides.sh` and run in **Git Bash**:

```bash
#!/bin/bash

EVENT_ID="5270653"
KEY="CAEFA4522A10C3E58E300BA9A1347EDF"
BASE_URL="https://event.on24.com/media/news/corporatevideo/events/${EVENT_ID}"
API_URL="https://event.on24.com/apic/utilApp/EventConsoleCachedServlet?eventId=${EVENT_ID}&displayProfile=player&key=${KEY}&contentType=A"

# Create output directory
mkdir -p slides

# Fetch JSON data
curl -s "$API_URL" -o response.json

# Extract background
bg=$(python3 -c "import json; d=json.load(open('response.json')); print(d['mediaUrlInfo']['background'])")
if [ -n "$bg" ]; then
    curl -s "${BASE_URL}/${bg}" -o "slides/background.png"
    echo "Downloaded background: $bg"
fi

# Download each slide
python3 -c "
import json
data = json.load(open('response.json'))
slides = data['mediaUrlInfo']['slides']
for i, slide in enumerate(slides):
    print(f'{i:03d}|{slide[\"url\"]}')
" | while IFS='|' read -r index url; do
    curl -s "${BASE_URL}/${url}" -o "slides/slide_${index}.jpg"
    echo "Downloaded slide $index: $url"
done

# Create timestamps file
python3 -c "
import json
data = json.load(open('response.json'))
logs = data['presentationLogInfo']['presentationLog']
with open('timestamps.txt', 'w') as f:
    for i, log in enumerate(logs):
        offset = log['ondemandoffset']
        f.write(f'slide_{i:03d}.jpg {offset}\n')
"

echo "Done! Slides saved to ./slides/ and timestamps to timestamps.txt"
```

Run it:

```bash
bash extract_slides.sh
```

### Step 4: Combine Slides + Audio into Video

First, download the audio track using yt-dlp:

```bash
yt-dlp --cookies-from-browser chrome -f audio -o "audio.wav" "YOUR_ON24_URL"
```

Then create a video from slides using ffmpeg:

```bash
# Create a video where each slide displays for its duration
# This requires building a concat file from timestamps

python3 << 'EOF'
import json

data = json.load(open('response.json'))
logs = data['presentationLogInfo']['presentationLog']
total_duration = logs[-1]['ondemandoffset'] + 10  # last slide + 10s buffer

with open('slide_durations.txt', 'w') as f:
    for i, log in enumerate(logs):
        start = log['ondemandoffset']
        end = logs[i+1]['ondemandoffset'] if i+1 < len(logs) else total_duration
        duration = end - start
        f.write(f"file 'slides/slide_{i:03d}.jpg'\n")
        f.write(f"duration {duration}\n")
    # Last slide needs to be listed twice for concat to work
    f.write(f"file 'slides/slide_{len(logs)-1:03d}.jpg'\n")
EOF

# Combine slides into video
ffmpeg -f concat -safe 0 -i slide_durations.txt -i audio.wav \
  -c:v libx264 -pix_fmt yuv420p -c:a aac -shortest output_with_slides.mp4
```

---

## Method 4: Screen Recording (Last Resort)

If all else fails (DRM, blocked downloads, etc.), screen recording always works.

### Option A: OBS Studio (Recommended for Screen Recording)

#### Install

```powershell
winget install OBSProject.OBSStudio
```

Or download from: https://obsproject.com/download

#### Setup & Record

1. Open OBS Studio
2. In **Sources** → click **+** → **Display Capture** (to capture entire screen) or **Window Capture** (capture only the browser)
3. For audio: In **Sources** → **+** → **Audio Output Capture** (captures system audio)
4. Click **Settings** → **Output** → Set recording quality to "High Quality, Medium File Size"
5. Click **Start Recording**
6. Play the ON24 video in full screen
7. Click **Stop Recording** when done

> 💡 **Tip:** Set the ON24 player to full screen (click the full screen icon in the player) before recording for best quality.

### Option B: Windows Game Bar (Built-in, No Install)

1. Press **Win + G** to open Game Bar
2. Click the **Record** button (or press **Win + Alt + R**)
3. Play the video
4. Press **Win + Alt + R** again to stop

> ⚠️ **Limitation:** Game Bar only records one window/app at a time and may not capture browser audio correctly. OBS is more reliable.

---

## Troubleshooting

### Common Issues & Solutions

| Symptom | Likely Cause | Solution |
|---------|-------------|----------|
| `HTTP Error 403: Forbidden` | Session expired or cookies missing | Re-login to O'Reilly, re-export cookies |
| `File name too long` | ON24 URL is very long | Use `-o "shortname.mp4"` to set output name |
| `Unsupported URL` | yt-dlp too old | Update: `yt-dlp -U` |
| `ERROR: Unable to extract` | URL format changed | Check yt-dlp is ≥ 2025.03.31 |
| `SSL Certificate error` | Corporate proxy/firewall | Add `--no-check-certificate` |
| Download stuck or very slow | Throttled by ON24 | Add `--limit-rate 2M` to avoid triggering throttle |
| `Could not read cookies` | Browser is open/locked | Close browser before using `--cookies-from-browser` |
| Video quality is low | ON24 adaptive streaming | The stream quality depends on your connection when the URL was generated |

### Getting Help

1. **Check yt-dlp debug output:**

   ```bash
   yt-dlp --verbose --cookies-from-browser chrome "URL"
   ```

2. **Test connectivity:**

   ```bash
   curl -I "https://event.on24.com/"
   ```

3. **Verify cookie validity:**
    - Revisit O'Reilly, play any ON24 video
    - Re-export cookies immediately
    - Try downloading right away

---

## Understanding the ON24 URL

Your ON24 URL contains these key pieces:

| Parameter | Example Value | Purpose |
|-----------|---------------|---------|
| `eventid` | `5270653` | Unique event identifier (7-digit number) |
| `key` | `CAEFA4522A10C3E58E300BA9A1347EDF` | 32-character hex authentication key |
| `sessionid` | `1` | Session number within the event |
| `format` | `fhvideo1` | Media format selector |
| `contenttype` | `A` | Content type identifier |
| `eventuserid` | `829567966` | Your user ID for this event |
| `referrer` | (URL-encoded) | The O'Reilly page that linked here |

**How ON24 Authentication Works:**

The `key` parameter is your ticket in — it's generated when O'Reilly redirects you to ON24. The yt-dlp extractor uses this key to call:

```
https://event.on24.com/apic/utilApp/EventConsoleCachedServlet?eventId=5270653&displayProfile=player&key=CAEFA4522A10C3E58E300BA9A1347EDF&contentType=A
```

This API returns JSON with the actual video URLs, which yt-dlp then downloads.

---

## Quick Reference Card

```bash
# ═══════════════════════════════════════════
# ON24 VIDEO DOWNLOAD — QUICK COMMANDS
# ═══════════════════════════════════════════

# 1. Install tools (PowerShell as Admin)
winget install ffmpeg
winget install yt-dlp.yt-dlp
# Or with pip: pip install yt-dlp

# 2. Update yt-dlp
yt-dlp -U
# Or with pip: pip install --upgrade yt-dlp

# 3. Download video (close browser first!)
yt-dlp --cookies-from-browser chrome -o "lecture.mp4" "ON24_URL"

# 4. Download HIGHEST quality video+audio
yt-dlp --cookies-from-browser chrome -f "bestvideo+bestaudio/best" --merge-output-format mp4 -o "lecture.mp4" "ON24_URL"

# 5. List available formats
yt-dlp --cookies-from-browser chrome -F "ON24_URL"

# 6. Download audio only
yt-dlp --cookies-from-browser chrome -f audio -o "lecture.wav" "ON24_URL"

# 7. Manual stream capture
# F12 → Network → filter mp4 → copy URL → 
ffmpeg -i "STREAM_URL" -c copy output.mp4

# 8. Poor connection workaround
yt-dlp --cookies-from-browser chrome -o "lecture.mp4" --retries 10 --limit-rate 1M "ON24_URL"
```

---

## References

- **YouTube Tutorial:** [How to download recordings of videos from online streaming services (such as ON24)?](https://www.youtube.com/watch?v=U0zTAaBz2Vg)
- **yt-dlp GitHub:** https://github.com/yt-dlp/yt-dlp
- **yt-dlp ON24 Extractor Source:** https://github.com/yt-dlp/yt-dlp/blob/master/yt_dlp/extractor/on24.py
- **ON24 Support PR:** https://github.com/yt-dlp/yt-dlp/pull/12800
- **ffmpeg Official Site:** https://ffmpeg.org/
- **OBS Studio:** https://obsproject.com/
- **O'Reilly Learning Platform:** https://learning.oreilly.com/

---

> 🤖 Guide generated with assistance from Claude Code | 2026-05-30
