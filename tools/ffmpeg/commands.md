
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
