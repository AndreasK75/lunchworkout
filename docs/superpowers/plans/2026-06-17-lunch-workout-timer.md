# Lunch Workout Timer Implementation Plan

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a single-page workout timer app with 3 configurable sessions, circular progress ring, and voice countdown.

**Architecture:** Single `index.html` with inline CSS and JS. No dependencies, no build step. State machine drives session flow (Ready → Running → Completed). `requestAnimationFrame` loop with wall-clock tracking for timer precision. Web Speech API for voice.

**Tech Stack:** Vanilla HTML/CSS/JS, SVG, Web Speech API, Cloudflare Pages

**Spec:** `docs/superpowers/specs/2026-06-17-lunch-workout-timer-design.md`

---

## Chunk 1: Static Shell and Session Navigation

### Task 1: HTML structure and CSS foundation

**Files:**
- Create: `index.html`

- [ ] **Step 1: Create index.html with full CSS and static HTML structure**

Create `index.html` with:
- `<!DOCTYPE html>`, viewport meta, title "Lunch Workout"
- Inline `<style>` with all CSS:
  - CSS custom properties for the color palette: `--bg: #1a1512`, `--surface: #2a2218`, `--text-primary: #f5f0ea`, `--text-muted: #8a7a6a`, `--accent: #f59e0b`, `--accent-glow: rgba(245,158,11,0.125)`
  - Body: dark warm background, flexbox centering, `min-height: 100vh`, `margin: 0`, `font-family: system-ui`
  - `.app` container: centered column, max-width ~400px, padding
  - `.session-pills`: horizontal flex row, gap 8px, centered
  - `.pill`: padding 8px 20px, border-radius 24px, background `var(--surface)`, color `var(--text-muted)`, font-size 15px, cursor default
  - `.pill.active`: background `var(--accent-glow)`, border 1px solid `var(--accent)`, color `var(--accent)`
  - `.ring-container`: relative positioned, width/height 260px, margin 32px auto
  - SVG circle styles for track and progress
  - `.timer-display`: absolute centered inside ring, monospace font stack (`'SF Mono', 'Fira Code', 'Courier New', monospace`), font-size 52px, weight 200, color `var(--text-primary)`
  - `.time-selector`: flex row with −/+ buttons and duration text, gap 16px, centered
  - `.time-selector button`: width/height 52px, border-radius 50%, background `var(--surface)`, color `var(--text-primary)`, font-size 22px, border none, cursor pointer, min touch target 44px
  - `.action-btn`: width 64px, height 64px, border-radius 50%, background `var(--accent)`, border none, cursor pointer, margin 24px auto, display flex, align-items center, justify-content center
  - `.action-btn.next-btn`: width auto, padding 16px 40px, border-radius 32px, font-size 18px, color `var(--bg)`, font-weight 600
  - `.hidden`: display none
  - Mobile responsive: already uses relative sizing and flexbox, ensure no horizontal overflow

HTML body structure:
```html
<div class="app">
  <div class="session-pills">
    <div class="pill active" id="pill-0">Plankan</div>
    <div class="pill" id="pill-1">Jägarvila</div>
    <div class="pill" id="pill-2">Push-ups</div>
  </div>

  <div class="ring-container">
    <svg viewBox="0 0 260 260" width="260" height="260">
      <circle class="ring-track" cx="130" cy="130" r="106" />
      <circle class="ring-progress" id="ring-progress" cx="130" cy="130" r="106" />
    </svg>
    <div class="timer-display" id="timer-display">2:00</div>
  </div>

  <div class="time-selector" id="time-selector">
    <button id="btn-minus">−</button>
    <span id="duration-label">2:00</span>
    <button id="btn-plus">+</button>
  </div>

  <button class="action-btn" id="action-btn">
    <svg width="24" height="24" viewBox="0 0 24 24" fill="var(--bg)">
      <polygon points="6,3 20,12 6,21" />
    </svg>
  </button>
</div>
```

SVG circle styles:
```css
.ring-track {
  fill: none;
  stroke: var(--surface);
  stroke-width: 24;
}
.ring-progress {
  fill: none;
  stroke: var(--accent);
  stroke-width: 24;
  stroke-linecap: round;
  stroke-dasharray: 666.11; /* 2 * PI * 106 */
  stroke-dashoffset: 666.11; /* fully empty */
  transform: rotate(-90deg);
  transform-origin: center;
}
```

- [ ] **Step 2: Verify in browser**

Run: `open index.html` (or use a local server)
Expected: Dark warm page with 3 session pills (Plankan highlighted), empty progress ring showing "2:00", −/+ buttons, and a play button. Mobile-friendly layout.

- [ ] **Step 3: Commit**

```bash
echo ".superpowers/" > .gitignore
git add index.html .gitignore
git commit -m "feat: add static HTML shell with CSS and layout"
```

---

### Task 2: Session state machine and time selection

**Files:**
- Modify: `index.html` (add `<script>` block)

- [ ] **Step 1: Add JavaScript state machine and session data**

Add an inline `<script>` at the end of `<body>` with:

```javascript
const sessions = [
  { name: 'Plankan', default: 120, min: 15, max: 300, step: 15 },
  { name: 'Jägarvila', default: 120, min: 15, max: 300, step: 15 },
  { name: 'Push-ups', default: 30, min: 10, max: 60, step: 10 },
];

let currentSession = 0;
let durations = sessions.map(s => s.default); // user-selected durations in seconds
let state = 'ready'; // 'ready' | 'running' | 'completed'
```

DOM references:
```javascript
const pills = [document.getElementById('pill-0'), document.getElementById('pill-1'), document.getElementById('pill-2')];
const timerDisplay = document.getElementById('timer-display');
const durationLabel = document.getElementById('duration-label');
const timeSelector = document.getElementById('time-selector');
const actionBtn = document.getElementById('action-btn');
const btnMinus = document.getElementById('btn-minus');
const btnPlus = document.getElementById('btn-plus');
const ringProgress = document.getElementById('ring-progress');
const CIRCUMFERENCE = 2 * Math.PI * 106; // ~666.11
```

Helper to format seconds as `M:SS`:
```javascript
function formatTime(seconds) {
  const m = Math.floor(seconds / 60);
  const s = seconds % 60;
  return `${m}:${String(s).padStart(2, '0')}`;
}
```

`renderSession()` function that updates all UI based on `currentSession` and `state`:

```javascript
const playIcon = '<svg width="24" height="24" viewBox="0 0 24 24" fill="var(--bg)"><polygon points="6,3 20,12 6,21"/></svg>';

function renderSession() {
  // Cancel any in-flight animation
  if (animFrameId) {
    cancelAnimationFrame(animFrameId);
    animFrameId = null;
  }

  // Update pills
  pills.forEach((pill, i) => pill.classList.toggle('active', i === currentSession));

  // Update timer display and duration label
  const dur = durations[currentSession];
  durationLabel.textContent = formatTime(dur);

  if (state === 'ready') {
    timerDisplay.textContent = formatTime(dur);
    timeSelector.classList.remove('hidden');
    ringProgress.style.strokeDashoffset = CIRCUMFERENCE;
    actionBtn.className = 'action-btn';
    actionBtn.innerHTML = playIcon;
    actionBtn.classList.remove('hidden');
  } else if (state === 'running') {
    timeSelector.classList.add('hidden');
    actionBtn.classList.add('hidden');
  } else if (state === 'completed') {
    timeSelector.classList.add('hidden');
    const label = currentSession < 2 ? 'Next' : 'Restart';
    actionBtn.className = 'action-btn next-btn';
    actionBtn.textContent = label;
    actionBtn.classList.remove('hidden');
  }
}
```

- [ ] **Step 2: Wire up +/− buttons**

```javascript
btnMinus.addEventListener('click', () => {
  const s = sessions[currentSession];
  durations[currentSession] = Math.max(s.min, durations[currentSession] - s.step);
  renderSession();
});

btnPlus.addEventListener('click', () => {
  const s = sessions[currentSession];
  durations[currentSession] = Math.min(s.max, durations[currentSession] + s.step);
  renderSession();
});
```

- [ ] **Step 3: Wire up action button for session transitions**

```javascript
actionBtn.addEventListener('click', () => {
  if (state === 'ready') {
    startTimer();
  } else if (state === 'completed') {
    if (currentSession < 2) {
      currentSession++;
      state = 'ready';
      renderSession();
    } else {
      // Restart entire workout
      currentSession = 0;
      durations = sessions.map(s => s.default);
      state = 'ready';
      renderSession();
    }
    if (window.speechSynthesis) speechSynthesis.cancel();
  }
});
```

Add a placeholder `startTimer()` that just sets state to 'completed' for now:
```javascript
function startTimer() {
  state = 'running';
  renderSession();
  // Timer logic added in next task
  setTimeout(() => { state = 'completed'; renderSession(); }, 1000);
}
```

Call `renderSession()` on page load.

- [ ] **Step 4: Verify in browser**

Expected: Can click +/− to adjust time. Play button starts (briefly goes to completed). "Next" advances sessions. "Restart" on last session resets to beginning.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat: add session state machine and time selection"
```

---

## Chunk 2: Timer Engine and Progress Ring

### Task 3: Countdown timer with progress ring animation

**Files:**
- Modify: `index.html` (replace placeholder `startTimer()`)

- [ ] **Step 1: Implement wall-clock timer with requestAnimationFrame**

Replace the placeholder `startTimer()` with:

```javascript
let startTime = null;
let totalDuration = 0;
let animFrameId = null;
let lastAnnouncedThreshold = null;

function startTimer() {
  state = 'running';
  totalDuration = durations[currentSession];
  startTime = performance.now();
  lastAnnouncedThreshold = null;
  renderSession();
  tick();
}

function tick() {
  const elapsed = (performance.now() - startTime) / 1000;
  const remaining = Math.max(0, totalDuration - elapsed);
  const progress = 1 - (remaining / totalDuration);

  // Update timer display (ceiling so it shows 0:01 until truly 0)
  const displaySeconds = Math.ceil(remaining);
  timerDisplay.textContent = formatTime(displaySeconds);

  // Update ring
  ringProgress.style.strokeDashoffset = CIRCUMFERENCE * (1 - progress);

  // Voice announcements
  announceTime(remaining, displaySeconds);

  if (remaining <= 0) {
    // Session complete
    timerDisplay.textContent = '0:00';
    ringProgress.style.strokeDashoffset = 0;
    state = 'completed';
    renderSession();
    return;
  }

  animFrameId = requestAnimationFrame(tick);
}
```

- [ ] **Step 2: Verify in browser**

Start a session with a short time (e.g., 15 seconds). Expected: timer counts down smoothly, ring fills up, reaches 0:00 and stops, "Next" button appears.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: add countdown timer with progress ring animation"
```

---

### Task 4: Voice announcements

**Files:**
- Modify: `index.html` (add `speak()` and `announceTime()` functions)

- [ ] **Step 1: Implement speech helper and announcement logic**

```javascript
function speak(text) {
  if (!window.speechSynthesis) return;
  speechSynthesis.cancel();
  const utterance = new SpeechSynthesisUtterance(text);
  utterance.lang = 'en-US';
  utterance.rate = 1.0;
  speechSynthesis.speak(utterance);
}

function timeToWords(seconds) {
  const m = Math.floor(seconds / 60);
  const s = seconds % 60;
  const parts = [];
  if (m > 0) parts.push(m === 1 ? 'one minute' : `${m} minutes`);
  if (s > 0) parts.push(`${s} seconds`);
  return parts.join(' ') + ' remaining';
}

function announceTime(remaining, displaySeconds) {
  // Determine which threshold we just crossed
  let threshold = null;

  if (displaySeconds <= 0) {
    threshold = 0;
  } else if (displaySeconds <= 5) {
    threshold = displaySeconds; // 5,4,3,2,1
  } else if (displaySeconds <= 10 && displaySeconds === 10) {
    threshold = 10;
  } else if (displaySeconds <= 15 && displaySeconds === 15) {
    threshold = 15;
  } else if (displaySeconds % 30 === 0 && displaySeconds > 15) {
    threshold = displaySeconds;
  }

  // Only announce each threshold once, and only on crossing (not at start)
  if (threshold === null) return;
  if (threshold === lastAnnouncedThreshold) return;
  if (threshold === totalDuration) return; // Don't announce at start

  lastAnnouncedThreshold = threshold;

  if (threshold === 0) {
    if (currentSession === 2) {
      speak('zero, Well done everybody!');
    } else {
      speak('zero, and the session is completed');
    }
  } else if (threshold <= 5) {
    speak(String(threshold));
  } else if (threshold === 10) {
    speak('ten');
  } else if (threshold === 15) {
    speak('fifteen seconds');
  } else {
    speak(timeToWords(threshold));
  }
}
```

- [ ] **Step 2: Verify in browser**

Start a 30-second session. Expected voice announcements at: 15s ("fifteen seconds"), 10s ("ten"), 5-4-3-2-1, 0 ("zero, and the session is completed"). No announcement at 30s since that's the start value.

Start a 2:00 session. Expected: "one minute thirty seconds remaining" at 1:30, "one minute remaining" at 1:00, "thirty seconds remaining" at 0:30, then 15, 10, 5-1, 0.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: add voice countdown announcements"
```

---

## Chunk 3: Polish and Deploy

### Task 5: UI polish and mobile optimization

**Files:**
- Modify: `index.html`

- [ ] **Step 1: Add visual polish**

Add to CSS:
- Smooth state transitions: fade time-selector in/out with opacity transition
- Active button states: `.time-selector button:active { background: var(--accent); color: var(--bg); }`
- Add `user-select: none` and `-webkit-tap-highlight-color: transparent` on `.app` for mobile
- Action button hover/active: subtle scale transform
- Ensure action button in completed state shows text "Next" (sessions 0-1) or "Restart" (session 2) with `.next-btn` class applied
- Add a subtle text-shadow or glow to the timer display when running
- Completed state: pill for completed sessions gets a subtle checkmark or different style (e.g., muted accent color)

- [ ] **Step 2: Ensure mobile-first responsiveness**

Add/verify:
- `<meta name="viewport" content="width=device-width, initial-scale=1.0">`
- Ring container scales: use `max(200px, min(260px, 70vw))` for width/height
- Touch targets: all buttons minimum 44x44px (already specified in CSS, verify)
- No horizontal scroll on small screens
- Test at 375px width (iPhone SE) and 390px (iPhone 14)

- [ ] **Step 3: Verify on mobile and desktop**

Open in browser, use device toolbar to check various screen sizes. Expected: layout stays centered, buttons are large and tappable, timer is readable.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: add UI polish and mobile responsiveness"
```

---

### Task 6: Final integration and deployment readiness

**Files:**
- Modify: `index.html` (if any final tweaks)

- [ ] **Step 1: Full manual walkthrough**

Complete the entire workout flow:
1. Session 1 (Plankan): set to 0:15, start, verify countdown + voice + ring
2. Click Next → Session 2 (Jägarvila): verify time selector resets to default 2:00, set to 0:15, start
3. Click Next → Session 3 (Push-ups): verify default 0:30, set to 0:10, start
4. Verify final voice says "zero, Well done everybody!"
5. Click Restart → verify returns to session 1 with defaults

- [ ] **Step 2: Test edge cases**

- Set session to minimum time (15s for sessions 1-2, 10s for session 3)
- Verify voice announcements work correctly for short durations
- Rapidly click +/− buttons — verify duration stays in range
- Click Next immediately after completion — verify speech cancellation

- [ ] **Step 3: Commit final version**

```bash
git add -A
git commit -m "feat: lunch workout timer ready for deployment"
```

- [ ] **Step 4: Document deployment**

The repo is ready for Cloudflare Pages:
1. Connect repo to Cloudflare Pages
2. Build command: (none)
3. Output directory: `/` (root)
4. Deploy
