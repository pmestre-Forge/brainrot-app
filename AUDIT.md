# rotfeed — index.html Audit Report

**Date:** 2026-03-27
**Commit base:** `ec3c1bc` (Strip to core) + syntax fix commit
**File:** `index.html` (~590 lines, single-file SPA)

---

## 1. ARCHITECTURE

The entire app lives in one `index.html` file with three sections:

| Section | Lines | Purpose |
|---------|-------|---------|
| **HTML** | 1-265 | `<head>` with meta/PWA tags, inline `<style>`, `<body>` with DOM skeleton |
| **CSS** | 18-230 | ~212 lines of inline styles — layout, animations, portal UI, chaos overlay, responsive breakpoints |
| **JS** | 266-589 | ~320 lines of inline `<script>` — data, rendering, scroll, portal/chaos engine, keyboard shortcuts |

**No external JS dependencies.** Only external resources are Google Fonts (Space Grotesk) and picsum.photos for images.

**PWA support:** manifest.json link, apple-touch-icon, theme-color, display-mode media query — but no service worker registered in code.

---

## 2. RENDERING PIPELINE

```
P (array of 160 prompt/category objects)
  |
  v
sh(P) -> sf (shuffled copy)
  |
  v
gf() filters by active category (ac) -> returns array
  |
  v
lb2() loads next BS(5) items from filtered array
  |
  v
cc(item, index) builds a <div class="cd"> card:
  1. Creates <canvas> element
  2. Appends innerHTML overlay (rarity badge, watermark, caption bar)
  3. requestAnimationFrame -> either:
     a. _lazyObs exists: observe card for deferred render
     b. else: immediate imgMemeRender(canvas, prompt, category, seed)
  |
  v
imgMemeRender(canvas, text, mood, seed):
  1. 256x256 canvas
  2. Mood-based gradient fill (fallback to "chaotic" palette)
  3. Loads picsum.photos image via seed-based URL
  4. img.onload: draws image, applies dark overlay gradient + radial vignette
  5. Pixel-level film grain noise via seeded PRNG
  6. (NOTE: _memeText() exists but is NEVER called — see Dead Code)
```

**User-created cards** follow a parallel path:
```
doCollide() -> apiCollide()/demoCollide() -> createChaosCard(collision, words, mood)
  -> builds card DOM, calls imgMemeRender(), prepends to feed
```

---

## 3. SCROLL SYSTEM

**Infinite scroll** is driven by an `IntersectionObserver` watching the `#ld` (loader) div at the bottom of the feed:

```
IntersectionObserver (rootMargin: '600px')
  -> observes #ld element
  -> when intersecting: debounced lb2() call (150ms clearTimeout)
  -> lb2() appends next 5 cards, increments lc (load cursor)
  -> staggered .v class (opacity/transform animation) at 80ms intervals
```

**Scroll progress bar:** `window.scroll` listener updates `#sp` width as percentage of page scrolled.

**Category switching** (`#cb` click handler):
1. Resets `lc=0`, clears feed innerHTML
2. Re-shuffles P array
3. Re-initializes render queue (`_rQ`, `_rN`) and `_lazyObs`
4. Calls `lb2()` twice (immediate + 200ms delay)

**Edge case:** When filtered results exhaust (s >= its.length), `lb2()` returns early — the loader spinner stays visible forever. No "end of feed" state.

---

## 4. IMAGE SYSTEM

### imgMemeRender(canvas, text, mood, seed) — Line 318

**Canvas setup:** Fixed 256x256px (CSS scales to full card width via `width:100%`).

**Background:** Linear gradient using mood-to-color map:
- unhinged: `#ff2d55` -> `#ff6b00`
- cursed: `#1a0030` -> `#4a0080`
- dreamy: `#ff9a9e` -> `#fad0c4`
- angry: `#ff0000` -> `#990000`
- cosmic: `#0f0c29` -> `#302b63`
- chaotic: `#00ff87` -> `#60efff`

**IMPORTANT:** The mood key used is the **category string** from the P array (e.g., "corporate horror", "crime documentary"), but `moodColors` only has keys like "unhinged", "cursed", etc. **No P-array category matches any moodColors key**, so every card from the main feed falls through to `moodColors.chaotic` (green/cyan). Only user-created cards using `selectedMood` (default "unhinged") will hit a real match.

**Image source:** `https://picsum.photos/seed/rf{seed%9999}/512/512` — deterministic seed means same prompt always gets same image. Loaded at 512px, drawn to 256px canvas (downscaled).

**Post-processing layers (in img.onload):**
1. Vertical dark gradient overlay (transparent top -> 30% black bottom)
2. Radial vignette (transparent center -> 25% black edges)
3. Film grain: pixel-by-pixel noise using seeded PRNG (`sr(seed+1)`), +/-4 per channel. Wrapped in try/catch for CORS taint.

**img.onerror:** Empty handler — canvas shows just the gradient on failure.

---

## 5. LLM HOOKS

### Current State

| Component | Status | Location |
|-----------|--------|----------|
| `WORKER_URL` | Empty string `''` | Line 268 |
| `/api/collide` endpoint | Referenced but no backend exists | Line 475 |
| `apiCollide(words, mood)` | **Fully implemented** — POSTs to `WORKER_URL + '/api/collide'`, falls back to `demoCollide()` on error | Lines 473-485 |
| `demoCollide(words, mood)` | **Fully implemented** — local template-based sentence generation using `CURSED_TEMPLATES` | Lines 466-471 |
| `CURSED_TEMPLATES` | **Complete** — 6 mood categories, 8-10 templates each, `{1}/{2}/{3}` word substitution | Lines 397-463 |

### What's Missing for LLM Integration

1. **Set `WORKER_URL`** to a Cloudflare Worker (or any backend) URL
2. **Build the `/api/collide` endpoint** that accepts `{words: string[], mood: string}` and returns `{fragments: string[], collision: string, mood: string}`
3. **No streaming support** — `apiCollide` does a single fetch, expects complete JSON response
4. **No auth/rate limiting** — the fetch has no headers beyond Content-Type
5. **No API key management** — nothing client-side for key storage

### Flow When WORKER_URL is Set
```
doCollide() checks WORKER_URL truthy
  -> true:  apiCollide() -> POST /api/collide -> returns {collision, fragments, mood}
  -> false: 1500ms fake delay -> demoCollide() -> local template fill
```

The `demoCollide()` fallback is solid — the app works fully offline with template-based generation.

---

## 6. DEAD CODE

### Confirmed Dead Code

| Item | Line | Issue |
|------|------|-------|
| `_memeText(c, text, W, H)` | 300-317 | **Never called anywhere.** Was likely used by stripped render functions. Draws Impact-font meme text with stroke outline. |
| `visualStyle` variable | 354 | Computed in `createChaosCard()` (`Math.floor(Math.random()*3)`) but **never read**. Was meant to select between canvas art/meme/glitch styles before the other renderers were stripped. |
| Comment "Glitch-style canvas: scanlines, RGB split, fragmented text" | 339 | **Orphaned section header** — the glitch renderer was stripped, only the comment remains. |
| `lt` variable | 273 | Declared (`let lt=0`), **never read or written** after init. |
| `lk` variable (Set) | 273 | Declared (`let lk=new Set()`), **never read or written** after init. Was likely a "liked posts" tracker. |
| `.dth` element | 294 | Injected into every card's innerHTML (heart emoji div), but **no CSS class `.dth`** exists — invisible, does nothing. |
| `.rc` CSS rule | 44 | `display:none` — targets a class never used in JS or HTML. |
| CSS `.cp` `min-height:280px` | 36 | Category pills have `min-height:280px` which is clearly wrong for pill buttons — likely a paste error from the strip. Makes pills absurdly tall. |
| `.ib` CSS / install button | 32, 229 | CSS exists for an install button (`.ib`), but **no HTML element** with class `ib` exists in the DOM. |
| `#to` toast element | 264 | The `<div class="to" id="to">` exists in HTML with CSS, but **nothing in JS ever shows it** (no `.sh` class toggle). |

### CSS with No Matching JS/HTML

- `.ab`, `.ab.lk`, `.ac` — action button styles (like button) — no elements created
- `.hb` keyframe animation — never triggered
- The double-tap heart animation infrastructure (`.dth`, `.hb` keyframe) is vestigial

---

## 7. PERFORMANCE — Render Queue & Lazy Loading

### Render Queue (`_rQ`, `_rN`, `_drQ`)

The queue is **only initialized inside the category-switch click handler** (line 375). On initial page load, `_rQ`, `_rN`, and `_drQ` do not exist.

**Initial load path (no queue):** `cc()` -> `requestAnimationFrame` -> checks `window._lazyObs`:
- If `_lazyObs` exists: defers render via IntersectionObserver
- If not: **renders immediately** (every card on first load renders eagerly)

**After first category switch:** `_rQ=[]`, `_rN=0`, `_drQ` defined, `_lazyObs` created. From this point forward, cards are lazy-rendered one at a time with 150ms gaps.

**Bug:** On initial page load, all 5 cards fire `imgMemeRender` simultaneously (5 concurrent picsum.photos fetches + canvas draws). The render queue only kicks in after the user switches categories at least once. This means the first-load experience is the heaviest.

### Lazy IntersectionObserver

- `rootMargin: '200px'` — starts rendering cards 200px before they scroll into view
- Marks canvas with `data-rendered='1'` to prevent double-render
- Correctly unobserves after rendering
- **Only exists after category switch** (see above)

### Scroll Observer (infinite load)

- `rootMargin: '600px'` — triggers `lb2()` well before user reaches bottom
- Debounced at 150ms via `clearTimeout`/`setTimeout`
- `BS=5` cards per batch with 80ms staggered animation
- **Working correctly**

### Potential Performance Issues

1. **No render queue on first load** — all initial cards render eagerly
2. **`innerHTML +=` in `cc()`** (line 294) — destroys and recreates the canvas element that was just appended (line 293). The `requestAnimationFrame` callback queries it again, which works, but the original `cv` reference is orphaned.
3. **`dataset.item = JSON.stringify(it)`** — stores full prompt object as a string attribute on every card DOM node. Parsed back with `JSON.parse` on category switch. Works but is wasteful for 160+ items.
4. **256px canvas** — good tradeoff for performance. CSS scales it up, keeping the "lo-fi" aesthetic.
5. **picsum.photos dependency** — if the CDN is slow or down, `img.onerror` silently fails and cards show only the gradient. No retry logic.

---

## Summary of Issues by Severity

### Crashes (Fixed)
- ~~`await` outside async function~~ -- FIXED
- ~~Orphaned `m.dataset.m` reference~~ -- FIXED
- ~~Missing `collideBtn` and `cfab` HTML elements~~ -- FIXED

### Bugs (Remaining)
- `.cp` pills have `min-height:280px` (should be removed or ~28px)
- `moodColors` keys don't match P-array categories, so all feed cards get "chaotic" palette
- No render queue on initial page load (eager rendering)
- `innerHTML +=` orphans the initially created canvas reference

### Dead Code (Cleanup Candidates)
- `_memeText()` function — 18 lines, never called
- `visualStyle`, `lt`, `lk` variables — declared, never used
- `.dth`, `.rc`, `.ib`, `.ab`, `.ac`, `.hb` CSS — no matching elements
- `#to` toast — HTML exists, never shown
- Glitch renderer section comment — empty

### Missing for Production
- `WORKER_URL` backend
- Service worker for PWA offline support
- End-of-feed state when content exhausts
- Error/loading states for picsum.photos failures
