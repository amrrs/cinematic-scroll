# Cinematic Scroll Experience

Turn a video file into a scroll-driven animated website with animation variety and choreography.

## Input

Provide a video file path (MP4, MOV, etc.) and optionally:
- Theme/brand name
- Desired text sections and placement
- Color scheme
- Design direction

Use sensible creative defaults for anything not specified.

## Requirements

FFmpeg and FFprobe must be on PATH. Verify with `which ffmpeg` (Unix) or `where ffmpeg` (Windows).

---

## Premium Checklist (Non-Negotiable)

1. **Lenis smooth scroll** — native scroll feels "web page," Lenis feels "experience"
2. **4+ animation types** — never repeat the same entrance animation consecutively
3. **Staggered reveals** — label → heading → body → CTA, never all at once
4. **No glassmorphism cards** — hierarchy via font size/weight/color only
5. **Direction variety** — sections enter from different directions (left, right, up, scale, clip)
6. **Dark overlay for stats** — 0.88–0.92 opacity, counters animate up, only time center text is OK
7. **Horizontal text marquee** — at least one oversized sliding element (12vw+)
8. **Counter animations** — all numbers count up from 0, never appear statically
9. **Massive typography** — hero 12rem+, headings 4rem+, marquee 10vw+
10. **CTA persists** — `data-persist="true"` keeps final section visible
11. **Hero prominence** — hero gets 20%+ scroll range, 800vh+ total for 6 sections
12. **Side-aligned text only** — all text in outer 40% zones (`align-left`/`align-right`); exception: stats with full dark overlay
13. **Circle-wipe hero reveal** — hero is standalone 100vh, canvas reveals via `clip-path: circle()` as hero scrolls away
14. **Frame speed 1.8–2.2** — product animation completes by ~55% scroll

---

## Step 1: Analyze the Video

```bash
ffprobe -v error -select_streams v:0 \
  -show_entries stream=width,height,duration,r_frame_rate,nb_frames \
  -of csv=p=0 "<VIDEO_PATH>"
```

Decide:
- **Frame count**: 150–300 total
  - <10s → original fps, cap at 300
  - 10–30s → 10–15fps
  - 30s+ → 5–10fps
- **Resolution**: match aspect ratio, cap width at 1920px

---

## Step 2: Extract Frames

```bash
mkdir -p frames
ffmpeg -i "<VIDEO_PATH>" \
  -vf "fps=<FPS>,scale=<WIDTH>:-1" \
  -c:v libwebp -quality 80 \
  "frames/frame_%04d.webp"
```

Count extracted frames: `ls frames/ | wc -l`

---

## Step 3: Project Structure

```
project-root/
  index.html
  css/style.css
  js/app.js
  frames/frame_0001.webp ...
```

No bundler. Vanilla HTML/CSS/JS + CDN libraries.

---

## Step 4: index.html

Required structure (in order):

```html
<!-- 1. Loader -->
<div id="loader">
  <div class="loader-brand"></div>
  <div id="loader-bar"></div>
  <div id="loader-percent"></div>
</div>

<!-- 2. Fixed header -->
<header class="site-header">
  <nav><!-- logo + links --></nav>
</header>

<!-- 3. Hero (standalone 100vh) -->
<section class="hero-standalone">
  <span class="section-label"></span>
  <h1 class="hero-heading"><!-- words in <span>s --></h1>
  <p class="hero-tagline"></p>
  <!-- scroll indicator arrow -->
</section>

<!-- 4. Canvas -->
<div class="canvas-wrap">
  <canvas id="canvas"></canvas>
</div>

<!-- 5. Dark overlay -->
<div id="dark-overlay"></div>

<!-- 6. Marquee -->
<div class="marquee-wrap">
  <div class="marquee-text"></div>
</div>

<!-- 7. Scroll container (800vh+) -->
<div id="scroll-container">
  <!-- content sections -->
  <!-- stats section -->
  <!-- CTA section (data-persist="true") -->
</div>
```

Content section:
```html
<section class="scroll-section section-content align-left"
         data-enter="22" data-leave="38" data-animation="slide-left">
  <div class="section-inner">
    <span class="section-label">002 / Feature</span>
    <h2 class="section-heading">Headline</h2>
    <p class="section-body">Description.</p>
  </div>
</section>
```

Stats section:
```html
<section class="scroll-section section-stats"
         data-enter="54" data-leave="72" data-animation="stagger-up">
  <div class="stats-grid">
    <div class="stat">
      <span class="stat-number" data-value="24" data-decimals="0">0</span>
      <span class="stat-suffix">hrs</span>
      <span class="stat-label">Cold retention</span>
    </div>
  </div>
</section>
```

CDN scripts (end of body, this order):
```html
<script src="https://cdn.jsdelivr.net/npm/lenis@1/dist/lenis.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/gsap@3/dist/gsap.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/gsap@3/dist/ScrollTrigger.min.js"></script>
<script src="js/app.js"></script>
```

---

## Step 5: css/style.css

```css
:root {
  --bg-light: #f5f3f0;
  --bg-dark: #111111;
  --text-on-light: #1a1a1a;
  --text-on-dark: #f0ede8;
  --font-display: '[DISPLAY FONT]', sans-serif;
  --font-body: '[BODY FONT]', sans-serif;
}

/* Side-aligned text zones */
.align-left  { padding-left: 5vw;  padding-right: 55vw; }
.align-right { padding-left: 55vw; padding-right: 5vw;  }
.align-left  .section-inner,
.align-right .section-inner { max-width: 40vw; }
```

Key rules:
- Hero is standalone 100vh with solid bg. Canvas starts hidden, reveals via circle-wipe.
- Scroll sections: `position: absolute` within scroll container, positioned at midpoint of enter/leave range with `transform: translateY(-50%)`.
- Body text minimum `#666` on light backgrounds; never `#999`.
- Mobile (<768px): collapse side alignment to centered text with dark backdrop overlays, reduce scroll height to ~550vh.

---

## Step 6: js/app.js

### 6a. Lenis Smooth Scroll

```js
const lenis = new Lenis({
  duration: 1.2,
  easing: (t) => Math.min(1, 1.001 - Math.pow(2, -10 * t)),
  smoothWheel: true
});
lenis.on("scroll", ScrollTrigger.update);
gsap.ticker.add((time) => lenis.raf(time * 1000));
gsap.ticker.lagSmoothing(0);
```

### 6b. Frame Preloader

Two-phase: load first 10 frames immediately (fast first paint), then load remaining in background. Show progress bar. Hide loader only after all frames are ready.

### 6c. Canvas Renderer — Padded Cover Mode

```js
const IMAGE_SCALE = 0.85; // 0.82–0.90 sweet spot

function drawFrame(index) {
  const img = frames[index];
  if (!img) return;
  const cw = canvas.width, ch = canvas.height;
  const iw = img.naturalWidth, ih = img.naturalHeight;
  const scale = Math.max(cw / iw, ch / ih) * IMAGE_SCALE;
  const dw = iw * scale, dh = ih * scale;
  const dx = (cw - dw) / 2, dy = (ch - dh) / 2;
  ctx.fillStyle = bgColor; // sampled from frame corners
  ctx.fillRect(0, 0, cw, ch);
  ctx.drawImage(img, dx, dy, dw, dh);
}
```

- Sample background color from frame edge pixels every ~20 frames with `sampleBgColor()`
- Apply `devicePixelRatio` scaling for crisp rendering

### 6d. Frame-to-Scroll Binding

```js
const FRAME_SPEED = 2.0; // 1.8–2.2

ScrollTrigger.create({
  trigger: scrollContainer,
  start: "top top",
  end: "bottom bottom",
  scrub: true,
  onUpdate: (self) => {
    const accelerated = Math.min(self.progress * FRAME_SPEED, 1);
    const index = Math.min(Math.floor(accelerated * FRAME_COUNT), FRAME_COUNT - 1);
    if (index !== currentFrame) {
      currentFrame = index;
      requestAnimationFrame(() => drawFrame(currentFrame));
    }
  }
});
```

### 6e. Section Animation System

```js
function setupSectionAnimation(section) {
  const type    = section.dataset.animation;
  const persist = section.dataset.persist === "true";
  const enter   = parseFloat(section.dataset.enter) / 100;
  const leave   = parseFloat(section.dataset.leave) / 100;
  const children = section.querySelectorAll(
    ".section-label, .section-heading, .section-body, .section-note, .cta-button, .stat"
  );

  const tl = gsap.timeline({ paused: true });

  switch (type) {
    case "fade-up":
      tl.from(children, { y: 50, opacity: 0, stagger: 0.12, duration: 0.9, ease: "power3.out" }); break;
    case "slide-left":
      tl.from(children, { x: -80, opacity: 0, stagger: 0.14, duration: 0.9, ease: "power3.out" }); break;
    case "slide-right":
      tl.from(children, { x: 80, opacity: 0, stagger: 0.14, duration: 0.9, ease: "power3.out" }); break;
    case "scale-up":
      tl.from(children, { scale: 0.85, opacity: 0, stagger: 0.12, duration: 1.0, ease: "power2.out" }); break;
    case "rotate-in":
      tl.from(children, { y: 40, rotation: 3, opacity: 0, stagger: 0.1, duration: 0.9, ease: "power3.out" }); break;
    case "stagger-up":
      tl.from(children, { y: 60, opacity: 0, stagger: 0.15, duration: 0.8, ease: "power3.out" }); break;
    case "clip-reveal":
      tl.from(children, { clipPath: "inset(100% 0 0 0)", opacity: 0, stagger: 0.15, duration: 1.2, ease: "power4.inOut" }); break;
  }

  // Play/reverse on scroll. If persist=true, never reverse past the leave point.
}
```

### 6f. Counter Animations

```js
document.querySelectorAll(".stat-number").forEach(el => {
  const target   = parseFloat(el.dataset.value);
  const decimals = parseInt(el.dataset.decimals || "0");
  gsap.from(el, {
    textContent: 0,
    duration: 2,
    ease: "power1.out",
    snap: { textContent: decimals === 0 ? 1 : 0.01 },
    scrollTrigger: {
      trigger: el.closest(".scroll-section"),
      start: "top 70%",
      toggleActions: "play none none reverse"
    }
  });
});
```

### 6g. Horizontal Text Marquee

```js
document.querySelectorAll(".marquee-wrap").forEach(el => {
  const speed = parseFloat(el.dataset.scrollSpeed) || -25;
  gsap.to(el.querySelector(".marquee-text"), {
    xPercent: speed,
    ease: "none",
    scrollTrigger: {
      trigger: scrollContainer,
      start: "top top",
      end: "bottom bottom",
      scrub: true
    }
  });
  // Fade in/out via opacity transitions based on scroll range
});
```

### 6h. Dark Overlay

```js
function initDarkOverlay(enter, leave) {
  const overlay   = document.getElementById("dark-overlay");
  const fadeRange = 0.04;
  ScrollTrigger.create({
    trigger: scrollContainer,
    start: "top top",
    end: "bottom bottom",
    scrub: true,
    onUpdate: (self) => {
      const p = self.progress;
      let opacity = 0;
      if      (p >= enter - fadeRange && p <= enter)          opacity = (p - (enter - fadeRange)) / fadeRange;
      else if (p > enter && p < leave)                        opacity = 0.9;
      else if (p >= leave && p <= leave + fadeRange)          opacity = 0.9 * (1 - (p - leave) / fadeRange);
      overlay.style.opacity = opacity;
    }
  });
}
```

### 6i. Circle-Wipe Hero Reveal

```js
function initHeroTransition() {
  ScrollTrigger.create({
    trigger: scrollContainer,
    start: "top top",
    end: "bottom bottom",
    scrub: true,
    onUpdate: (self) => {
      const p = self.progress;
      heroSection.style.opacity = Math.max(0, 1 - p * 15);
      const wipeProgress = Math.min(1, Math.max(0, (p - 0.01) / 0.06));
      const radius = wipeProgress * 75;
      canvasWrap.style.clipPath = `circle(${radius}% at 50% 50%)`;
    }
  });
}
```

---

## Step 7: Test

1. Serve locally: `npx serve .` or `python -m http.server 8000`
2. Scroll through fully — verify each section has a **different** animation type
3. Confirm: smooth scroll, frame playback, staggered reveals, marquee slides, counters count up, dark overlay fades, CTA persists

---

## Animation Types Reference

| Type | From | Duration |
|------|------|----------|
| `fade-up` | y:50, opacity:0 | 0.9s, power3.out |
| `slide-left` | x:-80, opacity:0 | 0.9s, power3.out |
| `slide-right` | x:80, opacity:0 | 0.9s, power3.out |
| `scale-up` | scale:0.85, opacity:0 | 1.0s, power2.out |
| `rotate-in` | y:40, rotation:3, opacity:0 | 0.9s, power3.out |
| `stagger-up` | y:60, opacity:0 | 0.8s, power3.out |
| `clip-reveal` | clipPath:inset(100% 0 0 0) | 1.2s, power4.inOut |

All types use stagger 0.1–0.15s.

## Clip-Path Variations

```
Circle reveal:    circle(0% at 50% 50%)  →  circle(75% at 50% 50%)
Wipe from left:   inset(0 100% 0 0)      →  inset(0 0% 0 0)
Wipe from bottom: inset(100% 0 0 0)      →  inset(0% 0 0 0)
Polygon wipe:     polygon(50% 0%, 50% 0%, 50% 100%, 50% 100%)  →  polygon(0% 0%, 100% 0%, 100% 100%, 0% 100%)
```

---

## Anti-Patterns

- **Cycling feature cards in a pinned section** — give each feature its own section (8–10% scroll range) with its own animation
- **`IMAGE_SCALE` at 1.0** — product clips into header; use 0.82–0.90
- **Pure contain mode** — leaves a visible border that won't match page bg
- **`FRAME_SPEED` < 1.8** — animation feels sluggish
- **Hero < 20% scroll range** — first impression needs breathing room
- **Same animation for consecutive sections** — always vary entrance type
- **Wide centered grids over canvas** — use vertical lists in the 40% side zone
- **Scroll height < 800vh for 6 sections** — everything feels rushed

---

## Troubleshooting

| Issue | Fix |
|-------|-----|
| Frames not loading | Must serve via HTTP, not `file://` |
| Choppy scrolling | Increase `scrub` value, reduce frame count |
| White flashes | Ensure all frames loaded before hiding loader |
| Blurry canvas | Apply `devicePixelRatio` scaling to canvas dimensions |
| Lenis conflicts | Ensure `lenis.on("scroll", ScrollTrigger.update)` is connected |
| Counters not animating | Verify `data-value` exists and snap matches decimal places |
| Memory issues on mobile | Reduce frames to <150, resize to 1280px wide |
| FFmpeg not found | `brew install ffmpeg` / `apt install ffmpeg` / ffmpeg.org |
