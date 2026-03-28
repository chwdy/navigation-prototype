# Navigation Prototype — Implementation Plan
## CS6750 HCI Project: Combined Lane-Aware + Simplified Direction Design

---

## Overview

Build a single self-contained `index.html` file deployable to GitHub Pages with no build step, no npm, no server. The page simulates a mobile navigation UI across 5 driving phases, controlled by a bottom slider. Everything — HTML, CSS, SVG drawing logic, animations — lives in one file.

---

## File Structure

```
index.html         ← entire project, single file
```

No folders, no assets, no dependencies except one Google Fonts CDN link.

---

## Aesthetic Direction

**Theme: Dark cockpit / heads-up display.** Night-mode navigation feel. Dark road surfaces (`#111318`), pure white dashed lane markers, a vivid teal route path (`#00C8A0`), and an amber accent (`#F5A623`) for warnings and lane-change prompts. The car is a white-filled triangle with a subtle glow. Typography uses `DM Mono` for the banner distance readout (gives a digital instrument feel) and `Inter` for secondary labels. The overall effect should feel like a real navigation app screenshot, not a wireframe.

---

## HTML Structure

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <!-- meta, Google Fonts: DM Mono + Inter -->
  <!-- inline <style> block -->
</head>
<body>

  <!-- 1. HEADER BAR (persistent) -->
  <div id="header-bar">
    <div id="turn-icon"></div>        <!-- SVG arrow, updates per phase -->
    <div id="instruction-text"></div> <!-- e.g. "Move to right lane" -->
    <div id="distance-readout"></div> <!-- e.g. "1 mi" in DM Mono -->
  </div>

  <!-- 2. MAP CANVAS (fills screen between header and slider) -->
  <div id="map-canvas">
    <svg id="scene-svg" width="100%" height="100%"></svg>
  </div>

  <!-- 3. BOTTOM PANEL (persistent) -->
  <div id="bottom-panel">
    <!-- ETA bar -->
    <div id="eta-bar">
      <span id="eta-time">3:11</span>
      <span id="eta-dist">· · ·</span>
    </div>

    <!-- Phase slider -->
    <div id="slider-track">
      <input type="range" id="phase-slider" min="0" max="4" step="1" value="0">
      <div id="phase-labels">
        <span data-phase="0">1 mi ahead</span>
        <span data-phase="1">500 ft</span>
        <span data-phase="2">At exit</span>
        <span data-phase="3">Complex turn</span>
        <span data-phase="4">After turn</span>
      </div>
    </div>
  </div>

</body>
</html>
```

---

## CSS Specification

### Layout
- `body`: `display: flex; flex-direction: column; height: 100vh; overflow: hidden; background: #111318; color: #fff; font-family: 'Inter', sans-serif;`
- `#header-bar`: `height: 72px; flex-shrink: 0; display: flex; align-items: center; padding: 0 20px; gap: 12px; background: #1A1D24; border-bottom: 1px solid rgba(255,255,255,0.08);`
- `#map-canvas`: `flex: 1; position: relative; overflow: hidden;`
- `#bottom-panel`: `height: 110px; flex-shrink: 0; background: #1A1D24; border-top: 1px solid rgba(255,255,255,0.08); padding: 12px 20px;`

### Header bar elements
- `#turn-icon`: `width: 40px; height: 40px;` — contains inline SVG arrow, swapped per phase
- `#instruction-text`: `font-size: 16px; font-weight: 600; flex: 1;` — primary action label
- `#distance-readout`: `font-family: 'DM Mono', monospace; font-size: 22px; color: #00C8A0;`

### Phase labels
- Default: `font-size: 11px; color: rgba(255,255,255,0.35); cursor: pointer; transition: color 0.2s;`
- Active: `color: #00C8A0; font-weight: 600;`
- Layout: `display: flex; justify-content: space-between;` beneath the slider

### Slider styling (CSS custom range)
- Track: `height: 3px; background: rgba(255,255,255,0.15); border-radius: 2px;`
- Track fill (before thumb): use a CSS linear-gradient trick to fill left of thumb in `#00C8A0`
- Thumb: `width: 18px; height: 18px; border-radius: 50%; background: #00C8A0; border: 2px solid #111318; cursor: pointer;`
- Remove default browser styles with `-webkit-appearance: none`

### ETA bar
- `font-size: 13px; color: rgba(255,255,255,0.5); margin-bottom: 10px;`
- `#eta-time`: `color: #fff; font-weight: 600;`

### Lane change banner (phases 1 only)
- Position: absolute, top of `#map-canvas`, full width
- `background: rgba(245, 166, 35, 0.15); border-bottom: 1px solid rgba(245,166,35,0.4);`
- `color: #F5A623; font-size: 13px; font-weight: 600; padding: 6px 16px;`
- Text: "● Move to right lane in 0.5 mi"

---

## SVG Scene System

All road drawing happens by calling `renderPhase(n)` which clears `#scene-svg` and draws the appropriate scene. Use JavaScript to write SVG elements via `document.createElementNS`.

### Helper functions to implement

```javascript
// Create any SVG element with attributes
function svgEl(tag, attrs) { ... }

// Append element to scene
function add(el) { document.getElementById('scene-svg').appendChild(el); }

// Get canvas dimensions
function W() { return scene.clientWidth; }
function H() { return scene.clientHeight; }

// Clear scene
function clearScene() { scene.innerHTML = ''; }
```

### Perspective road drawing function

Used for phases 0, 1, 2, 4 (highway perspective view):

```javascript
function drawPerspectiveRoad(laneCount, vanishX, vanishY, opts) {
  // vanishX, vanishY: vanishing point coords (typically W()/2, H()*0.35)
  // laneCount: number of lanes (3 for highway phases)
  // opts: { exitLane: bool, exitSide: 'right', showRamp: bool }

  // Road surface: dark trapezoid from bottom to vanishing point
  // Lane dividers: dashed white lines converging to vanishing point
  // Each divider: series of short <line> segments with gaps, perspective-scaled
  //   - segment length and gap shrink as y decreases toward vanishing point
  // Road edges: solid white lines (leftmost and rightmost)
}
```

**Perspective math for dashed lines:**
```
For each divider at fraction f across road width (f = 0, 0.33, 0.67, 1.0):
  bottomX = roadLeft + f * roadWidth   (at y = H())
  topX = vanishX                        (at y = vanishY)
  Draw dashed line from (bottomX, H()) to (topX, vanishY)
  Dash length at y: map(y, vanishY, H(), 4, 18)
  Gap length at y: same mapping
```

### Car triangle function

```javascript
function drawCar(cx, cy, opts) {
  // cx, cy: center position
  // opts.lane: which lane (0=left, 1=middle, 2=right) — determines cx
  // Draw: white-filled upward-pointing triangle, width ~28px
  // Add subtle teal drop-shadow via filter for glow effect
  // Triangle points: [cx, cy-16], [cx-12, cy+10], [cx+12, cy+10]
}
```

### Route path function

```javascript
function drawRoutePath(points, opts) {
  // points: array of [x,y] coordinates
  // opts.color: default '#00C8A0'
  // opts.width: default 6
  // Draws a thick, slightly rounded polyline
  // For curves: use quadratic bezier <path d="M x0 y0 Q cx cy x1 y1">
}
```

### Exit ramp function (phases 1, 2)

```javascript
function drawExitRamp(side, divergeY) {
  // side: 'right'
  // divergeY: y-coordinate where ramp peels off the main road
  // Draw a curved path from the rightmost lane edge at divergeY
  // curving to the right edge of the canvas by y = H()*0.7
  // Use quadratic bezier
  // Dashed white border lines on the ramp
}
```

### Schematic intersection function (phase 3 only)

```javascript
function drawSchematicIntersection(opts) {
  // Top-down-ish simplified view (not full perspective)
  // opts.turnDirection: 'right' | 'left'
  // Draw:
  //   - Vertical approach road (2 lanes, dashed center)
  //   - Horizontal crossing road
  //   - Route path: thick teal line going straight then curving right/left
  //   - Car triangle at bottom of vertical road
  //   - Small turn arrow indicator at intersection center
  // Keep geometry clean and abstract — no buildings, no labels on roads
}
```

---

## Phase Definitions

Implement as a `PHASES` array of objects, each consumed by `renderPhase(n)`:

```javascript
const PHASES = [
  {
    id: 0,
    label: '1 mi ahead',
    instruction: 'Stay in lane',
    distance: '1 mi',
    turnIconType: 'straight',   // no action yet
    bannerVisible: false,
    etaTime: '3:11',
    scene: 'perspective-highway',
    carLane: 1,                 // middle lane (0=left, 1=mid, 2=right)
    showExit: false,
    routePath: 'straight',      // thick teal line straight up center lane
  },
  {
    id: 1,
    label: '500 ft',
    instruction: 'Move to right lane',
    distance: '500 ft',
    turnIconType: 'lane-right',
    bannerVisible: true,
    bannerText: '● Move to right lane',
    bannerColor: 'amber',
    etaTime: '3:11',
    scene: 'perspective-highway',
    carLane: 1,                 // still middle, about to shift
    showExit: true,             // exit ramp visible but not yet taken
    routePath: 'curving-right', // teal path curves toward right lane
  },
  {
    id: 2,
    label: 'At exit',
    instruction: 'Exit 1A — take ramp',
    distance: 'Exit 1A',
    turnIconType: 'exit-right',
    bannerVisible: false,
    etaTime: '2:54',
    scene: 'perspective-highway',
    carLane: 2,                 // now in right lane
    showExit: true,
    showExitLabel: true,        // "Exit 1A" sign box top-right
    routePath: 'onto-ramp',     // teal path peels right onto ramp
  },
  {
    id: 3,
    label: 'Complex turn',
    instruction: 'Turn right at light',
    distance: '200 ft',
    turnIconType: 'turn-right',
    bannerVisible: false,
    etaTime: '2:41',
    scene: 'schematic-intersection',
    turnDirection: 'right',
    routePath: 'turn',
  },
  {
    id: 4,
    label: 'After turn',
    instruction: 'Continue straight',
    distance: '0.8 mi',
    turnIconType: 'straight',
    bannerVisible: false,
    etaTime: '2:38',
    scene: 'perspective-highway',
    carLane: 1,
    showExit: false,
    routePath: 'straight',
  },
];
```

---

## Turn Icon SVG Shapes

Draw these inline as SVG paths inside `#turn-icon` (40×40 viewBox):

| `turnIconType` | Description |
|---|---|
| `straight` | Upward arrow, centered |
| `lane-right` | Arrow going up then curving right to a vertical line (lane change) |
| `exit-right` | Arrow going up then branching diagonally right |
| `turn-right` | Arrow curving 90° to the right |

All icons: stroke `#00C8A0`, stroke-width 2.5, fill none, stroke-linecap round.

---

## Animation Spec

### Phase transition animation
When `renderPhase(n)` is called, apply to `#scene-svg`:
```css
@keyframes fadeIn {
  from { opacity: 0; transform: translateY(6px); }
  to   { opacity: 1; transform: translateY(0); }
}
```
Duration: 300ms ease-out. Apply class `phase-enter` then remove after animation ends.

### Lane change banner (phase 1)
Animate in from top:
```css
@keyframes slideDown {
  from { transform: translateY(-100%); opacity: 0; }
  to   { transform: translateY(0); opacity: 1; }
}
```

### Teal route path draw-on effect
For each phase, animate the route path stroke:
```javascript
// After drawing path, set:
path.style.strokeDasharray = pathLength;
path.style.strokeDashoffset = pathLength;
path.style.transition = 'stroke-dashoffset 0.5s ease-out';
// Then in next frame:
path.style.strokeDashoffset = '0';
```

---

## Interaction Spec

### Slider
```javascript
document.getElementById('phase-slider').addEventListener('input', e => {
  renderPhase(parseInt(e.target.value));
  updateSliderFill(e.target.value);
});
```

### Slider fill (CSS trick for teal fill left of thumb)
```javascript
function updateSliderFill(val) {
  const pct = (val / 4) * 100;
  slider.style.background = `linear-gradient(to right, #00C8A0 ${pct}%, rgba(255,255,255,0.15) ${pct}%)`;
}
```

### Phase label clicks
```javascript
document.querySelectorAll('#phase-labels span').forEach(el => {
  el.addEventListener('click', () => {
    const p = parseInt(el.dataset.phase);
    slider.value = p;
    renderPhase(p);
    updateSliderFill(p);
  });
});
```

### Active label highlight
Inside `renderPhase(n)`, update label styles:
```javascript
document.querySelectorAll('#phase-labels span').forEach((el, i) => {
  el.classList.toggle('active', i === n);
});
```

---

## Responsive Behavior

- The layout must work at mobile widths (375px) through desktop (1200px)
- `#map-canvas` fills available height — SVG drawing functions use `W()` and `H()` to recompute coordinates dynamically
- On window resize, call `renderPhase(currentPhase)` to redraw
- Bottom panel fixed height, does not shrink
- Header fixed height, does not shrink

---

## Exit Sign Component (phase 2)

Positioned absolutely in top-right of `#map-canvas`:
```html
<div id="exit-sign" style="display:none">
  <span class="exit-label">Exit</span>
  <span class="exit-number">1A</span>
</div>
```
CSS: green background (`#1A6B3A`), white border, white text, rounded corners, `font-family: DM Mono`, font-size 18px, padding 6px 14px. Only visible in phase 2 (controlled by JS show/hide).

---

## Sequence of Implementation Steps for Claude Code

1. Create `index.html` with full HTML skeleton (head, meta viewport, font CDN links, body structure as specified above)
2. Write all CSS in a `<style>` block — layout, colors, typography, slider, labels, header, bottom panel
3. Write SVG helper functions (`svgEl`, `add`, `clearScene`, `W`, `H`)
4. Write `drawPerspectiveRoad()` with perspective math
5. Write `drawCar()`, `drawRoutePath()`, `drawExitRamp()`, `drawSchematicIntersection()`
6. Write `drawTurnIcon(type)` for all 4 icon types
7. Define `PHASES` array
8. Write `renderPhase(n)` — master function that reads phase data and calls draw functions
9. Write `updateSliderFill(val)` and `updateHeader(phase)`
10. Wire up all event listeners (slider input, label clicks, window resize)
11. Call `renderPhase(0)` and `updateSliderFill(0)` on load
12. Test all 5 phases render without errors
13. Verify responsive behavior at 375px width

---

## GitHub Pages Deployment

Repo setup instructions to include as a comment at top of `index.html`:
```
<!-- 
  Deploy: push index.html to a GitHub repo.
  Go to Settings → Pages → Source: main branch / root.
  Your prototype will be live at https://[username].github.io/[repo-name]/
-->
```

---

## Definition of Done

- [ ] Single `index.html` file, no external assets
- [ ] All 5 phases render distinct, recognizable scenes
- [ ] Slider and label clicks both advance phases
- [ ] Phase transition animation plays on every change
- [ ] Route path draws on with animation each phase
- [ ] Lane change banner appears only on phase 1
- [ ] Exit sign appears only on phase 2
- [ ] Header instruction, distance, and turn icon update each phase
- [ ] Works at 375px mobile width
- [ ] Works at 1200px desktop width
- [ ] No console errors
