# Noodle — Spec

A living, physics-based fidget companion that exists both as an app and a home screen widget. Touch Noodle and it reacts. Leave it alone and it lives its own tiny life. No scores, no goals, no words.

## Stack

- **Framework:** React Native (Expo, managed workflow)
- **Rendering:** @shopify/react-native-skia (metaball blob, caustics, particles, glow)
- **Animation:** react-native-reanimated (spring physics, gestures)
- **Haptics:** expo-haptics (smooth rubbery curves, ripples, pops)
- **Audio:** expo-av (procedural foley playback)
- **Widget:** iOS WidgetKit via expo config plugin + Android Glance via expo config plugin (or bare native modules if needed)
- **Shared storage:** expo-file-system + App Group container (iOS) / shared preferences (Android) for widget ↔ app communication
- **No backend. No auth. No network calls.**

## Noodle — The Character

Noodle is a single gelatinous metaball creature with a translucent body, internal caustics, and a radial glow. It lives on a matte dark void. It has moods but never dies, never begs, never judges.

## Core Interactions

| Touch | Action | Response |
|-------|--------|----------|
| **Tap** | Quick finger down + up on blob | Circular ripple from touch point, decays over ~1.5s. Soft *pop* sound + ripple haptic. |
| **Slow drag** | Finger down + move + release | Noodle stretches toward finger like warm taffy. Finger up → snap back with wobble. Pitch-rising *blorp* sound + smooth elastic haptic. |
| **Flick** | Fast swipe across blob | Noodle launches in swipe direction, bounces off screen edges (spring physics), settles over 3-4s. *Boing* sound + snappy haptic. Edge bounce = muffled *thump*. |
| **Double-tap** | Two quick taps anywhere | Cycles active color palette. No sound, subtle flash. The only gesture that changes state. |

## Canvas Interaction

- **Pan:** Single finger drag on empty canvas space pans the view.
- **Zoom:** Pinch-to-zoom to get closer or pull back from Noodle.
- Canvas has infinite feel — Noodle can be moved around freely.
- No camera reset button. If Noodle is off-screen, it slowly drifts back toward center when idle.

## Visual Design

### Canvas
- Matte near-black: `#0D0D0D`
- Subtle noise/grain texture overlay (Skia shader)
- No grids, no borders, no chrome

### Noodle Body
- Single metaball rendered via Skia
- Semi-translucent with internal caustics (shifting highlights suggesting thick liquid interior)
- No outline — edges fade into void
- Radial bloom glow, color matches active palette
- Glow intensity pulses with breathing (~1 breath cycle every 4s)

### Particles
- Internal particles float within the blob (type depends on palette)
- Subtle secondary animation layer

### Color Palettes

| Name | Body | Glow | Particles |
|------|------|------|-----------|
| Jellyfish | `#FF6B9D` pink | `#FF8E72` coral | White specks, slow drift |
| Deep Sea | `#00D2C8` teal | `#00F5FF` cyan | Bioluminescent blue dots |
| Ember | `#FF8C42` orange | `#FFD166` gold | Floating orange embers |
| Matcha | `#7ECB76` green | `#C7F0B5` sage | Tiny leaf-like particles |
| Voidberry | `#9B5DE5` purple | `#F72585` magenta | Star-like sparkle dots |
| Bone | `#F5E6CC` ivory | `#FFF3E0` cream | Delicate pearl shimmer |

Default palette on first launch: Jellyfish.

### Animation Principles
- Spring physics for all motion (never linear easing)
- No hard resets, no snapping — always settles gently
- Flick deceleration: natural physics, like a water balloon sliding on glass

## Sound Design

All sounds are short synthesized clips (no recordings). Volume scales with interaction speed. Respects device silent/vibrate mode.

| Event | Sound | Character |
|-------|-------|-----------|
| Tap ripple | Soft *pop*, soap bubble burst | Quick, bright, satisfying |
| Drag stretch | Wet elastic *blorp*, pitch rises with stretch | Rubbery, playful |
| Flick launch | Short *boing*, fast decay | Cartoonish, not grating |
| Edge bounce | Muffled *thump*, water balloon on glass | Soft impact |
| Settle | Tiny descending *bloop*, exhale | Relief, rest |
| Idle breathing | Near-silent whoosh, seashell to ear | Calm, hypnotic |
| Sleep snore | Micro-squeaks, irregular | Endearing |

Sound assets are generated procedurally at build time or sourced from royalty-free SFX libraries.

## Haptics

Each interaction has a distinct tactile identity:

- **Tap:** Quick single pulse, light intensity
- **Ripple:** Decaying concentric pulses over ~1s
- **Drag:** Long smooth envelope, like pulling warm honey, intensity increases with stretch distance
- **Flick:** Snappy single pulse, higher intensity
- **Edge bounce:** Muffled single pulse, medium intensity
- **Settle:** Two descending-intensity pulses

Use expo-haptics `notificationFeedbackAsync` style patterns — no custom haptic engine needed for the MVP, but the pattern timing should match each interaction.

## Energy & Personality System

`noodleEnergy` (0.0–1.0 float) drives all behavior modulation:

| Range | Behavior | Visual | Sound |
|-------|----------|--------|-------|
| 0.9–1.0 | Energetic, visitors frequent, fast ripples | Full glow, fast particles, high saturation | Louder, higher-pitched |
| 0.5–0.8 | Normal reactions, calm breathing | Default intensity | Normal |
| 0.2–0.4 | Slower ripples, heavier drag, long settles | Dimmed, desaturated ~30% | Quieter, lower-pitched |
| 0.0–0.1 | Barely responsive, slow breathing, no visitors | Near-transparent, minimal glow | Only faint breathing |

**Energy gain:**
- +0.05 per minute of active touch interaction (cap at 1.0)
- +0.15 on first app open if previous open was >24h ago (Noodle missed you)

**Energy decay:**
- -0.05 per calendar day with zero app opens (cap at 0.0, Noodle is never "dead")

**Consecutive days:** Tracked silently. 7+ consecutive days → rare visitors appear. 30+ consecutive days → Noodle does a special recognition wiggle on app open. No number shown, no badge, no notification. Just a quiet reward.

## Home Widget

### Sizing
- iOS: Medium widget (systemMedium)
- Android: 2×2 cells

### Visual
- Cropped view into Noodle's void (same `#0D0D0D` background)
- Noodle occupies ~40% of widget frame
- System-rounded corners (match platform)
- Renders a static preview + updates at system widget refresh cadence

### Time-Based States (no real-time connection to app)

State is estimated from `lastOpen` timestamp + current time of day:

| Time | State | Behavior |
|------|-------|----------|
| 6–9 AM | Waking | Slow stretch, lazy wobbles, dim glow |
| 9 AM–6 PM | Active | Bouncy, brighter, responsive to widget taps |
| 6–10 PM | Playful | Energetic, visitors more likely |
| 10 PM–6 AM | Sleeping | Curled ball, dim glow, tiny squeak bubble animations |

### Visitors (Random, ~3–4 times/day)

| Visitor | Behavior | Duration |
|---------|----------|----------|
| Lil' Blorb | Tiny Noodle variant, different color. Hops in, Noodle boops it, it leaves. | ~8s |
| Moth | Glowing speck, flutters near Noodle. Noodle stretches toward it. Moth escapes. | ~5s |
| Sparkle | Three floating dots orbit Noodle twice, then fade. Noodle looks proud. | ~6s |
| Raindrop | Only during rain (system weather API). Drip splashes near Noodle, creates ripple. | ~4s |

### Widget Tap
- Noodle does a startled wiggle, then settles
- If visitor present, it scurries off
- Plays *meep* sound (tiny, muffled, distant)
- Does NOT launch app (system handles app launch separately)

### Seasonal Accessories (Automatic, Time + Weather-Based)

Accessories apply in both the app and the widget. They layer — if two conditions overlap, both accessories appear. Fully automatic, no settings, no notifications.

#### Festival Accessories (Date-Based)

Check calendar on app launch + widget refresh. Yearly recurring unless a specific lunar/regional calendar is noted.

| Festival | Date/Logic | Accessory | Region |
|----------|-----------|-----------|--------|
| New Year | Jan 1 | Tiny party hat + confetti particle burst on open | Global |
| Maghe Sankranti | Jan 14 | Noodle sits in a tiny woven basket, sesame seed particles | Nepal, India |
| Lunar New Year | Lunar calendar (Jan 21 – Feb 20) | Red envelope tucked beside Noodle + gold glitter particles | East Asia, global diaspora |
| Valentine's Day | Feb 14 | Small floating heart that drifts near Noodle | Global |
| Maha Shivaratri | Lunar (Feb/Mar) | Tiny damaru drum beside Noodle, body shifts to cooler blue tones | Nepal, India |
| Holi | Full moon, Phalgun (Feb/Mar) | Multi-color splotches across body, scattered powder particles | Nepal, India |
| Nowruz | Mar 21 | Sprouting wheat grass beside Noodle, fresh green body tint | Iran, Central Asia, global |
| Easter | First Sunday after Paschal full moon (Mar/Apr) | Tiny decorated egg nestled next to Noodle | Global |
| Eid al-Fitr | Lunar (varies yearly) | Small crescent moon + star floating above Noodle, warm gold glow | Global Islamic |
| Songkran | Apr 13–15 | Tiny water droplets on Noodle's body, playful splash particles | Thailand, SE Asia |
| Buddha Jayanti | Lunar (Apr/May) | Small glowing lotus beneath Noodle, soft white halo | Nepal, India, Buddhist regions |
| Raksha Bandhan | Full moon, Shravan (Aug) | Tiny colorful rakhi thread tied around a small point on Noodle | Nepal, India |
| Teej | Lunar (Aug/Sep) | Small red bangles on one side, mehendi-like swirl pattern on body | Nepal, North India |
| Janai Purnima | Full moon, Shravan (Aug) | Tiny sacred thread across Noodle | Nepal |
| Dashain | Lunar (Sep/Oct) | Red tika on forehead, small kite string trailing | Nepal |
| Tihar / Diwali | Lunar (Oct/Nov) | Small diya flickering behind Noodle, warm orange particles, tika (Tihar day 1) | Nepal, India, global |
| Halloween | Oct 31 | Tiny sheet-ghost costume (white draped over Noodle with eye holes) | Global |
| Chhath | Lunar, 6 days after Diwali (Oct/Nov) | Noodle near a miniature water pond, soft golden sunrise particles | Nepal, Bihar, UP |
| Christmas | Dec 25 | Tiny Santa hat, present box beside Noodle, snowflake particles | Global |
| New Year's Eve | Dec 31 | Noodle wears a sparkly bow tie, fireworks particles at midnight | Global |

#### Weather Accessories (Live, via System Weather API)

Checked on app open + widget refresh. Requires location permission (coarse/general city-level is sufficient).

| Weather Condition | Accessory |
|-------------------|-----------|
| Sunny / Clear | Tiny sunglasses on Noodle, warm golden haze in background |
| Cloudy / Overcast | Small grey umbrella tilted beside Noodle, muted background tones |
| Rain / Drizzle | Small raindrop hat, tiny puddle beneath Noodle, ripple effects |
| Heavy Rain / Thunderstorm | Noodle shivers subtly, tiny lightning flash particles, deeper puddle |
| Snow | Noodle wears a tiny scarf, snowflake particles drift down, pale blue background tint |
| Fog / Haze | Noodle's edges soften extra, faint mist particles, muted glow |
| Windy | Noodle's internal particles blow sideways, slight body lean/tilt |
| Hot (>35°C / 95°F) | Tiny sweat droplets on Noodle, body slightly more translucent (melty vibe) |
| Cold (<0°C / 32°F) | Frost particles on Noodle's surface, body slightly more solid/frozen, ice crystal edges |

If location permission is denied, weather accessories are simply skipped — Noodle stays in its base state.

#### Accessory Priority & Layering

- Festival and weather accessories draw simultaneously when both apply
- Max 2 accessories visible at once to avoid visual clutter
- Priority: Festival > Weather (festival accessory takes precedence if conflict)
- When 3+ conditions overlap, pick the most specific (festival wins, then rarest weather condition)
- Accessories appear at launch, fade in over ~2s, persist until app close or widget refresh
- Party hat from Jan 1 + Santa hat from Dec 25 can overlap if applicable (different visual elements)

No user-visible settings for accessories. Fully automatic based on device date + optional weather.

## Data Layer

### Shared State (App ↔ Widget)

Single JSON file in App Group / shared preferences container:

```json
{
  "lastOpen": "2026-05-26T18:45:00Z",
  "consecutiveDays": 4,
  "activePalette": "jellyfish",
  "noodleEnergy": 0.85,
  "totalPlayMinutes": 23,
  "lastVisitor": {
    "type": "moth",
    "seenAt": "2026-05-26T14:12:00Z"
  }
}
```

- **App writes** on close/background: all fields
- **Widget reads** on refresh: all fields
- **Widget writes** on visitor appearance: `lastVisitor`
- **No network, no sync, no auth**

### Storage APIs
- expo-file-system for JSON read/write
- expo-shared-group or native config for App Group access on iOS
- SharedPreferences or DataStore on Android

## App Screen Architecture

Single screen, no navigation:

```
App.tsx
├── <GestureHandler>           // Tap, drag, flick, double-tap, pan, pinch
│   └── <Canvas>               // Skia canvas, fills screen
│       ├── BackgroundLayer    // #0D0D0D + grain shader
│       ├── NoodleLayer        // Metaball blob + caustics + particles + glow
│       └── RippleLayer        // Transient ripple effects from taps
├── <SoundEngine>              // Invisible, plays procedural clips on events
└── <StateManager>             // Tracks energy, consecutive days, palette
```

No tabs. No settings screen. No onboarding screens. First launch: Noodle is just there, breathing, at 0.5 energy, Jellyfish palette.

## Sound Asset Generation

Generate short synthesized clips at build time. Alternative: bundle 10–15 small WAV files (~2–5 KB each) sourced from royalty-free SFX (e.g., freesound.org, filtered for CC0). Each clip trimmed to <1 second.

Clips needed:
- pop.wav (tap)
- blorp_low.wav, blorp_mid.wav, blorp_high.wav (drag, pitch-shifted at runtime)
- boing.wav (flick)
- thump.wav (edge bounce)
- bloop.wav (settle)
- whoosh.wav (breathing, looped)
- squeak_1.wav, squeak_2.wav (snore)
- meep.wav (widget tap)

## Build Order (Agent Instructions)

1. **Scaffold:** `npx create-expo-app@latest noodle` with TypeScript template
2. **Install deps:** react-native-reanimated, @shopify/react-native-skia, expo-haptics, expo-av, expo-file-system
3. **Build canvas:** Skia background + grain shader
4. **Render Noodle:** Skia metaball + glow + caustics shader + breathing animation
5. **Add gestures:** react-native-gesture-handler for tap/drag/flick/double-tap
6. **Add physics:** Spring animations via reanimated, edge bounce detection
7. **Add ripples:** Transient ripple circles on tap, decaying in radius and opacity
8. **Add particles:** Skia particle system inside the metaball, palette-dependent
9. **Add haptics:** Wire gesture handlers → expo-haptics patterns
10. **Add sound:** Bundle audio assets, wire gesture handlers → expo-av playback
11. **Add energy system:** Interaction tracking, decay timer, behavior modulation
12. **Add state persistence:** Read/write shared JSON on background
13. **Add palette cycling:** Double-tap handler, palette switch with crossfade
14. **Build iOS widget:** WidgetKit extension via expo config plugin, renders Noodle in static + time-based states
15. **Build Android widget:** Glance widget via expo config plugin
16. **Wire widget data:** Both widgets read from shared container
17. **Add visitors:** Time-based random visitor spawning logic in widget
18. **Add seasonal accessories:** Date-check logic in widget
19. **Test:** Run on iOS simulator + Android emulator, verify all interactions, widget refresh

## Verification

- [ ] Open app → Noodle appears, breathing, Jellyfish palette
- [ ] Tap Noodle → ripple + pop sound + haptic
- [ ] Drag Noodle → stretches, snap-back, blorp sound + elastic haptic
- [ ] Flick Noodle → launches, bounces off edges, settles, boing + thump sounds
- [ ] Double-tap anywhere → palette cycles, blob crossfades to new colors
- [ ] Pan on empty canvas → view pans
- [ ] Pinch → zoom in/out
- [ ] Close app, wait 5 min → energy has updated in shared JSON
- [ ] Add widget to home screen → Noodle visible, breathing
- [ ] Widget shows correct time-of-day state (sleeping at 2 AM, active at noon)
- [ ] Tap widget → Noodle wiggles, meep sound
- [ ] Device on silent → no sounds play (respects silent mode)
- [ ] Leave app unopened 3+ days → reopen, Noodle sluggish, slow reactions
- [ ] Open app 7 consecutive days → rare visitors start appearing in widget
- [ ] Seasonal accessory appears on correct date (check Dashain/Holi dates for current year)
