# Bagh Chal — Build Plan

Ordered, file-by-file implementation plan. Each step lists files to create/edit, dependencies to install, and a verification gate. Work through these in strict order. Do not skip steps. Do not combine steps. Verify each step before continuing.

---

## Phase 0: Scaffold

### Step 1: Create Expo project (SDK 54)

```bash
npx create-expo-app@latest bagh-chal --template blank-typescript
cd bagh-chal
```

Pin SDK 54 in `app.json` — ensure `expo.sdkVersion` is `54.0.0` and `expo.version` matches.

**Verify:** `npx expo start` launches. Metro compiles. App shows default screen on iOS simulator.

### Step 2: Install core dependencies

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

**Verify:** App still launches without errors. Reanimated plugin loads.

---

## Phase 1: Board & Setup (Stage 1 — Static Board)

### Step 3: Define all types

**Create:** `src/types.ts`

```ts
export type Player = 'tiger' | 'goat';
export type GamePhase = 'placement' | 'movement';
export type Difficulty = 'easy' | 'medium' | 'hard';
export type GameMode = 'ai' | 'pass-and-play';
export type Side = 'tiger' | 'goat';

export interface Point {
  row: number; // 0-4
  col: number; // 0-4
}

export interface Piece {
  type: 'tiger' | 'goat';
  id: string;
}

export type Board = (Piece | null)[][];

export interface GameState {
  board: Board;
  phase: GamePhase;
  currentTurn: Player;
  goatsPlaced: number;
  goatsCaptured: number;
  selectedPoint: Point | null;
  gameOver: boolean;
  winner: Player | null;
  mode: GameMode;
  side: Side;
  difficulty: Difficulty;
  moveCount: number;
  movesSinceLastCapture: number;
}

export interface Move {
  from: Point | null; // null = placement (goat placed from off-board)
  to: Point;
}

export interface GameStats {
  gamesPlayed: number;
  winsAsGoats: number;
  winsAsTigers: number;
  losses: number;
  preferredDifficulty: Difficulty;
  tutorialSeen: boolean;
}
```

**Verify:** TypeScript compiles. No type errors on import.

### Step 4: Define board geometry constants

**Create:** `src/boardGeometry.ts`

```ts
export const GRID_SIZE = 5;
export const BOARD_POINTS = 25;

// Adjacency list: for each point, list of adjacent points connected by board lines.
// Points indexed row-major: index = row * 5 + col.
// Connections: horizontal, vertical, both diagonals within each 2×2 cell.
export function buildAdjacencyList(): number[][] {
  const adj: number[][] = Array.from({ length: 25 }, () => []);

  for (let r = 0; r < 5; r++) {
    for (let c = 0; c < 5; c++) {
      const idx = r * 5 + c;
      // Horizontal
      if (c < 4) adj[idx].push(r * 5 + (c + 1));
      if (c > 0) adj[idx].push(r * 5 + (c - 1));
      // Vertical
      if (r < 4) adj[idx].push((r + 1) * 5 + c);
      if (r > 0) adj[idx].push((r - 1) * 5 + c);
      // Diagonal ↘ (top-left to bottom-right)
      if (r < 4 && c < 4) adj[idx].push((r + 1) * 5 + (c + 1));
      if (r > 0 && c > 0) adj[idx].push((r - 1) * 5 + (c - 1));
      // Diagonal ↙ (top-right to bottom-left)
      if (r < 4 && c > 0) adj[idx].push((r + 1) * 5 + (c - 1));
      if (r > 0 && c < 4) adj[idx].push((r - 1) * 5 + (c + 1));
    }
  }
  return adj;
}

export const ADJACENCY = buildAdjacencyList();

export function pointToIndex(p: { row: number; col: number }): number {
  return p.row * 5 + p.col;
}

export function indexToPoint(idx: number): { row: number; col: number } {
  return { row: Math.floor(idx / 5), col: idx % 5 };
}

export function isAdjacent(a: number, b: number): boolean {
  return ADJACENCY[a].includes(b);
}

export function areCollinear(a: number, b: number, c: number): boolean {
  const pa = indexToPoint(a);
  const pb = indexToPoint(b);
  const pc = indexToPoint(c);
  // Same row
  if (pa.row === pb.row && pb.row === pc.row) return true;
  // Same column
  if (pa.col === pb.col && pb.col === pc.col) return true;
  // Same diagonal (row diff = col diff)
  const dr1 = pb.row - pa.row;
  const dc1 = pb.col - pa.col;
  const dr2 = pc.row - pb.row;
  const dc2 = pc.col - pb.col;
  return dr1 === dc1 && dr2 === dc2 || dr1 === -dc1 && dr2 === -dc2;
}

// Line definitions for rendering: pairs of point indices
export function buildLines(): [number, number][] {
  const lines: [number, number][] = [];
  for (let i = 0; i < 25; i++) {
    for (const j of ADJACENCY[i]) {
      if (i < j) lines.push([i, j]);
    }
  }
  return lines;
}

export const BOARD_LINES = buildLines();
```

**Verify:** Import in a test console.log — `BOARD_LINES.length` is correct, `ADJACENCY[0]` (corner) has fewer connections than `ADJACENCY[12]` (center).

### Step 5: Render the static board

**Create:** `src/BoardCanvas.tsx`

Skia canvas that fills the screen. Renders:
1. Dark wood background (`#2C1810` base + noise shader for grain)
2. Board surface area (`#3D2417`) as a slightly lighter rounded rect inset
3. `BOARD_LINES` drawn as brass-colored strokes (`#C9A96E`, 2px, with blur glow)
4. 25 intersection dots (brass, 6px radius)
5. 4 tiger pieces at corners, styled as circles with ears

Calculate cell size: `min(screenWidth, screenHeight * 0.6) / 5`. Board origin centered.

The component accepts no props yet — just renders the static starting position.

```tsx
import { Canvas, Circle, Line, Rect, Group, Paint, BlurMask, RadialGradient } from '@shopify/react-native-skia';
import { View, useWindowDimensions } from 'react-native';
import { BOARD_LINES, indexToPoint } from './boardGeometry';

const BG_COLOR = '#2C1810';
const BOARD_SURFACE = '#3D2417';
const BRASS = '#C9A96E';
const TIGER_ORANGE = '#E8782A';
const TIGER_DARK = '#C5551B';

export function BoardCanvas() {
  const { width, height } = useWindowDimensions();
  const cellSize = Math.min(width, height * 0.6) / 5;
  const boardPixelSize = cellSize * 4; // 4 gaps between 5 points
  const originX = (width - boardPixelSize) / 2;
  const originY = (height * 0.15); // leave room for HUD

  const toPixel = (row: number, col: number) => ({
    x: originX + col * cellSize,
    y: originY + row * cellSize,
  });

  return (
    <View style={{ flex: 1, backgroundColor: BG_COLOR }}>
      <Canvas style={{ flex: 1 }}>
        {/* Board surface */}
        <Rect
          x={originX - 20} y={originY - 20}
          width={boardPixelSize + 40} height={boardPixelSize + 40}
          color={BOARD_SURFACE}
          rx={12} ry={12}
        />

        {/* Grid lines */}
        {BOARD_LINES.map(([a, b], i) => {
          const pa = toPixel(indexToPoint(a).row, indexToPoint(a).col);
          const pb = toPixel(indexToPoint(b).row, indexToPoint(b).col);
          return (
            <Line key={i} p1={pa} p2={pb} color={BRASS} strokeWidth={2} />
          );
        })}

        {/* Intersection dots */}
        {Array.from({ length: 25 }, (_, i) => {
          const { row, col } = indexToPoint(i);
          const { x, y } = toPixel(row, col);
          return <Circle key={i} cx={x} cy={y} r={6} color={BRASS} />;
        })}

        {/* Tigers at corners */}
        {[[0,0], [0,4], [4,0], [4,4]].map(([row, col], i) => {
          const { x, y } = toPixel(row, col);
          return (
            <Group key={i}>
              {/* Glow */}
              <Circle cx={x} cy={y} r={18} color={TIGER_ORANGE} opacity={0.3}>
                <Paint>
                  <BlurMask blur={8} />
                </Paint>
              </Circle>
              {/* Body */}
              <Circle cx={x} cy={y} r={16}>
                <Paint>
                  <RadialGradient
                    c={{ x, y }}
                    r={16}
                    colors={[TIGER_ORANGE, TIGER_DARK]}
                  />
                </Paint>
              </Circle>
            </Group>
          );
        })}
      </Canvas>
    </View>
  );
}
```

**Verify:** Dark wood board appears with brass lines, 25 dots, 4 glowing orange tiger pieces at corners. Board is centered.

### Step 6: Draw tiger with ears, stripes, and eyes (procedural Skia paths)

**Modify:** `src/BoardCanvas.tsx` — extract tiger rendering into a helper, replace the simple `<Circle>` tigers with the full path-based version.

Add inside `BoardCanvas.tsx` (or better, a separate helper):

```ts
// Tiger piece render helper — returns Skia elements for one tiger
function renderTiger(cx: number, cy: number, radius: number, selected: boolean) {
  const stripeColor = '#3D1A0A';

  // Ear tip positions
  const leftEarTip = { x: cx - radius * 0.55, y: cy - radius * 0.7 };
  const rightEarTip = { x: cx + radius * 0.55, y: cy - radius * 0.7 };

  return (
    <Group>
      {selected && (
        <Circle cx={cx} cy={cy} r={radius + 4} color="#F5D06B" opacity={0.6}>
          <Paint><BlurMask blur={6} /></Paint>
        </Circle>
      )}
      {/* Body */}
      <Circle cx={cx} cy={cy} r={radius}>
        <Paint>
          <RadialGradient c={{ x: cx, y: cy }} r={radius}
            colors={['#E8782A', '#C5551B']} />
        </Paint>
      </Circle>
      {/* Ears — two small triangles pointing upward */}
      <Path
        path={`M ${cx - radius * 0.4} ${cy - radius * 0.3} L ${leftEarTip.x} ${leftEarTip.y} L ${cx - radius * 0.1} ${cy - radius * 0.3} Z`}
        color="#C5551B"
      />
      <Path
        path={`M ${cx + radius * 0.1} ${cy - radius * 0.3} L ${rightEarTip.x} ${rightEarTip.y} L ${cx + radius * 0.4} ${cy - radius * 0.3} Z`}
        color="#C5551B"
      />
      {/* Stripes — 3 short lines across body */}
      <Line p1={{ x: cx - 8, y: cy - 4 }} p2={{ x: cx + 8, y: cy - 4 }}
        color={stripeColor} strokeWidth={1.5} opacity={0.6} />
      <Line p1={{ x: cx - 9, y: cy }} p2={{ x: cx + 9, y: cy }}
        color={stripeColor} strokeWidth={1.5} opacity={0.6} />
      <Line p1={{ x: cx - 7, y: cy + 4 }} p2={{ x: cx + 7, y: cy + 4 }}
        color={stripeColor} strokeWidth={1.5} opacity={0.6} />
      {/* Eyes */}
      <Circle cx={cx - 4} cy={cy - 5} r={2.5} color="white" />
      <Circle cx={cx - 4} cy={cy - 5} r={1.2} color="#1A0A00" />
      <Circle cx={cx + 4} cy={cy - 5} r={2.5} color="white" />
      <Circle cx={cx + 4} cy={cy - 5} r={1.2} color="#1A0A00" />
    </Group>
  );
}
```

Replace the 4 corner tiger placeholders with calls to `renderTiger(x, y, 16, false)`.

**Verify:** Tigers now have ears, stripes, and eyes. Distinct from the goat design (not yet rendered).

---

## Phase 2: Manual Play (Stage 2 — Playable Game)

### Step 7: Create the game engine — state creation and move validation

**Create:** `src/gameEngine.ts`

Pure functions with no React dependencies. This is the authoritative rules engine.

```ts
import { Board, GameState, GameMode, Side, Difficulty, Player, Move, Point, Piece } from './types';
import { ADJACENCY, pointToIndex, isAdjacent, areCollinear, indexToPoint } from './boardGeometry';

let goatIdCounter = 0;
let tigerIdCounter = 0;

export function createInitialState(
  mode: GameMode,
  side: Side,
  difficulty: Difficulty
): GameState {
  goatIdCounter = 0;
  tigerIdCounter = 0;

  const board: Board = Array.from({ length: 5 }, () => Array(5).fill(null));

  // Place 4 tigers at corners
  const corners: [number, number][] = [[0, 0], [0, 4], [4, 0], [4, 4]];
  for (const [r, c] of corners) {
    board[r][c] = { type: 'tiger', id: `t${tigerIdCounter++}` };
  }

  return {
    board,
    phase: 'placement',
    currentTurn: 'goat', // goats always go first
    goatsPlaced: 0,
    goatsCaptured: 0,
    selectedPoint: null,
    gameOver: false,
    winner: null,
    mode,
    side,
    difficulty,
    moveCount: 0,
    movesSinceLastCapture: 0,
  };
}

export function getValidMoves(state: GameState, point: Point): Move[] {
  const { board, phase, currentTurn } = state;
  const piece = board[point.row][point.col];
  if (!piece || piece.type !== currentTurn) return [];

  const fromIdx = pointToIndex(point);
  const moves: Move[] = [];

  if (piece.type === 'tiger') {
    for (const adjIdx of ADJACENCY[fromIdx]) {
      const adj = indexToPoint(adjIdx);
      const target = board[adj.row][adj.col];

      if (target === null) {
        // Move to empty adjacent spot
        moves.push({ from: point, to: adj });
      } else if (target.type === 'goat') {
        // Try to capture: jump over goat
        const dr = adj.row - point.row;
        const dc = adj.col - point.col;
        const beyond = { row: adj.row + dr, col: adj.col + dc };

        if (
          beyond.row >= 0 && beyond.row < 5 &&
          beyond.col >= 0 && beyond.col < 5 &&
          board[beyond.row][beyond.col] === null &&
          isAdjacent(adjIdx, pointToIndex(beyond)) &&
          areCollinear(fromIdx, adjIdx, pointToIndex(beyond))
        ) {
          moves.push({ from: point, to: beyond });
        }
      }
    }
  } else if (piece.type === 'goat') {
    if (phase === 'movement') {
      for (const adjIdx of ADJACENCY[fromIdx]) {
        const adj = indexToPoint(adjIdx);
        if (board[adj.row][adj.col] === null) {
          moves.push({ from: point, to: adj });
        }
      }
    }
    // During placement, goats don't move — they're placed from off-board
  }

  return moves;
}

export function getPlacementMoves(state: GameState): Point[] {
  if (state.phase !== 'placement' || state.currentTurn !== 'goat') return [];
  const points: Point[] = [];
  for (let r = 0; r < 5; r++) {
    for (let c = 0; c < 5; c++) {
      if (state.board[r][c] === null) {
        points.push({ row: r, col: c });
      }
    }
  }
  return points;
}

export function makeMove(state: GameState, move: Move): GameState {
  const newState = cloneState(state);
  const { board, phase } = newState;
  const piece = move.from ? board[move.from.row][move.from.col] : null;

  if (move.from === null) {
    // Placing a goat
    board[move.to.row][move.to.col] = { type: 'goat', id: `g${goatIdCounter++}` };
    newState.goatsPlaced++;
    if (newState.goatsPlaced >= 20) {
      newState.phase = 'movement';
    }
  } else if (piece && piece.type === 'tiger') {
    // Check if this is a capture move
    const fromIdx = pointToIndex(move.from);
    const toIdx = pointToIndex(move.to);
    const dr = move.to.row - move.from.row;
    const dc = move.to.col - move.from.col;
    const dist = Math.abs(dr) + Math.abs(dc);

    if (dist > 2) {
      // Capture: jump over goat
      const jumpedRow = move.from.row + dr / 2;
      const jumpedCol = move.from.col + dc / 2;
      board[jumpedRow][jumpedCol] = null;
      newState.goatsCaptured++;
      newState.movesSinceLastCapture = 0;
    } else {
      newState.movesSinceLastCapture++;
    }

    // Move tiger
    board[move.from.row][move.from.col] = null;
    board[move.to.row][move.to.col] = piece;
  } else if (piece && piece.type === 'goat') {
    // Move goat
    board[move.from.row][move.from.col] = null;
    board[move.to.row][move.to.col] = piece;
    newState.movesSinceLastCapture++;
  }

  // Switch turn
  newState.currentTurn = newState.currentTurn === 'tiger' ? 'goat' : 'tiger';
  newState.moveCount++;
  newState.selectedPoint = null;

  // Check win conditions
  newState.winner = checkWinCondition(newState);
  if (newState.winner) {
    newState.gameOver = true;
  }

  // Check draw (50 moves without capture)
  if (newState.movesSinceLastCapture >= 50) {
    newState.gameOver = true;
    newState.winner = null; // draw
  }

  return newState;
}

function checkWinCondition(state: GameState): Player | null {
  // Tigers win: 5+ goats captured
  if (state.goatsCaptured >= 5) return 'tiger';

  // Goats win: all tigers have no legal moves
  const tigersHaveMoves = findTigersWithMoves(state);
  if (!tigersHaveMoves) return 'goat';

  return null;
}

function findTigersWithMoves(state: GameState): boolean {
  for (let r = 0; r < 5; r++) {
    for (let c = 0; c < 5; c++) {
      const piece = state.board[r][c];
      if (piece && piece.type === 'tiger') {
        const moves = getValidMoves(state, { row: r, col: c });
        if (moves.length > 0) return true;
      }
    }
  }
  return false;
}

function cloneState(state: GameState): GameState {
  return {
    ...state,
    board: state.board.map(row => row.map(cell => cell ? { ...cell } : null)),
  };
}

export function getAIMove(state: GameState): Move | null {
  // Placeholder — returns null for now, AI implemented in Phase 3
  return null;
}
```

**Verify:** Import and test:
```ts
const state = createInitialState('ai', 'goat', 'medium');
console.log(state.board[0][0]?.type); // 'tiger'
console.log(state.currentTurn); // 'goat'
const placements = getPlacementMoves(state);
console.log(placements.length); // 21 (25 - 4 tigers)
```

### Step 8: Wire tap-to-select and valid move display

**Modify:** `src/BoardCanvas.tsx`

Add React state via props. Accept `gameState`, `onPlaceGoat`, `onSelectPiece`, `onMovePiece` callbacks.

Add tap gesture detection on the canvas. On tap:
- Convert pixel coordinates to nearest board intersection (snap within `cellSize * 0.4`)
- If no piece selected and tap hits a piece of current player → call `onSelectPiece(point)`
- If piece already selected and tap hits a valid move destination → call `onMovePiece(move)`
- If piece already selected and tap hits empty/invalid → deselect
- During goat placement phase and goat turn → tap empty point → call `onPlaceGoat(point)`

Render valid move indicators: small pulsing gold dots (`#F5D06B`, 5px radius, opacity oscillates 0.5→1.0) at reachable intersections when a piece is selected.

Render selection ring around selected piece.

**Verify:** Tap a tiger → golden ring appears, valid moves highlighted. Tap valid destination → console.log fires. Tap elsewhere → deselects.

### Step 9: Create GameScreen — wire state to board

**Create:** `src/GameScreen.tsx`

The orchestrator component. Holds game state in a `useState`. Wires `BoardCanvas` callbacks to `gameEngine` functions.

```tsx
import { useState, useCallback } from 'react';
import { View } from 'react-native';
import { GameState, GameMode, Side, Difficulty, Point, Move } from './types';
import { createInitialState, getPlacementMoves, getValidMoves, makeMove } from './gameEngine';
import { BoardCanvas } from './BoardCanvas';

export function GameScreen({ mode, side, difficulty }: {
  mode: GameMode;
  side: Side;
  difficulty: Difficulty;
}) {
  const [state, setState] = useState<GameState>(
    () => createInitialState(mode, side, difficulty)
  );

  const handleSelectPiece = useCallback((point: Point) => {
    setState(prev => ({ ...prev, selectedPoint: point }));
  }, []);

  const handlePlaceGoat = useCallback((point: Point) => {
    setState(prev => makeMove(prev, { from: null, to: point }));
  }, []);

  const handleMovePiece = useCallback((to: Point) => {
    setState(prev => {
      if (!prev.selectedPoint) return prev;
      return makeMove(prev, { from: prev.selectedPoint, to });
    });
  }, []);

  return (
    <View style={{ flex: 1 }}>
      <BoardCanvas
        gameState={state}
        onSelectPiece={handleSelectPiece}
        onPlaceGoat={handlePlaceGoat}
        onMovePiece={handleMovePiece}
      />
    </View>
  );
}
```

**Verify:** Launch the app → board shows. Tap empty point during goat turn → goat appears. Tigers alternate turns (tap tiger, tap adjacent empty → tiger moves). Placement phase works. Can play a full manual game.

### Step 10: Add goat piece rendering

**Modify:** `src/BoardCanvas.tsx` — add `renderGoat` helper analogous to `renderTiger`.

```ts
function renderGoat(cx: number, cy: number, radius: number, selected: boolean) {
  return (
    <Group>
      {selected && (
        <Circle cx={cx} cy={cy} r={radius + 3} color="#FFE8CC" opacity={0.5}>
          <Paint><BlurMask blur={5} /></Paint>
        </Circle>
      )}
      {/* Body */}
      <Circle cx={cx} cy={cy} r={radius}>
        <Paint>
          <RadialGradient c={{ x: cx, y: cy }} r={radius}
            colors={['#F5E6CC', '#D4C5A0']} />
        </Paint>
      </Circle>
      {/* Horns — two small arcs */}
      <Circle cx={cx - 3} cy={cy - 8} r={4} color="#C4B590" style="stroke" strokeWidth={1} />
      <Circle cx={cx + 3} cy={cy - 8} r={4} color="#C4B590" style="stroke" strokeWidth={1} />
      {/* Eye */}
      <Circle cx={cx} cy={cy - 2} r={1.5} color="#3D1A0A" />
      {/* Highlight */}
      <Circle cx={cx - 3} cy={cy - 4} r={3} color="white" opacity={0.2} />
    </Group>
  );
}
```

The board render loop now calls `renderGoat` for each goat on the board.

**Verify:** During placement, goats appear as cream-colored circles with tiny horns and a single eye. Distinct from tigers.

### Step 11: Add HUD

**Create:** `src/HUD.tsx`

Top bar showing game info:

```tsx
import { View, Text, StyleSheet, TouchableOpacity } from 'react-native';
import { GameState } from './types';

export function HUD({ state, onPause }: { state: GameState; onPause: () => void }) {
  const turnLabel = state.currentTurn === 'tiger' ? 'Tiger' : 'Goat';
  const phaseLabel = state.phase === 'placement' ? 'Placement' : 'Movement';
  const goatsRemaining = 20 - state.goatsPlaced;
  const goatsOnBoard = state.goatsPlaced - state.goatsCaptured;

  return (
    <View style={styles.container}>
      <View style={styles.info}>
        <Text style={styles.label}>
          {state.phase === 'placement'
            ? `Goats to place: ${goatsRemaining}`
            : `Goats: ${goatsOnBoard}`}
        </Text>
        <Text style={styles.phase}>{phaseLabel} · {turnLabel}'s Turn</Text>
      </View>
      <TouchableOpacity onPress={onPause} style={styles.pauseBtn}>
        <Text style={styles.pauseText}>⚙️</Text>
      </TouchableOpacity>
    </View>
  );
}

const styles = StyleSheet.create({
  container: {
    flexDirection: 'row', justifyContent: 'space-between', alignItems: 'center',
    paddingHorizontal: 20, paddingTop: 60, paddingBottom: 12,
    backgroundColor: 'rgba(13, 8, 5, 0.9)',
  },
  info: { flex: 1 },
  label: { color: '#C9A96E', fontSize: 14, fontWeight: '600' },
  phase: { color: '#8A7560', fontSize: 12, marginTop: 2 },
  pauseBtn: { padding: 8 },
  pauseText: { fontSize: 20 },
});
```

**Verify:** HUD shows at top. Displays correct phase, turn, and goat count. Updates as state changes.

### Step 12: Wire in-game win concurrency — placement dot indicator

**Modify:** `src/HUD.tsx` — add a row of 20 small dots that fill as goats are placed.

```tsx
// Inside HUD, below the info row:
<View style={styles.dotRow}>
  {Array.from({ length: 20 }, (_, i) => (
    <View key={i} style={[
      styles.dot,
      { backgroundColor: i < state.goatsPlaced ? '#C9A96E' : '#3D2A1A' }
    ]} />
  ))}
</View>
```

**Verify:** Dots fill left to right as goats are placed. All 20 filled when movement phase begins.

### Step 13: Add game over detection and display

**Modify:** `src/GameScreen.tsx` — when `state.gameOver` is true, show a result banner.

For now, an inline overlay:
```tsx
{state.gameOver && (
  <View style={styles.gameOverOverlay}>
    <Text style={styles.gameOverText}>
      {state.winner
        ? `${state.winner === 'tiger' ? '🐅 Tigers' : '🐐 Goats'} Win!`
        : 'Draw'}
    </Text>
    <TouchableOpacity onPress={handleNewGame}>
      <Text style={styles.playAgain}>Play Again</Text>
    </TouchableOpacity>
  </View>
)}
```

**Verify:** Play through a game. When 5 goats captured → "Tigers Win!" shows. When all tigers trapped → "Goats Win!" shows. "Play Again" resets state.

---

## Phase 3: AI Opponent (Stage 3 — vs AI)

### Step 14: Create board evaluation function

**Create:** `src/aiEngine.ts` — start with the evaluation function.

```ts
import { GameState, Player, Move, Point } from './types';
import { getValidMoves, getPlacementMoves } from './gameEngine';
import { pointToIndex, ADJACENCY, indexToPoint } from './boardGeometry';

// Higher = better for tigers. Lower (more negative) = better for goats.
export function evaluateBoard(state: GameState): number {
  if (state.gameOver) {
    if (state.winner === 'tiger') return 1000;
    if (state.winner === 'goat') return -1000;
    return 0; // draw
  }

  let score = 0;

  // Captured goats (tigers want more, goats want fewer)
  score += state.goatsCaptured * 100;

  // Count tiger moves (mobility)
  let tigerMoves = 0;
  let goatMoves = 0;
  let tigerPositions: Point[] = [];

  for (let r = 0; r < 5; r++) {
    for (let c = 0; c < 5; c++) {
      const piece = state.board[r][c];
      if (!piece) continue;
      const moves = getValidMoves(state, { row: r, col: c });
      if (piece.type === 'tiger') {
        tigerMoves += moves.length;
        tigerPositions.push({ row: r, col: c });
      } else {
        goatMoves += moves.length;
      }
    }
  }

  // Mobility: tigers want to keep options open, goats want to restrict tigers
  score += tigerMoves * 10;
  score -= goatMoves * 3;

  // Tiger centralization bonus (center control = more options)
  for (const pos of tigerPositions) {
    const distFromCenter = Math.abs(pos.row - 2) + Math.abs(pos.col - 2);
    score += (4 - distFromCenter) * 5; // max bonus at center
  }

  // Trapped tigers penalty
  for (const pos of tigerPositions) {
    if (getValidMoves(state, pos).length === 0) {
      score -= 200;
    }
  }

  // Phase adjustment
  if (state.phase === 'placement') {
    // During placement, more goats on board = harder for tigers
    score -= state.goatsPlaced * 3;
  }

  return score;
}
```

**Verify:** Write a quick console.log test:
```ts
const s = createInitialState('ai', 'goat', 'medium');
console.log(evaluateBoard(s)); // a positive number (tigers advantaged early)
```

### Step 15: Build minimax with alpha-beta pruning

**Modify:** `src/aiEngine.ts` — add the search algorithm.

```ts
export function findBestMove(
  state: GameState,
  difficulty: 'easy' | 'medium' | 'hard'
): Move | null {
  const depthMap = { easy: 2, medium: 4, hard: 6 };
  const depth = depthMap[difficulty];
  const aiPlayer = state.currentTurn;

  let bestMove: Move | null = null;
  let bestScore = aiPlayer === 'tiger' ? -Infinity : Infinity;
  const alpha = -Infinity;
  const beta = Infinity;

  const moves = getAllMovesForPlayer(state, aiPlayer);

  if (moves.length === 0) return null;

  // Shuffle for variety (especially on easy)
  shuffle(moves);

  for (const move of moves) {
    const newState = makeMove(state, move);
    const score = minimax(newState, depth - 1, alpha, beta, aiPlayer, false);

    if (aiPlayer === 'tiger') {
      if (score > bestScore) {
        bestScore = score;
        bestMove = move;
      }
    } else {
      if (score < bestScore) {
        bestScore = score;
        bestMove = move;
      }
    }
  }

  return bestMove;
}

function minimax(
  state: GameState,
  depth: number,
  alpha: number,
  beta: number,
  aiPlayer: Player,
  maximizing: boolean
): number {
  if (depth === 0 || state.gameOver) {
    const score = evaluateBoard(state);
    return aiPlayer === 'tiger' ? score : -score;
  }

  const currentPlayer = state.currentTurn;
  const moves = getAllMovesForPlayer(state, currentPlayer);

  if (moves.length === 0) {
    return evaluateBoard(state);
  }

  // Move ordering: captures first for better pruning
  sortMoves(moves);

  if (currentPlayer === aiPlayer) {
    // Maximizing for the AI player
    let maxEval = -Infinity;
    for (const move of moves) {
      const newState = makeMove(state, move);
      const evalScore = minimax(newState, depth - 1, alpha, beta, aiPlayer, false);
      maxEval = Math.max(maxEval, evalScore);
      alpha = Math.max(alpha, evalScore);
      if (beta <= alpha) break;
    }
    return maxEval;
  } else {
    // Minimizing for the opponent
    let minEval = Infinity;
    for (const move of moves) {
      const newState = makeMove(state, move);
      const evalScore = minimax(newState, depth - 1, alpha, beta, aiPlayer, true);
      minEval = Math.min(minEval, evalScore);
      beta = Math.min(beta, evalScore);
      if (beta <= alpha) break;
    }
    return minEval;
  }
}

function getAllMovesForPlayer(state: GameState, player: Player): Move[] {
  if (player === 'goat' && state.phase === 'placement') {
    return getPlacementMoves(state).map(to => ({ from: null, to }));
  }

  const moves: Move[] = [];
  for (let r = 0; r < 5; r++) {
    for (let c = 0; c < 5; c++) {
      const piece = state.board[r][c];
      if (piece && piece.type === player) {
        const validMoves = getValidMoves(state, { row: r, col: c });
        for (const move of validMoves) {
          moves.push(move);
        }
      }
    }
  }
  return moves;
}

function sortMoves(moves: Move[]): void {
  // Prioritize capture moves (distance > 2 = capture for tigers)
  moves.sort((a, b) => {
    const aCapture = a.from && Math.abs(a.to.row - a.from.row) + Math.abs(a.to.col - a.from.col) > 2;
    const bCapture = b.from && Math.abs(b.to.row - b.from.row) + Math.abs(b.to.col - b.from.col) > 2;
    if (aCapture && !bCapture) return -1;
    if (!aCapture && bCapture) return 1;
    return 0;
  });
}

function shuffle<T>(arr: T[]): void {
  for (let i = arr.length - 1; i > 0; i--) {
    const j = Math.floor(Math.random() * (i + 1));
    [arr[i], arr[j]] = [arr[j], arr[i]];
  }
}
```

**Verify:** Call `findBestMove(state, 'easy')` — returns a valid Move. Depth 2 returns in <200ms.

### Step 16: Create opening book for Hard difficulty

**Create:** `src/openingBook.ts`

```ts
// Opening book: board state hash → recommended move.
// State hash = stringified board + phase + goatsPlaced.
// Covers first 6 goat placements + corresponding tiger responses.

const OPENING_BOOK: Record<string, { from: { row: number; col: number } | null; to: { row: number; col: number } }> = {
  // Goat openings: best first placements
  'empty_0': { from: null, to: { row: 2, col: 2 } },    // Goat: place center
  'empty_1': { from: null, to: { row: 1, col: 2 } },    // Goat: support center
  'empty_2': { from: null, to: { row: 3, col: 2 } },    // Goat: build wall
  'empty_3': { from: null, to: { row: 2, col: 1 } },    // Goat: flank
  'empty_4': { from: null, to: { row: 2, col: 3 } },    // Goat: encircle
  'empty_5': { from: null, to: { row: 1, col: 1 } },    // Goat: corner trap prep
};

export function hashState(state: { board: any[][]; phase: string; goatsPlaced: number }): string {
  if (state.goatsPlaced <= 5) {
    return `empty_${state.goatsPlaced}`;
  }
  // After 6 moves, fall through to minimax
  return '';
}

export function lookupOpeningMove(hash: string): { from: { row: number; col: number } | null; to: { row: number; col: number } } | null {
  return OPENING_BOOK[hash] || null;
}
```

**Verify:** `lookupOpeningMove('empty_0')` returns center placement. `lookupOpeningMove('empty_99')` returns null.

### Step 17: Integrate AI into game flow

**Modify:** `src/GameScreen.tsx`

Add a `useEffect` that triggers AI move after human turn in vs-AI mode:

```tsx
useEffect(() => {
  if (state.gameOver) return;
  if (state.mode !== 'ai') return;

  const isAITurn = state.currentTurn !== state.side;

  if (isAITurn) {
    const timer = setTimeout(() => {
      const move = findBestMove(state, state.difficulty);
      if (move) {
        setState(prev => makeMove(prev, move));
      }
    }, 400); // Brief delay so player sees their move first
    return () => clearTimeout(timer);
  }
}, [state.currentTurn, state.gameOver]);
```

Show "Thinking..." in HUD during AI turn.

**Verify:** Play as goats vs AI tigers. AI tiger moves after each goat placement/move. AI responds within 2 seconds on all difficulties.

### Step 18: Add Pass & Play mode support

**Modify:** `src/GameScreen.tsx` — Pass & Play requires no AI logic. The same game engine works, just both sides are human.

When `mode === 'pass-and-play'`, skip the AI `useEffect`. Add a brief turn-switch indicator:

```tsx
// After makeMove in pass & play, optionally flip board:
const [boardFlipped, setBoardFlipped] = useState(false);

// In toggle in pause menu:
setBoardFlipped(prev => !prev);
```

Pass `flipped={boardFlipped}` to `BoardCanvas` — when flipped, render pieces rotated 180° (transform group on the Skia canvas with `rotate(Math.PI)` at board center).

**Verify:** Start pass & play. Both players alternate on same device. No AI fires. Board flips when toggled.

---

## Phase 4: Feedback (Stage 4 — Sound + Haptics + Animation)

### Step 19: Bundle sound assets

**Create:** `assets/sounds/` directory. Source 8 short WAV files (CC0 from freesound.org) or generate simple synth tones:

```
assets/sounds/
├── place.wav      // Soft stone tap (<0.2s)
├── move.wav       // Wooden slide (<0.3s)
├── capture.wav    // Sharp clack + thud (<0.4s)
├── invalid.wav    // Muffled thud (<0.2s)
├── transition.wav // Low bell hit (<0.5s)
├── victory.wav    // Rising 3-note chime (<1s)
├── defeat.wav     // Descending 2-note (<0.5s)
└── select.wav     // Soft click (<0.1s)
```

**Verify:** All files play via expo-av `Audio.Sound.createAsync()`.

### Step 20: Create sound engine

**Create:** `src/SoundEngine.tsx`

```tsx
import { Audio } from 'expo-av';
import { useEffect, useRef } from 'react';

const SOUNDS = {
  place: require('../assets/sounds/place.wav'),
  move: require('../assets/sounds/move.wav'),
  capture: require('../assets/sounds/capture.wav'),
  invalid: require('../assets/sounds/invalid.wav'),
  transition: require('../assets/sounds/transition.wav'),
  victory: require('../assets/sounds/victory.wav'),
  defeat: require('../assets/sounds/defeat.wav'),
  select: require('../assets/sounds/select.wav'),
} as const;

type SoundName = keyof typeof SOUNDS;

export function useSoundEngine() {
  const soundsRef = useRef<Record<string, Audio.Sound | null>>({});

  useEffect(() => {
    // Preload all sounds
    for (const [name, source] of Object.entries(SOUNDS)) {
      Audio.Sound.createAsync(source, { shouldPlay: false })
        .then(({ sound }) => { soundsRef.current[name] = sound; });
    }
    return () => {
      // Unload all
      for (const sound of Object.values(soundsRef.current)) {
        sound?.unloadAsync();
      }
    };
  }, []);

  const play = async (name: SoundName) => {
    const sound = soundsRef.current[name];
    if (sound) {
      await sound.setPositionAsync(0);
      await sound.playAsync();
    }
  };

  return { play };
}
```

**Verify:** `play('place')` plays the tap sound. `play('capture')` plays the capture sound. All 8 sounds load and play.

### Step 21: Create haptics wrapper

**Create:** `src/Haptics.ts`

```ts
import * as Haptics from 'expo-haptics';

export const Haptic = {
  select:      () => Haptics.impactAsync(Haptics.ImpactFeedbackStyle.Light),
  move:        () => Haptics.impactAsync(Haptics.ImpactFeedbackStyle.Medium),
  capture:     async () => {
    await Haptics.impactAsync(Haptics.ImpactFeedbackStyle.Heavy);
    await new Promise(r => setTimeout(r, 100));
    await Haptics.impactAsync(Haptics.ImpactFeedbackStyle.Light);
  },
  invalid:     () => Haptics.notificationAsync(Haptics.NotificationFeedbackType.Error),
  transition:  async () => {
    await Haptics.impactAsync(Haptics.ImpactFeedbackStyle.Medium);
    await new Promise(r => setTimeout(r, 150));
    await Haptics.impactAsync(Haptics.ImpactFeedbackStyle.Medium);
  },
  victory:     async () => {
    for (const style of [Haptics.ImpactFeedbackStyle.Light, Haptics.ImpactFeedbackStyle.Medium, Haptics.ImpactFeedbackStyle.Heavy]) {
      await Haptics.impactAsync(style);
      await new Promise(r => setTimeout(r, 120));
    }
  },
  defeat:      () => Haptics.impactAsync(Haptics.ImpactFeedbackStyle.Heavy),
};
```

**Verify:** Call each function — distinct haptic patterns fire. Tones feel differentiated.

### Step 22: Wire sound and haptics into game events

**Modify:** `src/GameScreen.tsx`

Track previous state. On state change, detect what happened and fire appropriate feedback:

```tsx
const prevState = useRef(state);

useEffect(() => {
  const prev = prevState.current;
  if (prev === state) return;

  // Phase transition
  if (prev.phase === 'placement' && state.phase === 'movement') {
    play('transition');
    Haptic.transition();
  }

  // Capture detected (goatsCaptured increased)
  if (state.goatsCaptured > prev.goatsCaptured) {
    play('capture');
    Haptic.capture();
  }
  // Placement
  else if (state.goatsPlaced > prev.goatsPlaced) {
    play('place');
    Haptic.move();
  }
  // Regular move
  else if (!state.gameOver && prev.currentTurn !== state.currentTurn) {
    play('move');
    Haptic.select();
  }

  // Game over
  if (state.gameOver && !prev.gameOver) {
    if (state.winner === state.side) {
      play('victory');
      Haptic.victory();
    } else {
      play('defeat');
      Haptic.defeat();
    }
  }

  prevState.current = state;
}, [state]);
```

**Verify:** Play through turns. Every action has matching sound + haptic. Phase transition plays bell. Win/loss plays distinctive feedback.

### Step 23: Add piece movement animations

**Modify:** `src/BoardCanvas.tsx`

Animate piece movement using Reanimated shared values instead of jumping instantly. When a move occurs:
- Animate position from `fromPoint` to `toPoint` over 250ms with spring easing
- For captures: scale goat to 0 + fade (springOut), tiger arcs with slight overshoot

Add a `moveAnimation` prop to BoardCanvas:
```ts
interface MoveAnimation {
  from: Point | null; // null = placement
  to: Point;
  isCapture: boolean;
}
```

Use `useAnimatedReaction` or a simple state-driven approach: when `moveAnimation` changes, animate.

For an MVP animation approach (without full Reanimated shared values on Skia canvas), use React state + `Animated` values or Skia's built-in `useValue` + `useComputedValue`:

```tsx
import { useValue, runTiming, Easing } from '@shopify/react-native-skia';

// Inside BoardCanvas:
const animProgress = useValue(1);

useEffect(() => {
  if (moveAnimation) {
    animProgress.current = 0;
    runTiming(animProgress, 1, { duration: 250, easing: Easing.out(Easing.exp) });
  }
}, [moveAnimation]);
```

**Verify:** Pieces slide smoothly between positions. Captured goats shrink and fade. Newly placed goats bounce in.

---

## Phase 5: Screens & Flow (Stage 5 — Complete App)

### Step 24: Build Main Menu

**Create:** `src/MainMenu.tsx`

Full menu screen with:
- App title "Bagh Chal" in a traditional serif font
- Subtitle "The Ancient Game of Tigers & Goats"
- Side selection: two large cards — "🐅 Play as Tigers" / "🐐 Play as Goats" (highlight selected)
- Mode toggle: "vs AI" / "Pass & Play"
- Difficulty selector (shown only when vs AI selected): Easy / Medium / Hard chips
- "Start Game" button

```tsx
import { View, Text, TouchableOpacity, StyleSheet } from 'react-native';
import { useState } from 'react';
import { Difficulty, GameMode, Side } from './types';

export function MainMenu({ onStart }: {
  onStart: (mode: GameMode, side: Side, difficulty: Difficulty) => void;
}) {
  const [mode, setMode] = useState<GameMode>('ai');
  const [side, setSide] = useState<Side>('goat');
  const [difficulty, setDifficulty] = useState<Difficulty>('easy');

  return (
    <View style={styles.container}>
      <Text style={styles.title}>Bagh Chal</Text>
      <Text style={styles.subtitle}>The Ancient Game of Tigers & Goats</Text>

      {/* Side selection */}
      <Text style={styles.sectionLabel}>Choose Your Side</Text>
      <View style={styles.sideRow}>
        <TouchableOpacity
          style={[styles.sideCard, side === 'tiger' && styles.selected]}
          onPress={() => setSide('tiger')}
        >
          <Text style={styles.sideEmoji}>🐅</Text>
          <Text style={styles.sideLabel}>Tigers</Text>
          <Text style={styles.sideDesc}>Hunter</Text>
        </TouchableOpacity>
        <TouchableOpacity
          style={[styles.sideCard, side === 'goat' && styles.selected]}
          onPress={() => setSide('goat')}
        >
          <Text style={styles.sideEmoji}>🐐</Text>
          <Text style={styles.sideLabel}>Goats</Text>
          <Text style={styles.sideDesc}>Strategist</Text>
        </TouchableOpacity>
      </View>

      {/* Mode toggle */}
      <Text style={styles.sectionLabel}>Mode</Text>
      <View style={styles.modeRow}>
        <TouchableOpacity
          style={[styles.modeChip, mode === 'ai' && styles.activeChip]}
          onPress={() => setMode('ai')}
        >
          <Text style={styles.modeText}>vs AI</Text>
        </TouchableOpacity>
        <TouchableOpacity
          style={[styles.modeChip, mode === 'pass-and-play' && styles.activeChip]}
          onPress={() => setMode('pass-and-play')}
        >
          <Text style={styles.modeText}>Pass & Play</Text>
        </TouchableOpacity>
      </View>

      {/* Difficulty (vs AI only) */}
      {mode === 'ai' && (
        <>
          <Text style={styles.sectionLabel}>Difficulty</Text>
          <View style={styles.modeRow}>
            {(['easy', 'medium', 'hard'] as Difficulty[]).map(d => (
              <TouchableOpacity
                key={d}
                style={[styles.modeChip, difficulty === d && styles.activeChip]}
                onPress={() => setDifficulty(d)}
              >
                <Text style={styles.modeText}>{d.charAt(0).toUpperCase() + d.slice(1)}</Text>
              </TouchableOpacity>
            ))}
          </View>
        </>
      )}

      <TouchableOpacity
        style={styles.startBtn}
        onPress={() => onStart(mode, side, difficulty)}
      >
        <Text style={styles.startText}>Start Game</Text>
      </TouchableOpacity>
    </View>
  );
}
```

Style the menu with the same dark wood + brass palette. Background `#2C1810`, text gold `#C9A96E`.

**Verify:** All selections work. Tapping "Start Game" calls `onStart` with correct params. Difficulty hidden in pass & play mode.

### Step 25: Wire App.tsx navigation

**Modify:** `App.tsx`

```tsx
import { useState } from 'react';
import { GameMode, Side, Difficulty } from './src/types';
import { MainMenu } from './src/MainMenu';
import { GameScreen } from './src/GameScreen';

type Screen = 'menu' | 'game';

export default function App() {
  const [screen, setScreen] = useState<Screen>('menu');
  const [gameConfig, setGameConfig] = useState<{
    mode: GameMode;
    side: Side;
    difficulty: Difficulty;
  } | null>(null);

  const handleStart = (mode: GameMode, side: Side, difficulty: Difficulty) => {
    setGameConfig({ mode, side, difficulty });
    setScreen('game');
  };

  const handleBackToMenu = () => {
    setScreen('menu');
    setGameConfig(null);
  };

  if (screen === 'menu' || !gameConfig) {
    return <MainMenu onStart={handleStart} />;
  }

  return (
    <GameScreen
      mode={gameConfig.mode}
      side={gameConfig.side}
      difficulty={gameConfig.difficulty}
      onBackToMenu={handleBackToMenu}
    />
  );
}
```

**Verify:** App starts at menu. Start Game → board appears. Back from game → menu. Full loop works.

### Step 26: Add tutorial overlay (first play only)

**Create:** `src/TutorialOverlay.tsx`

```tsx
import { View, Text, TouchableOpacity, StyleSheet } from 'react-native';

export function TutorialOverlay({ onDismiss }: { onDismiss: () => void }) {
  return (
    <View style={styles.overlay}>
      <View style={styles.card}>
        <Text style={styles.title}>Bagh Chal</Text>
        <Text style={styles.body}>
          Tigers hunt. Goats surround.{'\n\n'}
          Tap a piece to select it, then tap where you want it to go.{'\n\n'}
          Tigers capture by jumping over goats. Goats win by trapping all tigers.{'\n\n'}
          Goats move first.
        </Text>
        <TouchableOpacity style={styles.dismissBtn} onPress={onDismiss}>
          <Text style={styles.dismissText}>Got it</Text>
        </TouchableOpacity>
      </View>
    </View>
  );
}
```

Show in `GameScreen` on first mount, conditionally based on `Storage.getStats().tutorialSeen`.

**Verify:** First app launch → tutorial overlay appears. Dismiss → game visible. Second launch → no tutorial.

### Step 27: Add stats persistence

**Create:** `src/Storage.ts`

```tsx
import * as FileSystem from 'expo-file-system';
import { GameStats, Difficulty } from './types';

const STATS_PATH = FileSystem.documentDirectory + 'baghchal_stats.json';

const DEFAULT_STATS: GameStats = {
  gamesPlayed: 0,
  winsAsGoats: 0,
  winsAsTigers: 0,
  losses: 0,
  preferredDifficulty: 'easy',
  tutorialSeen: false,
};

export async function loadStats(): Promise<GameStats> {
  try {
    const data = await FileSystem.readAsStringAsync(STATS_PATH);
    return { ...DEFAULT_STATS, ...JSON.parse(data) };
  } catch {
    return { ...DEFAULT_STATS };
  }
}

export async function saveStats(stats: GameStats): Promise<void> {
  await FileSystem.writeAsStringAsync(STATS_PATH, JSON.stringify(stats));
}
```

Integrate into `GameScreen`: on game over, update stats. On mount, check `tutorialSeen`.

**Verify:** Play a game, close app, reopen. Stats persist. Tutorial only shows once.

### Step 28: Add Game Over modal

**Create:** `src/GameOverModal.tsx`

Polished end-of-game overlay replacing the inline banner from Step 13:

```tsx
export function GameOverModal({ winner, playerSide, onPlayAgain, onMenu }: {
  winner: Player | null;
  playerSide: Side;
  onPlayAgain: () => void;
  onMenu: () => void;
}) {
  const playerWon = winner === playerSide;
  const isDraw = winner === null;

  return (
    <View style={styles.overlay}>
      <Text style={styles.resultEmoji}>
        {isDraw ? '🤝' : playerWon ? '🏆' : '💀'}
      </Text>
      <Text style={styles.resultText}>
        {isDraw ? 'Draw' : playerWon ? 'You Win!' : 'You Lose'}
      </Text>
      <Text style={styles.resultDetail}>
        {isDraw ? '50 moves without capture'
          : winner === 'tiger' ? 'Tigers devoured the goats'
          : 'Goats trapped the tigers'}
      </Text>
      <TouchableOpacity style={styles.btn} onPress={onPlayAgain}>
        <Text style={styles.btnText}>Play Again</Text>
      </TouchableOpacity>
      <TouchableOpacity style={styles.btnSecondary} onPress={onMenu}>
        <Text style={styles.btnSecondaryText}>Main Menu</Text>
      </TouchableOpacity>
    </View>
  );
}
```

**Verify:** Win/loss/draw all show correct modal. Play Again resets. Main Menu navigates back.

### Step 29: Add pause menu

**Create:** `src/PauseMenu.tsx`

Accessed via ⚙️ button in HUD. Shows as a semi-transparent overlay:
- Resume
- Flip Board (pass & play only)
- Concede Game (confirms before acting)
- Back to Main Menu (confirms before acting)

**Verify:** Pause opens from game. Resume works. Concede ends game as loss. Back to menu navigates.

---

## Phase 6: Polish (Stage 6 — Ship It)

### Step 30: Accessibility — reduced motion

Check `AccessibilityInfo.isReduceMotionEnabled()`. When true:
- Piece animations: instant position change + quick fade instead of slide
- Valid move dots: static, no pulsing
- Capture: instant removal, no particles or shake

Pass `reducedMotion` flag through components. Toggle all animation durations to 0 and disable spring physics.

**Verify:** Enable Reduce Motion in device settings. All animations are instant. Re-disable → animations return.

### Step 31: Board flip in Pass & Play

In pass & play mode, add a `flipped` state that rotates the Skia canvas 180° around board center. When toggled:
- All pieces and labels render upside-down
- Tap coordinates are transformed accordingly (invert row/col mapping)
- Turn indicator shows who is "at the bottom"

```tsx
// In BoardCanvas Skia group:
<Group transform={[{ rotate: flipped ? Math.PI : 0 }]} origin={{ x: boardCenterX, y: boardCenterY }}>
  {/* all board content */}
</Group>

// Flip tap coordinates when flipped:
const actualPoint = flipped
  ? { row: 4 - tappedRow, col: 4 - tappedCol }
  : { row: tappedRow, col: tappedCol };
```

**Verify:** Toggle flip board in pause menu. Board rotates 180°. Tapping still works at correct positions.

### Step 32: Performance and edge cases

- [ ] AI search: set 2-second timeout. If depth N exceeds timeout, return best move from depth N-1
- [ ] Memory: check no state updates cause re-render of entire board — use `React.memo` on piece rendering
- [ ] Particles on capture: cap at 8, clean up after animation completes
- [ ] BoardCanvas only re-renders on state change (use `React.memo` with shallow comparison)
- [ ] Handle device rotation gracefully — lock to portrait in `app.json`
- [ ] Test with screen reader (TalkBack / VoiceOver) — pieces announce their type and position
- [ ] Test on both iOS simulator and Android emulator
- [ ] Cold launch <2 seconds on mid-range device

### Step 33: Final test pass

Play through complete games on both platforms:
- [ ] vs AI as goats, all 3 difficulties
- [ ] vs AI as tigers, all 3 difficulties
- [ ] Pass & play, full game with board flips
- [ ] Tutorial shows once, never again
- [ ] Stats persist across cold restarts
- [ ] No console warnings or errors
- [ ] Reduced motion accessibility works
- [ ] Sound respects device silent/vibrate mode

---

## File Manifest (Expected Output)

```
bagh-chal/
├── app.json                          # Expo config, portrait lock, SDK 54
├── App.tsx                           # Entry: menu ↔ game navigation
├── babel.config.js                   # Reanimated plugin
├── tsconfig.json                     # TypeScript config
├── assets/
│   └── sounds/                       # 8 WAV files
│       ├── place.wav
│       ├── move.wav
│       ├── capture.wav
│       ├── invalid.wav
│       ├── transition.wav
│       ├── victory.wav
│       ├── defeat.wav
│       └── select.wav
└── src/
    ├── types.ts                      # All type definitions
    ├── boardGeometry.ts              # Grid constants, adjacency, lines
    ├── gameEngine.ts                 # Pure game logic, rules, state machine
    ├── aiEngine.ts                   # Minimax + alpha-beta + evaluation
    ├── openingBook.ts                # Hard-mode opening positions
    ├── BoardCanvas.tsx               # Skia board + pieces + animations
    ├── GameScreen.tsx                # Game orchestrator, state + AI + feedback
    ├── MainMenu.tsx                  # Side/mode/difficulty selection
    ├── HUD.tsx                       # Turn info, goat count, placement dots, phase
    ├── GameOverModal.tsx             # Win/loss/draw overlay
    ├── PauseMenu.tsx                 # Concede, flip, back to menu
    ├── TutorialOverlay.tsx           # First-play instruction card
    ├── SoundEngine.tsx               # expo-av wrapper
    ├── Haptics.ts                    # expo-haptics wrapper
    └── Storage.ts                    # Stats persistence
```

## Build Time Estimate

| Phase | Steps | ~Hours |
|-------|-------|--------|
| Phase 0: Scaffold | 1–2 | 0.5 |
| Phase 1: Board & Setup | 3–6 | 2.5 |
| Phase 2: Manual Play | 7–13 | 4.5 |
| Phase 3: AI Opponent | 14–18 | 4 |
| Phase 4: Feedback | 19–23 | 2.5 |
| Phase 5: Screens & Flow | 24–29 | 3 |
| Phase 6: Polish | 30–33 | 2 |
| **Total** | **33 steps** | **~19 hours** |
