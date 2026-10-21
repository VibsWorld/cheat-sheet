
## Convert a single file format from `.webm` to `.mp4`
```bash
ffmpeg -i Swayam-Wizard-PaymentDemo-20261021-070504.webm -c:v libx264 -crf 20 -pix_fmt yuv420p swayam_demo_output.mp4

# Medium Quality
ffmpeg -i input.webm -c:v libx264 -crf 20 -preset medium -pix_fmt yuv420p -c:a aac -movflags +faststart output.mp4

# Higher Quality
# If you want to prioritize visual quality and don't mind a slightly larger file size, use a slower preset and a lower CRF:
ffmpeg -i input.webm -c:v libx264 -crf 18 -preset slow -c:a aac -b:a 192k output.mp4

```
• `-c:v libx264`: This explicitly converts your VP8 video stream into H.264, which is the global standard for MP4 files.
• `-crf 20`: This maintains high visual quality (lower means better quality, 20 is a great sweet spot).
• `-pix_fmt yuv420p`: WebM files often use pixel formats that standard media players (like Windows Media Player or QuickTime) can't read. This flag ensures maximum playback compatibility.

## Convert multiple WebM files
### Windows
```
for %i in (*.webm) do ffmpeg -i "%i" -c:v libx264 -crf 20 -c:a aac "%~ni.mp4"
```
### Linux / Bash
```
for i in *.webm; do ffmpeg -i "$i" -c:v libx264 -crf 20 -c:a aac "${i%.*}.mp4"; done
```

## Useful FFmpeg variations

### Optimize for web upload / faster processing
```bash
ffmpeg -i input.webm -c:v libx264 -crf 28 -preset veryfast -c:a aac -movflags +faststart output.mp4
```
Use this when you want a smaller file and faster conversion, especially for web uploads or social media.

### Resize to 720p while converting
```bash
ffmpeg -i input.webm -vf "scale=1280:720:force_original_aspect_ratio=decrease,pad=1280:720:(ow-iw)/2:(oh-ih)/2" -c:v libx264 -crf 23 -c:a aac output.mp4
```
This keeps the video within a standard 720p display size while preserving the original aspect ratio.

### Convert to MP4 with audio only
```bash
ffmpeg -i input.webm -vn -c:a aac -movflags +faststart audio-only.mp4
```
Useful when you only need the soundtrack and want a lightweight MP4 container.

### Batch convert all files into a `converted/` folder
#### Windows
```bat
mkdir converted
for %i in (*.webm) do ffmpeg -i "%i" -c:v libx264 -crf 20 -preset medium -c:a aac -movflags +faststart "converted\%~ni.mp4"
```

#### Linux / Bash
```bash
mkdir -p converted
for i in *.webm; do ffmpeg -i "$i" -c:v libx264 -crf 20 -preset medium -c:a aac -movflags +faststart "converted/${i%.*}.mp4"; done
```

### Force compatibility with older players
```bash
ffmpeg -i input.webm -c:v libx264 -crf 20 -pix_fmt yuv420p -c:a aac -movflags +faststart -profile:v high -level 4.0 output.mp4
```
This helps with older devices, some browsers, and playback software that is stricter about codec/profile compatibility.

### Quick notes
- `veryfast` = faster conversion, larger size
- `medium` = balanced quality and speed
- `slow` = better quality, slower conversion
- Lower CRF values = higher quality
- `-movflags +faststart` helps video begin playing quickly in browsers and streaming apps
