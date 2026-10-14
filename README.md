# Elgato Cam Link → streaming protocols

`camlink-stream` captures the Cam Link (video + its USB audio) with ffmpeg, encodes H.264
(NVENC → VAAPI → x264, whichever works first) and either serves it locally over every
common protocol or pushes it to a remote ingest.

Requires `ffmpeg` and `v4l2-ctl` (package `v4l-utils`).

## Serve locally (RTSP, RTMP, SRT, HLS, WebRTC at once)

    ./camlink-stream serve

| Protocol | URL                                        | Use it in                         |
|----------|--------------------------------------------|-----------------------------------|
| RTSP     | `rtsp://<ip>:8554/camlink`                 | VLC, ffplay, NVRs, OBS Media Source |
| RTMP     | `rtmp://<ip>:1936/camlink`                 | OBS Media Source, Restreamer input |
| SRT      | `srt://<ip>:8890?streamid=read:camlink`    | OBS, vMix, ffplay                 |
| HLS      | `http://<ip>:8888/camlink/`                | Any browser, phones               |
| WebRTC   | `http://<ip>:8889/camlink/`                | Browser, sub-second latency       |

RTMP is on **1936**, not 1935, because the `restreamer` Docker container already uses 1935.
Override with `RTMP_PORT=... ./camlink-stream serve`.

The stream carries two audio tracks (Opus for WebRTC, AAC for RTMP); each protocol gets the
one it supports. The server is powered by MediaMTX v1.21.2, which the script downloads into
`mediamtx/` on first run (x86-64 Linux). It listens on all
interfaces without authentication, so anyone on your LAN can watch. Add `authInternalUsers`
in `mediamtx/mediamtx.yml` if that matters.

## Push to a platform or remote server

    ./camlink-stream push rtmp://live.twitch.tv/app/<stream-key>
    ./camlink-stream push rtmps://a.rtmps.youtube.com/live2/<stream-key>
    ./camlink-stream push "srt://host:port?streamid=...&pkt_size=1316"

To feed your Restreamer container instead, push to `rtmp://127.0.0.1:1935/...` using the
RTMP path/token shown in Restreamer's UI.

## Options

    --size 3840x2160 --fps 30   capture mode (default: whatever the HDMI source sends)
    --bitrate 8M                video bitrate
    --encoder libx264           force an encoder
    --no-audio                  video only
    --device /dev/videoN        skip auto-detection
    --test                      synthetic test pattern (no Cam Link needed)

## Troubleshooting

- **Not detected** (`v4l2-ctl --list-devices` shows no "Cam Link"): use a USB 3 port
  directly on the machine (not a USB 2 hub); check `journalctl -k | grep -i 0fd9` after plugging in.
- **Stream is black and silent, ffmpeg shows a large `drop=` count**: the Cam Link has no
  HDMI signal. With no input it outputs black 1080p at ~92 fps, and ffmpeg drops the extra
  frames to keep 59.94. Check the source is powered on, outputting HDMI (not mirroring only to
  its own screen) and not HDCP-protected (game consoles: disable HDCP; Apple TV/streaming
  apps can't be captured).
- **Supported modes** depend on the HDMI source: `v4l2-ctl -d /dev/videoN --list-formats-ext`.
  Make `--size/--fps` match what the source sends.
- **"Device or resource busy"**: OBS or another app has the Cam Link open; close it.
- **Garbled/green image or wrong format on Cam Link 4K**: update its firmware with Elgato's
  Camera Hub / Firmware Updater (on a Windows/macOS machine).
