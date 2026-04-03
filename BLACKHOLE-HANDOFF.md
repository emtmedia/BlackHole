# BlackHole — Project Handoff for Claude Code

## What This Project Is

BlackHole is a PWA video downloader app. The user copies a video URL (YouTube, Instagram, TikTok, Twitter/X, Facebook, etc.), opens the app, taps a central "black hole" button, and the app reads the clipboard, sends the URL to a cobalt API backend, and triggers a download.

## Architecture

```
┌────────────────────────┐          ┌──────────────────────┐
│   PWA Frontend         │  HTTPS   │   Cobalt API         │
│   GitHub Pages (free)  │─────────▶│   Railway (free)     │
│   Reads clipboard      │          │   Proxies & serves   │
│   Triggers download    │◀─────────│   video files        │
└────────────────────────┘          └──────────────────────┘
```

- **Frontend**: Pure static HTML/CSS/JS PWA — single `index.html` + manifest + service worker + icons
- **Backend**: Self-hosted [cobalt](https://github.com/imputnet/cobalt) instance deployed on Railway (free tier)
- **Hosting**: GitHub Pages for the frontend (free HTTPS via `username.github.io`)
- **No build tools, no frameworks, no dependencies**

## Current State — What Has Been Built

### Frontend (COMPLETE — ready to deploy)

All frontend files exist and are functional. The user has a zip file containing:

```
frontend/
├── index.html        # The entire app (HTML + CSS + JS in one file)
├── manifest.json     # PWA manifest for "Add to Home Screen"
├── sw.js             # Service Worker for offline caching
├── icon-192.png      # PWA icon (192x192, generated via Pillow)
└── icon-512.png      # PWA icon (512x512, generated via Pillow)
```

#### Frontend Tech Details

- **Fonts**: Orbitron (titles) + Chakra Petch (body) from Google Fonts
- **Aesthetic**: Dark space theme, purple (#6c3ce0 / #8b5cf6) accent, orbiting rings around a black hole button, starfield background
- **Features implemented**:
  - Clipboard API reading (with manual paste fallback for when clipboard is blocked)
  - Format selector: VIDEO (downloadMode: "auto") / AUDIO (downloadMode: "audio")
  - Quality selector: 360p / 720p / 1080p / MAX (maps to cobalt's `videoQuality` field)
  - Audio format selector: MP3 / OGG / WAV / BEST (shown when AUDIO mode selected)
  - "Sucking" animation on the black hole button during download
  - Download history stored in localStorage
  - Settings panel for configuring Cobalt API URL and optional Authorization header
  - Picker modal for multi-item posts (Instagram carousels, etc.)
  - Responsive design for small screens

#### How the Frontend Talks to Cobalt

The frontend sends a `POST` request to the cobalt API root endpoint:

```javascript
fetch(apiUrl, {
  method: 'POST',
  headers: {
    'Accept': 'application/json',
    'Content-Type': 'application/json',
    // Optional: 'Authorization': 'Api-Key xxx' or 'Bearer xxx'
  },
  body: JSON.stringify({
    url: 'https://www.youtube.com/watch?v=...',
    downloadMode: 'auto',       // 'auto' | 'audio' | 'mute'
    videoQuality: '1080',       // '144'...'4320' | 'max'
    audioFormat: 'mp3',         // 'best' | 'mp3' | 'ogg' | 'wav' | 'opus'
    filenameStyle: 'pretty'     // 'classic' | 'pretty' | 'basic' | 'nerdy'
  })
})
```

Cobalt responds with one of:

```json
// Success — single file
{ "status": "tunnel", "url": "https://...", "filename": "video.mp4" }
// or
{ "status": "redirect", "url": "https://direct-source-url..." }

// Success — multiple items (carousel)
{ "status": "picker", "picker": [{type, url, thumb}, ...], "audio": "optional-url" }

// Error
{ "status": "error", "error": { "code": "error.code.here" } }
```

The frontend opens the `url` in a new tab to trigger the browser's native download.

### Backend (NOT yet deployed — user needs to do this)

The backend is **not custom code** — it's the official cobalt Docker image. The user needs to:

1. Deploy `ghcr.io/imputnet/cobalt:11` on Railway via one-click: https://railway.com/deploy/cobalt-media-downloader
2. Ensure `CORS_WILDCARD=1` is set (the Railway template does this automatically)
3. Copy the public Railway URL
4. Paste it into the PWA's Settings

### Deployment Status

| Component | Status | Where |
|-----------|--------|-------|
| Frontend code | ✅ Complete | User has zip file |
| Frontend hosting | ❌ Not deployed | Needs GitHub repo + Pages enabled |
| Backend code | ✅ Uses official cobalt | No custom code needed |
| Backend hosting | ❌ Not deployed | Needs Railway one-click deploy |
| Connection | ❌ Not configured | User pastes Railway URL in PWA Settings |

## What Might Need To Be Done Next

Here are likely next tasks the user may ask for:

### 1. GitHub Pages Subfolder Fix

If the repo is named anything other than `username.github.io`, the site lives at `/reponame/`. The manifest and service worker paths may need updating:

In `manifest.json`, change:
```json
"start_url": "/blackhole/"
```

In `index.html`, change the SW registration:
```javascript
navigator.serviceWorker.register('/blackhole/sw.js')
```

In `sw.js`, update the ASSETS array:
```javascript
const ASSETS = [
  '/blackhole/',
  '/blackhole/index.html',
  '/blackhole/manifest.json',
  '/blackhole/icon-192.png',
  '/blackhole/icon-512.png',
];
```

### 2. Better Icons

The current icons were auto-generated with Python Pillow — they're functional but basic (purple ring on black). The user may want proper designed icons.

### 3. Railway Configuration

If the user wants to protect their cobalt instance with an API key, they need to set env vars on Railway:
- `API_AUTH_REQUIRED=1`
- API key configuration per cobalt docs: https://github.com/imputnet/cobalt/blob/main/docs/protect-an-instance.md

### 4. YouTube Cookie Support

YouTube aggressively blocks cobalt. To improve reliability, the user can add a `cookies.json` to the Railway instance with YouTube login cookies. See cobalt docs for format.

### 5. Custom Domain

Instead of `username.github.io/blackhole`, the user could configure a custom domain in GitHub Pages settings (e.g., `blackhole.eltonmiranda.com`).

### 6. iOS-Specific Improvements

- Haptic feedback on button tap (not available in PWA, but could add visual feedback)
- Share Sheet integration (receiving shared URLs — requires a native wrapper or Shortcuts integration)
- The clipboard API sometimes fails on iOS Safari even with HTTPS — the manual paste fallback handles this

### 7. Future: Self-Hosted Backend

The user is planning to build a home server (Intel N100 mini PC + OpenMediaVault + Cloudflare Tunnel). When that's ready, cobalt can run locally:

```bash
docker run -d \
  --name cobalt \
  -p 9000:9000 \
  -e API_URL="https://cobalt.his-domain.com/" \
  -e CORS_WILDCARD=1 \
  ghcr.io/imputnet/cobalt:11
```

Then just update the API URL in the PWA Settings.

## Key Technical Notes

- **Cobalt API docs**: https://github.com/imputnet/cobalt/blob/main/docs/api.md
- **Cobalt's official `api.cobalt.tools` has bot protection (Turnstile)** and is NOT meant for third-party apps. The user MUST deploy their own instance.
- **CORS is critical**: The cobalt instance must have `CORS_WILDCARD=1` or explicitly allow the GitHub Pages origin.
- **HTTPS is mandatory**: The Clipboard API requires a secure context. GitHub Pages provides this. Railway provides this.
- **The entire frontend is a single HTML file** with inline CSS and JS. No build process. Edit `index.html` directly.
- **Settings are stored in localStorage**: `bh_api_url`, `bh_api_key`, `bh_history`
- **Service Worker caches the app shell** for offline launch (the app opens offline, but obviously needs network to download videos)

## File Locations If Working From the Zip

After unzipping `blackhole-pwa.zip`:

```
blackhole-pwa/
├── frontend/
│   ├── index.html
│   ├── manifest.json
│   ├── sw.js
│   ├── icon-192.png
│   └── icon-512.png
└── README.md
```

Only the files inside `frontend/` go into the GitHub repo root. The `README.md` is for reference.

## User Context

- The user is Elton Miranda, based in Minas Gerais, Brazil
- He is an Innovation Manager who works with web development, AI, and data projects
- He is comfortable with Docker, GitHub, and command-line tools
- He uses Claude Code for development
- His primary use case is downloading videos on his iPhone from various social platforms
- He was inspired by an iOS app called "Black Hole" that has a similar one-tap download UX
