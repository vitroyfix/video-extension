<div align="center">

<img src="public/icons/icon-128.png" width="96" alt="Media Bridge logo">

# Media Bridge — Video Downloader

**A Chromium extension (Manifest V3) for downloading videos from almost any website.**
HLS · MPEG-DASH · direct files · live streams · MediaSource stream capture · cloud upload

![Manifest V3](https://img.shields.io/badge/Manifest-V3-2563eb)
![TypeScript](https://img.shields.io/badge/TypeScript-5-3178c6)
![FFmpeg.wasm](https://img.shields.io/badge/FFmpeg-wasm-007808)
![License: MIT](https://img.shields.io/badge/License-MIT-yellow)
![E2E](https://img.shields.io/badge/e2e-20%20checks-16a34a)

</div>

---

## Why Media Bridge

Most "video downloader" extensions only catch `.mp4` links. Modern sites stream through adaptive manifests that often have no telling URL, or through MediaSource players with a `blob:` source and nothing to grab. Media Bridge combines three detection layers so it works on far more sites:

| Layer | What it catches |
|---|---|
| **Network sniffer** (service worker) | HLS/DASH manifests and progressive files, identified by **URL *and* `Content-Type`**: works for extension-less endpoints like `/api/stream?id=42`. Segments, byte-range slices, ads and tiny previews are filtered out. |
| **DOM observer** (content script) | `<video>`/`<source>` elements, including ones added later by single-page apps. |
| **Stream capture** (MAIN-world hook) | Players that feed `<video>` through Media Source Extensions with **no detectable source at all**. Media Bridge records the media the player loads and muxes it into a file. |

Everything is processed locally in your browser with FFmpeg.wasm. No servers, no accounts, no telemetry.

## Features

### Downloading
- **HLS** master and media playlists with a quality picker, separate audio renditions, and **AES-128** decryption
- **MPEG-DASH** with highest-bandwidth or user-selected video and audio, muxed to MP4
- **Direct files** (MP4, WebM, MOV, MKV, …) through Chrome's download manager
- **Live recording** of HLS and DASH live streams (REC → STOP → merged MP4)
- **Stop & save**: cancel a long download and keep what has been fetched so far
- **Header injection** of `Origin`/`Referer` through `declarativeNetRequest` for CDNs that require them
- Concurrent segment downloads with retries, backoff and a tolerated failure rate

### Works on more sites
- **Content-Type detection**: manifests and videos are found even when the URL has no `.m3u8`, `.mpd` or `.mp4`
- **Early-request replay**: autoplay players often fetch their manifest before content scripts load. Those detections are queued and replayed, so "turn off autoplay and reload" is rarely needed
- **Stream capture** for MediaSource players (see [below](#stream-capture))
- Every frame is scanned, so embedded players inside iframes are detected too

### Fast to use
- **In-page download button**: hover any video to get a button that lists every source detected for that player
- **Right-click menu**: *Download this video*, *Download linked media*, *Capture this stream*, *Download all detected videos*
- **Keyboard shortcuts**: download the best video on the page without opening anything
- **Toolbar badge** showing how many videos were detected in the tab
- **Download all** in the popup for pages with many clips

### After download
- **History** with re-download, copy URL and bulk delete
- **Cloud upload** to Google Drive or any S3-compatible store (AWS S3, Cloudflare R2, Backblaze B2, Wasabi, MinIO), with the S3 secret optionally encrypted at rest (AES-GCM)
- OS notification and auto-reveal of the file when a download finishes

## Installation

> Requires Node.js 18+ and a Chromium-based browser (Chrome, Edge, Brave, Opera, Vivaldi, Chromium).

```bash
git clone https://github.com/vitroyfix/video-extension.git
cd video-extension
npm install
npm run build
```

Then load it:

1. Open `chrome://extensions`
2. Turn on **Developer mode** (top right)
3. Click **Load unpacked** and select the `dist/` folder
4. Pin **Media Bridge** to the toolbar

## Usage

### 1. Detected videos (most sites)
Play the video, then either:
- hover the video and click the **Download** button in its corner, or
- click the toolbar icon (the badge shows how many videos were found) and press **Download** on a card.

For HLS/DASH you can pick a quality first with **Select quality**. Live streams show a **REC** button.

### 2. Paste a link
Use the popup's **Manifest** tab to paste any `.m3u8`, `.mpd` or direct video URL, load its qualities and download.

### 3. Stream capture
For players that hide their source (the popup says "No videos detected" while the video plays):

1. Hover the video → **Download** → **Fast capture from start**, or use the **Stream capture** card in the popup.
2. Media Bridge restarts the video muted at up to 16× speed and records everything the player loads.
3. When it reaches the end, click **Stop & save**. The tracks are merged into a single file in the background.

Use **Capture from here** to record from the current position at normal speed instead (useful when a player rejects high playback rates).

### Keyboard shortcuts

| Shortcut | Action |
|---|---|
| `Alt+Shift+M` | Open the Media Bridge popup |
| `Alt+Shift+D` | Download the best video detected on the current page |
| *(unassigned)* | Download all detected videos. Bind it at `chrome://extensions/shortcuts` (suggested `Alt+Shift+A`) |

All shortcuts can be changed at `chrome://extensions/shortcuts`.

## Stream capture

Stream capture is the feature that makes "any website" realistic, so here is exactly what it does and doesn't do.

**How it works**

```
 page (MAIN world)                      content script            capture bridge (extension page)   service worker / offscreen
 ┌──────────────────────────────┐       ┌──────────────┐          ┌──────────────────────────┐       ┌───────────────────────┐
 │ SourceBuffer.appendBuffer()  │       │              │ Message- │ writes chunks to the     │ token │ FFmpeg.wasm muxes     │
 │  ├─ keep latest init segment │──────▶│ capture      │ Channel  │ extension's IndexedDB    │──────▶│ video + audio → MP4 / │
 │  └─ when armed: keep media   │ post- │ client       │─────────▶│ (zero-copy transfers)    │       │ WebM / MKV, then save │
 │     segments keyed by decode │ Msg   │              │          └──────────────────────────┘       └───────────────────────┘
 │     time (tfdt / Timecode)   │       └──────────────┘
 └──────────────────────────────┘
```

- The hook is installed at `document_start`, before any player code runs, and only wraps `addSourceBuffer`, `appendBuffer` and `createObjectURL`. It never alters what the player receives.
- Segments are keyed by their **decode timestamp** (`tfdt` for fragmented MP4, cluster `Timecode` for WebM). Seeking back and re-buffering doesn't duplicate content, and out-of-order appends are sorted back into timeline order.
- When the player switches quality mid-stream, the rendition with the most captured data is kept.
- Buffers move between contexts with transferable `ArrayBuffer`s, so hundreds of MB aren't JSON-serialized.
- The bridge is authorized with a one-time token that the page never sees. If the bridge can't be used, the tracks are saved as separate files so a capture is never lost.
- The output container follows the codecs: H.264/AAC → **MP4**, VP9/AV1/Opus → **WebM**, mixed → **MKV**.

**Limits**

- **DRM is respected.** If the element has EME `MediaKeys`, or the init segment signals encryption (`encv`/`enca`/`tenc`, WebM `ContentEncryption`), the session is marked **DRM-protected**, nothing is retained, and saving is refused.
- Capture records what the player *loads*. Play to the end, use **Fast capture**, or seek through the parts you want. Gaps you skip stay gaps.
- Lock the player to one quality for the cleanest result.
- Players that run MediaSource inside a Web Worker aren't covered by the main-thread hook.
- Captured data lives in page memory until saved. The per-page cap is configurable (default 1.5 GB).

## Settings

Open **Settings** from the popup (gear icon).

| Section | Options |
|---|---|
| **Download Settings** | FFmpeg timeout, max concurrent segment downloads, history on/off |
| **Cloud Providers** | Google Drive (your own OAuth client ID), S3-compatible storage, CORS helper |
| **Recording** | Live poll interval bounds and cadence |
| **Page Integration** | In-page button on/off, minimum size for sniffed direct media, capture memory limit, auto-capture on playback |
| **Notifications** | OS notification on completion, auto-reveal file |
| **Advanced** | Retries, backoff, fragment failure tolerance, detection cache sizes, IndexedDB sync interval |

### Cloud upload setup

<details>
<summary><b>Google Drive</b></summary>

1. In [Google Cloud Console](https://console.cloud.google.com/), create or select a project and enable the **Google Drive API**.
2. **Credentials → Create credentials → OAuth client ID**, type **Web application**.
3. Add the redirect URI shown in **Settings → Cloud Providers → Google Drive** (looks like `https://<extension-id>.chromiumapp.org/`).
4. Paste the client ID into Settings and click **Sign in with Google**.

If prompted for a consent screen, choose **External**, fill the required fields and add yourself as a test user. Testing mode is enough for personal use.
</details>

<details>
<summary><b>S3 / S3-compatible</b></summary>

1. Enter bucket, region, access key ID and secret (and endpoint for non-AWS providers).
2. Click **Copy CORS Config** and paste it into your bucket's CORS settings. Without it the browser blocks uploads from the extension origin.
3. Optionally set a passphrase to encrypt the secret key at rest.
</details>

## Architecture

```
src/
├── service-worker.ts          # Orchestrator: downloads, messages, uploads, keep-alive
├── background/
│   ├── network-sniffer.ts     # URL + Content-Type classification, segment filtering, early-request replay
│   ├── badge.ts               # Per-tab detected-video counter
│   ├── page-actions.ts        # Context menus, keyboard shortcuts, download-all / best
│   └── capture-pipeline.ts    # Capture token check → FFmpeg mux → save
├── inject/
│   └── mse-capture.ts         # MAIN-world MediaSource hook (document_start, dependency-free)
├── content.ts                 # Isolated-world content script (detection entry point)
├── content/
│   ├── capture-client.ts      # Page ⇄ extension capture hand-off, bridge transfer, raw fallback
│   └── overlay.ts             # In-page download button (closed shadow DOM)
├── capture-bridge/            # Web-accessible page that writes captured chunks to IndexedDB
├── core/
│   ├── detection/             # Direct / HLS / DASH detection handlers
│   ├── downloader/            # Direct, HLS, M3U8, DASH and live-recording handlers
│   ├── parsers/               # m3u8-parser / mpd-parser wrappers
│   ├── ffmpeg/                # Offscreen document bridge
│   ├── database/              # IndexedDB: download state + raw chunks
│   ├── cloud/                 # Google Drive + S3 providers
│   └── storage/               # Typed settings (loadSettings) + secure storage
├── offscreen/                 # FFmpeg.wasm processing (serialized job queue)
├── popup/                     # Videos / Manifest / Downloads tabs, capture panel, download-all
└── options/                   # Settings + history
```

**Execution contexts:** service worker · isolated content script · MAIN-world hook · offscreen document (FFmpeg) · capture bridge iframe · popup · options page. They communicate over `chrome.runtime` messaging. Large binary data goes through IndexedDB, never JSON messages.

**Download flow (HLS/DASH):** parse manifest → fetch segments concurrently into IndexedDB → offscreen FFmpeg concatenates and muxes → `chrome.downloads` saves the result.

See [`CLAUDE.md`](CLAUDE.md) for deeper implementation notes: progress throttling, header injection, partial saves and more.

## Development

```bash
npm run dev          # rebuild on change (reload the extension in chrome://extensions)
npm run build        # production build → dist/
npm run type-check   # TypeScript only
```

### End-to-end tests

A real-browser suite loads `dist/` into Chromium, serves synthetic media locally and verifies:

- HLS detected from `Content-Type` alone, plus badge count
- Extension-less MP4 detected (including size from headers and early-request replay)
- Download-all → HLS downloaded and muxed, probed with `ffprobe` for audio+video and duration
- MSE capture with out-of-order **and duplicate** appends → deduped, ordered, muxed MP4 with both tracks
- EME/ClearKey session flagged as DRM, nothing retained, save refused
- In-page overlay injected on hover
- Output files named after the page title

```bash
npm run build
tests/e2e/generate-media.sh          # needs ffmpeg
CHROME_PATH=/path/to/chromium npm run test:e2e
```

## Limitations

- **DRM-protected content** (Widevine, PlayReady, FairPlay) can't be downloaded or captured. This is intentional.
- **~2 GB output**: muxing runs in browser memory. Very long or 4K videos may exceed it. Pick a lower quality.
- **Token/IP-locked CDNs** may reject requests made outside the player. Stream capture usually still works because it uses what the player already loaded.
- **Platform-specific players** (e.g. YouTube's UMP protocol) don't expose manifests. Use stream capture there.

## Troubleshooting

| Problem | Try |
|---|---|
| Popup says *No videos detected* | Start playback and skip ads. Then use **Force detection**, or **Stream capture** from the in-page button. |
| Download fails immediately | The stream may be DRM-protected or the CDN may require the page session. Try stream capture. |
| Merge step fails or times out | Raise the FFmpeg timeout in **Download Settings**, or pick a lower quality (≈2 GB limit). |
| Captured file has gaps | Capture only records what played. Use **Fast capture from start** and let it reach the end. |
| No in-page button | Check **Settings → Page Integration**. Fullscreen players hide page overlays, so exit fullscreen. |
| `Alt+Shift+A` does nothing | It's unassigned by default. Set it at `chrome://extensions/shortcuts`. |

## Legal

Media Bridge is a tool for saving media you have the right to save: your own uploads, openly licensed content, or material whose terms permit downloading. It doesn't bypass DRM and refuses to capture DRM-protected streams. You are responsible for complying with copyright law and each website's terms of service.

## Credits

Built on [**jvillegasd/media-bridge**](https://github.com/jvillegasd/media-bridge) (MIT). That project contributed the HLS/DASH/direct download engine, live recording, FFmpeg.wasm pipeline, history and cloud upload. This fork adds Content-Type network detection with early-request replay, MediaSource stream capture with DRM refusal, the in-page download button, context menus, keyboard shortcuts, badge counts, download-all, container-aware muxing, an FFmpeg loading race fix and an end-to-end test suite.

Powered by [FFmpeg.wasm](https://github.com/ffmpegwasm/ffmpeg.wasm), [m3u8-parser](https://github.com/videojs/m3u8-parser), [mpd-parser](https://github.com/videojs/mpd-parser) and [idb](https://github.com/jakearchibald/idb).

## License

[MIT](LICENSE). Original copyright notice retained.
