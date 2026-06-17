# Lunch Workout Timer — Design Spec

## Overview

A single-page web app that guides users through a 3-session office lunch workout with configurable timers, circular progress visualization, and voice countdown. Hosted on Cloudflare Pages with zero build step.

## Sessions

| # | Name       | Default | Range       | Step |
|---|------------|---------|-------------|------|
| 1 | Plankan    | 2:00    | 0:15 – 5:00 | 15s  |
| 2 | Jägarvila  | 2:00    | 0:15 – 5:00 | 15s  |
| 3 | Push-ups   | 0:30    | 0:10 – 1:00 | 10s  |

Each session's duration is independently configurable before starting that session.

## User Flow

1. User sees session 1 (Plankan) with time selector and Start button
2. User adjusts time if needed, presses Start
3. Timer counts down with voice announcements and circular progress ring filling
4. At zero: voice says "zero, and the session is completed", ring is fully filled
5. User clicks "Next" to advance to session 2 (Jägarvila) — rest happens naturally while they set up
6. Repeat for session 2
7. User clicks "Next" to advance to session 3 (Push-ups)
8. After session 3 completes, "Done" state — workout is finished

## UI Layout

Single screen, vertically centered:

1. **Session pills** — horizontal row of 3 pill indicators showing all sessions, current one highlighted with amber accent. Pills are non-interactive indicators only (no click navigation).
2. **Time selector** — `−` and `+` buttons flanking the duration (visible in Ready state). Min touch target 44x44px for mobile.
3. **Circular progress ring** — large SVG circle (24px stroke), timer text centered inside. Ring fills clockwise as time elapses. Empty = just started, full = session complete.
4. **Action button** — circular amber button below the ring:
   - Ready state: Play icon → starts timer
   - Running state: No pause — timer runs to completion once started
   - Completed state: "Next" text button (or "Restart" on last session to redo entire workout)

## States Per Session

### Ready
- Time selector visible with +/− buttons
- Progress ring empty (track color only)
- Timer shows selected duration
- Start button visible

### Running
- Time selector hidden
- Progress ring filling clockwise
- Timer counting down
- Voice announcements active

### Completed
- Progress ring fully filled (solid amber)
- Timer shows 0:00
- Voice: "zero, and the session is completed"
- "Next" button appears (sessions 1–2) or "Restart" button (session 3, restarts entire workout from session 1)
- Clicking Next/Restart cancels any pending speech utterance

## Voice Announcements

Using the Web Speech API (`speechSynthesis`):

- **Every 30 seconds**: "{time} remaining" (e.g., "one minute thirty seconds remaining")
- **At 15 seconds**: "fifteen seconds"
- **At 10 seconds**: "ten"
- **At 5, 4, 3, 2, 1**: spoken individually
- **At 0**: "zero, and the session is completed"

Voice language: English.

**Timing rules:**
- Announcements trigger when remaining time *crosses* a threshold during countdown (not at the initial value). E.g., a 30-second session does NOT announce "thirty seconds remaining" at the start.
- For sessions shorter than 30s (e.g., push-ups at 20s), announcements start at the 15s mark.
- Before queuing a new utterance, cancel any pending speech via `speechSynthesis.cancel()` to prevent overlap.

**Fallback:** If `speechSynthesis` is unavailable, the app degrades silently — timer and progress ring work normally, voice is simply absent. No error message needed.

## Circular Progress Ring

- SVG-based with two `<circle>` elements: track (background) and progress (foreground)
- Radius sized to be the dominant visual element (~200–240px diameter)
- Track color: `#2a2218`
- Progress color: `#f59e0b` (amber)
- Stroke width: 24px
- Stroke linecap: round
- Rotation: starts at 12 o'clock (SVG rotated -90deg)
- Animation: JS-driven via `requestAnimationFrame` — set `stroke-dashoffset` directly each frame, no CSS transition on this property (they conflict)
- Timer text centered inside the ring using absolute positioning

## Color Palette (Dark Warm)

| Token         | Value     | Usage                          |
|---------------|-----------|--------------------------------|
| bg            | `#1a1512` | Page background                |
| surface       | `#2a2218` | Cards, inactive elements       |
| text-primary  | `#f5f0ea` | Timer, headings                |
| text-muted    | `#8a7a6a` | Inactive session labels        |
| accent        | `#f59e0b` | Active ring, buttons, active pill |
| accent-glow   | `#f59e0b20` | Accent backgrounds (pill bg)  |

## Typography

- Timer: monospace (`'SF Mono', 'Fira Code', 'Courier New', monospace`), ~48–56px, weight 200
- Session pills: system-ui, 13px
- Buttons: system-ui, 14–16px

## Technical Implementation

### Stack
- Single `index.html` file with inline `<style>` and `<script>`
- No dependencies, no build step, no framework
- Web Speech API for voice
- SVG for circular progress
- `requestAnimationFrame` for smooth ring animation
- Wall-clock timer tracking: store `startTime` via `performance.now()`, compute remaining as `duration - (now - startTime)` each frame to avoid drift

### Deployment
- Cloudflare Pages: connect Git repo, serve root directory
- No build command needed — static HTML

### Browser Support
- Modern browsers with `speechSynthesis` support (Chrome, Edge, Safari, Firefox)
- Mobile responsive (flexbox centering, relative sizing)

### File Structure
```
/
├── index.html          # The entire app
├── .gitignore
└── docs/               # Design docs
```

## Out of Scope
- Sound effects / beeps (voice only)
- Workout history / persistence
- Multiple workout routines
- User accounts
- PWA / offline support (can be added later)
