# Bagh Chal — Spec

The classic Nepali tiger-goat strategy board game (~2000 years old), built as a polished mobile experience with AI opponent and local multiplayer. Four tigers hunt twenty goats on a 5×5 grid — tigers capture by leaping, goats win by encircling. Pure strategy, zero luck.

## Stack

- **Framework:** React Native (Expo SDK 54, managed workflow)
- **Target:** Expo Go on both iOS and Android (SDK 56 skipped — not yet available on Android Expo Go)
- **Rendering:** @shopify/react-native-skia (board, lines, pieces, effects)
- **Animation:** react-native-reanimated (piece sliding, captures, placement)
- **Haptics:** expo-haptics (move, capture, win/loss)
- **Audio:** expo-av (ambient, move clicks, capture sounds)
- **No backend. No auth. No network calls.**

## Game Rules

### Board

A 5×5 grid of 25 intersection points. Lines connect points horizontally, vertically, and diagonally within each 2×2 cell. Pieces sit on intersection points and move along lines.

```
T •───•───•───• T
│ ╳ │ ╳ │ ╳ │ ╳ │
•───•───•───•───•
│ ╳ │ ╳ │ ╳ │ ╳ │
•───•───•───•───•
│ ╳ │ ╳ │ ╳ │ ╳ │
•───•───•───•───•
│ ╳ │ ╳ │ ╳ │ ╳ │
T •───•───•───• T
```

Tigers start at the four corners. Goats start off the board.

### Phases

**Phase 1 — Placement (bakhri rakhne):**
- Goat player places one goat on any vacant intersection
- Tiger player moves one tiger (adjacent move or capture jump)
- Turns alternate until all 20 goats are placed
- Goats cannot move during placement — only place new ones

**Phase 2 — Movement (chalne):**
- After all 20 goats are placed, goats can now move
- Goat moves: slide one goat to an adjacent vacant intersection along a line
- Tiger moves: slide to adjacent vacant intersection, or jump over an adjacent goat to capture

### Capturing (Tigers)

A tiger captures by jumping over an adjacent goat onto the vacant intersection directly behind it, in a straight line. The jumped goat is removed. Tigers can only jump along the board lines. Multiple captures in one turn are allowed if the tiger lands and can immediately jump again (like checkers). A tiger cannot jump over another tiger.

### Win Conditions

| Winner | Condition |
|--------|-----------|
| **Goats** | All 4 tigers have no legal moves (surrounded) |
| **Tigers** | 5+ goats captured (goats can no longer surround all tigers) |

### Draw / Stalemate

No draws by repetition — the game continues. A stalemate where no player can force a win after 50 moves with no captures results in a draw.

## Visual Design

### Board Theme — Traditional Brass & Wood

The board should feel like a physical artifact: carved wood, inlaid brass lines, stone pieces.

| Element | Color / Material |
|---------|-----------------|
| Background | Dark wood texture (`#2C1810` base, subtle grain pattern via Skia noise) |
| Board surface | Slightly lighter carved recess (`#3D2417`) |
| Grid lines | Warm brass/gold, etched look (`#C9A96E`), 2px stroke, slight glow |
| Intersection points | Small brass dots at each of the 25 intersections, 6px radius |
| Selection highlight | Golden ring pulse around selected piece |

### Tiger Pieces

Drawn procedurally via Skia paths — no image assets. Each tiger is ~16px radius base.

**Path structure:**
- `tigerPath`: Circle base + two triangular ear bumps at top (left ear: -30° from top, right ear: +30°). Ears are small triangles extending ~5px from the circle edge
- `stripePaths`: 3 short curved lines across the body, dark brown with 0.6 opacity
- Eyes: Two small white dots with black pupils, positioned upper-center

| Property | Value |
|----------|-------|
| Body fill | Radial gradient, orange-amber (`#E8782A` center → `#C5551B` edge) |
| Ears | Same gradient fill, slightly darker at tips |
| Stripes | `#3D1A0A`, 1.5px stroke, 0.6 opacity |
| Eyes | White dot (2px) + black pupil (1px), gives personality |
| Active/selected | Golden glow ring (`#F5D06B`) 3px outside body, body scales to 1.15x, glow pulses slowly |
| Valid-move indicator | Small golden dot at destination intersection, pulses 0.5→1.0 scale |

**Skia draw order:**
1. Glow ring (if selected, blur + radial gradient behind body)
2. Body circle + ears (filled path)
3. Stripe paths (stroked)
4. Eyes (two small circles)
5. Highlight spot (small white radial gradient near top-left for convex stone look)

### Goat Pieces

Drawn procedurally via Skia paths. Each goat is ~11px radius base.

**Path structure:**
- `goatPath`: Circle base + two tiny curved horn arcs at top (sweeping outward, ~4px each)
- `hornPaths`: Two small bezier curves, `#D4C5A0` stroke, 1px
- Single eye dot: One small dark dot (no pair needed — profile/head-on ambiguity adds charm)

| Property | Value |
|----------|-------|
| Body fill | Radial gradient, cream/ivory (`#F5E6CC` center → `#D4C5A0` edge) |
| Horns | `#C4B590`, 1px stroke, small arcs |
| Eye | Single `#3D1A0A` dot, 1.5px |
| Active/selected | Soft warm glow (`#FFE8CC`), body scales to 1.15x |
| Placed | Scale from 0 to 1 with overshoot bounce (~300ms spring) |
| Captured | Shrink + fade + 8-particle burst scattering outward |

### Animations

All piece movement uses spring physics via Reanimated — pieces accelerate then settle, no linear slides.

| Event | Animation | Duration |
|-------|-----------|----------|
| Place goat | Scale from 0 to 1 with overshoot bounce | ~300ms |
| Move piece | Slide along line path, spring settle | ~250ms |
| Capture | Tiger leaps (arc path), goat shrinks + particles, slight screen shake | ~500ms |
| Invalid move attempt | Piece wobbles in place, no movement | ~200ms |
| Phase transition | Brief flash + text banner ("All goats placed — begin movement!") | ~1.5s |
| Win | Winning pieces pulse/enlarge, losing pieces dim | ~2s |

### Screen Layout

```
┌─────────────────────┐
│   ⚙️  │  Turn / HUD  │  ← Status bar (dark, minimal)
│       │  Tigers: 4   │
│       │  Goats: 20   │
├─────────────────────┤
│                     │
│    Board Canvas     │  ← Board centered, fills available space
│    (Skia)           │
│                     │
├─────────────────────┤
│  Undo  │  Phase     │  ← Bottom bar (transparent overlay)
│        │  Indicator │
└─────────────────────┘
```

## Main Menu

Before each game, the player sees:

- **Side Selection:** "Play as" with two large tappable icons — Tiger (aggressive, hunter) or Goat (defensive, strategic). Brief one-line description under each.
- **Mode Selection:** "vs AI" or "Pass & Play" (two players on same device)
- **Difficulty (vs AI only):** Easy (depth 2), Medium (depth 4), Hard (depth 6 + opening book). Shown as three tappable chips, Easy selected by default.
- **Start Game** button

No settings. No profile. No tutorial screens. Tutorial is inline on first play (see below).

## Game Screen

### Board Rendering (Skia Canvas)

- 5×5 grid centered in available screen space
- Cell size calculated dynamically: `min(screenWidth, screenHeight * 0.6) / 5`
- Board origin offset to center horizontally and vertically in the canvas
- Lines drawn first (brass color, 2px, slight glow via blur), then intersection dots, then pieces, then effects overlay
- Valid move indicators: small pulsing dots at reachable intersections when a piece is selected

### HUD

Top bar, semi-transparent dark overlay:

```
[Goats: 18/20]              [Phase: Placement]              [⚙️]
```

- Left: remaining goats to place / goats on board
- Center: current phase name + whose turn (highlight color matches active side)
- Right: settings/menu icon (opens pause overlay with concede, new game, back to menu)
- During AI thinking: center shows "Thinking..." with a subtle pulsing dot

### Interaction Flow

1. **Your turn:** Tap a piece → it highlights, valid moves glow → tap destination → piece animates there
2. **Tap empty space or same piece:** Deselects
3. **Tap invalid destination:** Piece wobbles, haptic rejection pulse
4. **AI turn:** HUD says "Thinking...", board is non-interactive, after response AI piece animates
5. **Pass & Play:** Same flow, just no AI — HUD shows whose turn, board may flip 180° between turns so each player sees "their side" (optional, toggled in pause menu)

### Phase Indicator

Prominent but not intrusive. During placement: a small row of 20 dots at top of board, fill in as goats are placed. When full, a brief "Movement Phase" banner appears and fades.

## AI Engine

### Minimax with Alpha-Beta Pruning

| Difficulty | Depth | Extras |
|------------|-------|--------|
| Easy | 2 ply | Random move tie-breaking |
| Medium | 4 ply | Alpha-beta pruning, positional eval |
| Hard | 6 ply | Alpha-beta + opening book for first 6 moves + capture-first heuristic |

### Evaluation Function (Scoring Heuristic)

The AI evaluates board positions based on:

- **Mobility:** Number of legal moves for tigers vs goats (weighted by phase)
- **Captures:** Goats captured so far (tigers maximize, goats minimize)
- **Tiger position:** Tigers near center control more area — bonus for central positions
- **Goat encirclement:** Goats forming tight clusters around tigers — negative score for tigers
- **Tiger safety:** Penalty for tigers with zero legal moves (immediate goat win)

The evaluation is asymmetric — tiger AI maximizes captures and mobility, goat AI maximizes encirclement and restriction.

### Opening Book (Hard Only)

Pre-computed strong opening positions for first 6 placements/moves. Stored as a lookup table keyed by board state hash. Covers common strong openings for both sides. ~50 entries.

### Performance

AI must respond in under 2 seconds on a mid-range phone. Progressive deepening: compute depth 2 first, show that move if time runs out, otherwise continue to deeper search.

## Haptics

| Event | Style | Intensity |
|-------|-------|-----------|
| Pick up piece (select) | `Light` impact | Low |
| Place/move piece | `Medium` impact | Medium |
| Tiger capture | `Heavy` impact + 100ms delay + `Light` | High |
| Invalid move | `Rigid` notification | Low |
| Phase transition | Double `Medium` (150ms apart) | Medium |
| Game win | Triple ascending `Light` → `Medium` → `Heavy` | High |
| Game loss | Single long vibration pattern (if available) | Medium |

## Sound Design

All sounds are short, warm, organic — not synthetic. Stone on wood, not digital bleeps.

| Event | Sound | Character |
|-------|-------|-----------|
| Piece placed (goat) | Soft stone tap on wood | Short, light |
| Piece moved | Wooden slide, brief | Warm friction |
| Tiger capture | Sharp clack + thud | Dramatic, satisfying |
| Invalid move | Muffled dull thud | Gentle rejection |
| Phase transition | Low gong/temple bell hit | Ceremonial |
| Victory | Rising chime sequence, 3 notes | Celebratory |
| Defeat | Descending two-note | Somber, brief |
| Menu select | Soft click | Subtle |

## First-Play Experience

On very first app launch, a single overlay card appears:

> "Tigers hunt. Goats surround.  
> Tap a piece to select it, then tap where you want it to go.  
> Tigers capture by jumping over goats. Goats win by trapping all tigers."

Single "Got it" button dismisses it. Never shown again. That's the entire tutorial.

## Data Layer

### Persisted State (expo-file-system)

```json
{
  "gamesPlayed": 42,
  "winsAsGoats": 12,
  "winsAsTigers": 8,
  "losses": 22,
  "difficulty": "medium",
  "tutorialSeen": true
}
```

- Auto-save on game end
- No mid-game save (games are ~10 minutes, against design philosophy)

### No Online Features

No leaderboards. No accounts. No cloud sync. No matchmaking. Stats are local-only.

## Accessibility

- **Reduced motion:** Disable piece slide animations, use instant position change + fade
- **Board flippable:** In pass-and-play, board rotates 180° so each player sees their perspective
- **Tap targets:** Minimum 44×44pt hit area for intersections and menu buttons
- **Color independence:** Tigers and goats are distinguishable by shape AND color (not color alone)
- **Haptic-only mode:** All feedback works without sound

## Game Modes Summary

| Mode | Description |
|------|-------------|
| vs AI — Play as Goats | Player places/moves goats, AI controls tigers. Strategic, defensive. |
| vs AI — Play as Tigers | Player moves tigers, captures goats. AI places/moves goats. Aggressive. |
| Pass & Play | Two humans, one device. Board flips between turns. No AI involved. |

## Component Architecture

```
App.tsx                         // Navigation state (menu vs game)
├── MainMenu.tsx                // Side select, mode, difficulty, start
├── GameScreen.tsx              // Orchestrates everything below
│   ├── GameEngine.ts           // Pure game logic: rules, validation, state machine
│   ├── AIEngine.ts             // Minimax + alpha-beta + opening book
│   ├── BoardCanvas.tsx         // Skia canvas: lines, points, pieces, effects
│   ├── HUD.tsx                 // Turn indicator, goat count, phase, pause
│   ├── GameOverModal.tsx       // Win/loss overlay, play again, back to menu
│   ├── TutorialOverlay.tsx     // First-play one-card tutorial
│   └── PauseMenu.tsx           // Concede, flip board, back to menu
├── SoundEngine.tsx             // expo-av wrapper: preload, play, volume
├── Haptics.ts                  // expo-haptics wrapper
├── Storage.ts                  // Read/write stats JSON
└── types.ts                    // BoardState, Piece, Move, GamePhase, etc.
```

## Build Stages

Built in 6 sequential stages, each fully testable before the next begins:

| Stage | What You Can Do | ~Hours |
|-------|----------------|--------|
| **1. Board & Setup** | See the board with pieces in starting position. No interaction yet. | 2 |
| **2. Manual Play** | Tap-to-select, tap-to-move. Full game loop works. Two humans can play a complete game. | 4 |
| **3. AI Opponent** | vs AI mode works at all 3 difficulties. AI thinks, responds, animates moves. | 4 |
| **4. Feedback** | Haptics, sound, animations — every interaction feels polished. | 3 |
| **5. Screens & Flow** | Main menu, game over modal, pause, tutorial, stats. Complete app flow. | 3 |
| **6. Polish** | Accessibility, reduced motion, board flip, performance, edge cases. | 2 |
| **Total** | | **~18 hours** |

## Verification

- [ ] Board renders correctly — 25 points, all lines, 4 tigers at corners
- [ ] Tap tiger → highlights → tap valid adjacent point → tiger slides there
- [ ] Tap tiger → tap point behind adjacent goat → tiger jumps, goat removed
- [ ] Phase 1: goats only place, don't move. Tigers move + capture normally.
- [ ] Phase 2 starts automatically when all 20 goats are placed
- [ ] Phase 2: goats can move to adjacent empty points
- [ ] Goat win detected: all tigers blocked
- [ ] Tiger win detected: 5 goats captured
- [ ] Invalid move: piece wobbles, no state change
- [ ] AI responds within 2 seconds on all difficulties
- [ ] Easy AI makes occasional blunders, Hard AI plays strong openings
- [ ] Pass & play: turn alternates, no AI, board flip works
- [ ] Sound and haptics respect device silent mode
- [ ] Stats persist across app restarts
- [ ] Tutorial shows on first launch, never again
