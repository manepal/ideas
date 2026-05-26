# Noodle — Build Plan

This is an ordered, file-by-file implementation plan. Each step lists:
- **Files to create/edit**
- **Exact dependencies to install**
- **Verification gate** — test before moving on

Work through these in strict order. Do not skip steps. Do not combine steps. Verify each step before continuing.

---

## Phase 0: Scaffold

### Step 1: Create Expo project
```bash
npx create-expo-app@latest noodle --template blank-typescript
cd noodle
```

**Files created:** Standard Expo template (App.tsx, package.json, tsconfig.json, app.json, etc.)

**Verify:** `npx expo start` launches. Metro bundler compiles. App shows default screen on simulator.

---

## Phase 1: Rendering Engine

### Step 2: Install rendering and animation dependencies
```bash
npx expo install @shopify/react-native-skia react-native-reanimated react-native-gesture-handler expo-haptics expo-av expo-file-system
```

Add to `babel.config.js`:
```js
module.exports = function(api) {
  api.cache(true);
  return {
    presets: ['babel-preset-expo'],
    plugins: ['react-native-reanimated/plugin'],
  };
};
```

**Verify:** App still launches without errors.

### Step 3: Create the dark canvas
**Create:** `src/Canvas.tsx`

Canvas fills the screen. Renders a `#0D0D0D` background with a Skia `<Rect>`. Apply noise shader overlay using Skia's `<RuntimeShader>` or a simple `<Image>` of a pre-generated grain PNG.

**Verify:** App shows pure dark screen with subtle grain. No crashes.

### Step 4: Create the color palette system
**Create:** `src/palettes.ts`

Export the 6 palettes as an array:
```ts
export const PALETTES = [
  { name: 'Jellyfish', body: '#FF6B9D', glow: '#FF8E72', particles: 'spark' },
  { name: 'Deep Sea',  body: '#00D2C8', glow: '#00F5FF', particles: 'dot' },
  { name: 'Ember',    body: '#FF8C42', glow: '#FFD166', particles: 'ember' },
  { name: 'Matcha',   body: '#7ECB76', glow: '#C7F0B5', particles: 'leaf' },
  { name: 'Voidberry', body: '#9B5DE5', glow: '#F72585', particles: 'star' },
  { name: 'Bone',     body: '#F5E6CC', glow: '#FFF3E0', particles: 'pearl' },
];

let paletteIndex = 0;
export function cyclePalette() { paletteIndex = (paletteIndex + 1) % PALETTES.length; }
export function getPalette() { return PALETTES[paletteIndex]; }
```

**Verify:** Import works. `getPalette()` returns first palette. `cyclePalette()` advances index.

### Step 5: Render Noodle as a metaball
**Create:** `src/NoodleBlob.tsx`

Render a single circle using Skia's `<Circle>` with a radial gradient for the glow, centered on screen. Apply a blur filter for the soft edge effect (no hard outline). Size: ~150px radius.

**Verify:** A glowing, soft-edged circle renders on the dark canvas.

### Step 6: Add breathing animation
**Modify:** `src/NoodleBlob.tsx`

Use `useSharedValue` + `withTiming` (reanimated) to pulse the blob scale between 1.0 and 1.08 over 4 seconds, repeating. Tied to a `useFrameCallback` or `useDerivedValue`.

**Verify:** Blob gently expands and contracts. Smooth spring feel, no jitter.

### Step 7: Add internal particles
**Create:** `src/Particles.tsx`

Skia particle system inside the blob. Type varies by palette — white specks, blue dots, orange embers, leaf bits, stars, pearl dust. Particles constrained to within blob radius. ~30 particles, each with random position, speed, and lifetime. Regenerate on loop.

**Verify:** Particles visible inside blob. They drift, fade, respawn. Different per palette.

### Step 8: Add internal caustics
**Modify:** `src/NoodleBlob.tsx`

Overlay a semi-transparent caustics shader (or pre-rendered animated pattern) inside the blob. Subtle shifting highlights that suggest thick liquid. Use Skia shader or a second `<Circle>` with gradient offset animation.

**Verify:** Internal highlights shift slowly. Looks like liquid/gelatin.

---

## Phase 2: Interactions

### Step 9: Add gesture detector overlay
**Modify:** `src/Canvas.tsx`

Wrap the Skia canvas in a `GestureDetector` from react-native-gesture-handler. Create four gestures:

```
Tap gesture → onEnd → dispatch 'tap' at (x, y)
Pan gesture → onUpdate → dispatch 'drag' | onEnd → dispatch 'release'
Fling gesture → onEnd → dispatch 'flick' with velocity
Double-tap gesture → onEnd → cyclePalette()
```

Use refs to track whether the touch point was inside Noodle's bounds vs. empty canvas.

- Touch inside blob: gesture affects Noodle
- Touch outside blob: pan gesture pans the canvas view offset
- Pinch gesture: zoom canvas scale

**Verify:** Log touch coordinates. Tap, drag, flick, double-tap all detected. Canvas pans on empty space drag.

### Step 10: Wire tap → ripple effect
**Create:** `src/RippleEffect.tsx`

On tap, render expanding concentric circles at touch point. Start at radius 0, expand to ~80px, fade opacity from 0.6 to 0 over 1.5s. Use reanimated shared values. Store active ripples in an array, remove when faded. Max 3 concurrent ripples.

**Modify:** `src/NoodleBlob.tsx` — trigger ripple from gesture event.

**Verify:** Tap blob → ripple circles expand and fade. Multiple taps = multiple ripples. Clean removal.

### Step 11: Wire drag → stretch physics
**Modify:** `src/NoodleBlob.tsx`

Pan gesture while touching blob: deform the circle into an ellipse stretched toward the finger. Displacement capped at 80px from center. On release: snap back with spring physics (`withSpring`, damping ~0.5). Use reanimated shared values for `stretchX`, `stretchY`, `offsetX`, `offsetY`.

**Verify:** Drag blob → stretches toward finger like taffy. Release → snaps back with wobble. Feels physically correct.

### Step 12: Wire flick → launch + edge bounce
**Modify:** `src/NoodleBlob.tsx`

On fling gesture, calculate velocity vector. Animate blob position with spring physics in that direction. When blob edge hits screen bounds, reverse velocity with damping (~0.6). Blob decelerates over 3-4 seconds and settles at a resting position. If idle for 10s, slowly drift back toward screen center.

**Verify:** Flick blob → launches, bounces off edges like a water balloon, settles. Multiple flicks chain correctly.

---

## Phase 3: Feedback (Sound + Haptics)

### Step 13: Bundle sound assets
**Create:** `assets/sounds/` directory

Add 10 WAV files (source from freesound.org CC0, or generate simple synth sounds):
- `pop.wav` (<0.3s)
- `blorp_low.wav`, `blorp_mid.wav`, `blorp_high.wav` (<0.4s each)
- `boing.wav` (<0.3s)
- `thump.wav` (<0.2s)
- `bloop.wav` (<0.3s)
- `whoosh.wav` (2s, loopable)
- `squeak_1.wav`, `squeak_2.wav` (<0.2s each)
- `meep.wav` (<0.3s)

**Verify:** All files play correctly via expo-av.

### Step 14: Create sound engine
**Create:** `src/SoundEngine.tsx`

Wrap expo-av `Audio.Sound` objects. Expose functions:
```ts
playPop(), playBlorp(stretchAmount: number), playBoing(), playThump(),
playBloop(), startBreathing(), stopBreathing(), playSqueak(), playMeep()
```

Pitch-shift `blorp` based on stretch amount. Volume scales with gesture speed (use `VelocityTracker` or manual calc from gesture deltas). Respect silent mode (check `Audio.getAudioModeAsync` or use `Audio.INTERRUPTION_MODE_IOS_DUCK_OTHERS`).

**Verify:** Each function plays correct sound. Fast flick = loud boing. Slow drag = quiet blorp. Silent mode respected.

### Step 15: Wire haptics
**Create:** `src/Haptics.ts`

Thin wrapper around expo-haptics:
```ts
import * as Haptics from 'expo-haptics';

export const tapHaptic   = () => Haptics.impactAsync(Haptics.ImpactFeedbackStyle.Light);
export const flickHaptic = () => Haptics.impactAsync(Haptics.ImpactFeedbackStyle.Medium);
export const bounceHaptic = () => Haptics.impactAsync(Haptics.ImpactFeedbackStyle.Soft);
export const rippleHaptic = async () => {
  await Haptics.impactAsync(Haptics.ImpactFeedbackStyle.Light);
  await new Promise(r => setTimeout(r, 150));
  await Haptics.impactAsync(Haptics.ImpactFeedbackStyle.Light);
};
```

Wire into gesture handlers: tap → tapHaptic, drag start → tapHaptic, drag ongoing → throttle to every 300ms, flick → flickHaptic, edge bounce → bounceHaptic, settle → tapHaptic.

**Verify:** Every interaction has matching haptic. Drag feels like a continuous envelope (approximated with throttled pulses).

---

## Phase 4: State & Personality

### Step 16: Create energy system
**Create:** `src/EnergySystem.ts`

Track `noodleEnergy` (0.0–1.0):
```ts
let energy = 0.5;
let lastInteractionTime = Date.now();
let consecutiveDays = 0;
let totalPlayMinutes = 0;

export function registerInteraction(durationMs: number) { /* +0.05 per min */ }
export function onAppOpen() { /* check streak, apply +0.15 if away >24h */ }
export function onDayTick() { /* -0.05 per missed day */ }
export function getEnergy() { return energy; }
export function getEnergyState() { /* returns 'peak' | 'normal' | 'lethargic' | 'sluggish' */ }
```

**Modify:** `src/NoodleBlob.tsx` — read `getEnergyState()` to modulate animation speed, glow intensity, and particle count.

**Verify:** Play for 2 minutes → energy rises. Set system clock forward 24h, reopen → energy drops. Values stay clamped 0.0–1.0.

### Step 17: Persist state to shared JSON
**Create:** `src/Persistence.ts`

On app background (use `AppState` listener), write state JSON to `expo-file-system` documents directory:
```ts
const state = {
  lastOpen: new Date().toISOString(),
  consecutiveDays,
  activePalette: getPalette().name,
  noodleEnergy: getEnergy(),
  totalPlayMinutes,
  lastVisitor: null, // updated by widget
};
await FileSystem.writeAsStringAsync(statePath, JSON.stringify(state));
```

On app foreground, read state and hydrate energy system.

**Configure App Group:** In `app.json`, add `expo` → `ios` → `entitlements` for App Group `group.com.noodle.shared`.

**Verify:** Open app, interact, background it. Check file exists. Reopen → energy restored correctly.

---

## Phase 5: Home Widget

### Step 18: Build iOS widget extension
**Create:** `ios/NoodleWidget/` (via expo config plugin or bare native)

Medium widget. Renders a simplified Noodle view:
- Dark background (`#0D0D0D`)
- Static blob render at ~50px radius (no real-time physics, just the breathing pulse)
- Read shared JSON for `noodleEnergy`, `activePalette`, `lastOpen`
- Time-of-day state logic: sleeping (10PM-6AM), waking (6-9AM), active (9AM-6PM), playful (6-10PM)
- Noodle appears curled during sleep, upright during day

**Create:** `ios/NoodleWidget/NoodleWidget.swift` — WidgetKit TimelineProvider, reads App Group JSON.

**Verify:** Widget appears in iOS widget gallery. Add to home screen. Displays Noodle in correct time-of-day state.

### Step 19: Build Android widget
**Create:** `android/app/src/main/java/com/noodle/widget/` (via expo config plugin or bare native)

Use Jetpack Glance or standard AppWidget. Same logic as iOS widget: read shared prefs, render simplified Noodle, time-of-day states.

**Verify:** Widget appears in Android widget picker. Add to home screen. Displays correctly.

### Step 20: Add visitors to both widgets
**Modify:** iOS and Android widget code

Random visitor spawning: pick a random time within the next refresh window. When visitor time arrives, show visitor animation. Write `lastVisitor` to shared state so app knows.

4 visitor types with simple draw logic:
- Lil' Blorb: smaller circle, random color, hops in/out
- Moth: small glowing dot, random flutter path
- Sparkle: 3 dots on circular orbit
- Raindrop: check weather API, show splash if raining

**Verify:** Wait for widget refresh. Visitor appears. Tapping widget makes visitor leave + meep sound.

### Step 21: Add seasonal accessories to both widgets and app
**Create:** `src/Accessories.ts`

Date-check logic for all 20 festivals. Weather-check via expo-location coarse + fetch from a free weather API (or skip weather accessories if no network — graceful degradation).

Key date logic patterns:
```ts
function getFestivalAccessory(date: Date): Accessory | null { /* ... */ }
function isLunarFestival(festival: string, year: number): Date { /* pre-computed dates */ }
function getWeatherAccessory(weatherCode: string): Accessory | null { /* ... */ }
```

Draw accessories as small Skia shapes positioned relative to Noodle:
- Tika: small red circle at top
- Diya: small flame shape beside
- Santa hat: triangle + pom-pom
- Sunglasses: two dark circles with bridge
- Scarf: horizontal band with dangling end
- etc.

**Modify:** `src/NoodleBlob.tsx` — render active accessory on top of blob.

**Modify:** Widget code on both platforms — render accessories during matching dates/weather.

**Verify:** Set device date to Oct 30 (Dashain 2026 = ~Oct 30). Noodle shows tika + kite string. Set to Dec 25 → Santa hat. Set to rainy weather → umbrella. Multiple accessories don't overlap messily.

---

## Phase 6: Polish

### Step 22: Polish pass
- [ ] Noodle idle drift-back when off-screen after 10s
- [ ] Palette cycle crossfade (300ms color lerp, not instant switch)
- [ ] App respects reduced motion accessibility setting (disable physics, use simple fade)
- [ ] Widget respects dark/light mode (always dark, but respect high contrast)
- [ ] Memory: particles capped at 30, ripples capped at 3
- [ ] No console warnings or errors in release build
- [ ] App launches cold in <2 seconds on mid-range device

### Step 23: Test on both platforms
- [ ] iOS simulator: all gestures, sounds, haptics, widget
- [ ] Android emulator: all gestures, sounds, haptics, widget
- [ ] Physical device (if available): real haptic feel, real widget interaction

---

## File Manifest (Expected Output)

```
noodle/
├── app.json                          # Expo config + App Group entitlement
├── App.tsx                           # Entry: renders Canvas, wires state lifecycle
├── babel.config.js                   # Reanimated plugin
├── assets/
│   └── sounds/                       # 10 WAV files
│       ├── pop.wav
│       ├── blorp_low.wav
│       ├── blorp_mid.wav
│       ├── blorp_high.wav
│       ├── boing.wav
│       ├── thump.wav
│       ├── bloop.wav
│       ├── whoosh.wav
│       ├── squeak_1.wav
│       ├── squeak_2.wav
│       └── meep.wav
├── src/
│   ├── Canvas.tsx                    # Skia Canvas + gesture detector
│   ├── NoodleBlob.tsx                # Metaball render + stretch/launch/breathing
│   ├── Particles.tsx                 # Internal particle system
│   ├── RippleEffect.tsx              # Tap ripple circles
│   ├── SoundEngine.tsx               # expo-av wrapper
│   ├── Haptics.ts                    # expo-haptics wrapper
│   ├── EnergySystem.ts               # Energy tracking + streak logic
│   ├── Persistence.ts                # JSON read/write for app ↔ widget
│   ├── Accessories.ts                # Festival + weather accessory logic + rendering
│   └── palettes.ts                   # Palette definitions + cycle
├── ios/
│   └── NoodleWidget/
│       └── NoodleWidget.swift        # iOS medium widget
└── android/
    └── app/src/main/java/com/noodle/widget/
        └── NoodleWidget.kt           # Android widget
```

## Build Time Estimate

| Phase | Steps | ~Hours |
|-------|-------|--------|
| Phase 0: Scaffold | 1 | 0.5 |
| Phase 1: Rendering | 2–8 | 3 |
| Phase 2: Interactions | 9–12 | 3 |
| Phase 3: Feedback | 13–15 | 2 |
| Phase 4: State | 16–17 | 2 |
| Phase 5: Widget | 18–21 | 5 |
| Phase 6: Polish | 22–23 | 2 |
| **Total** | **23 steps** | **~17.5 hours** |
