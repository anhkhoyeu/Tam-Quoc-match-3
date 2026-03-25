# Request for ChatGPT: Performance Optimization Implementation

**Date:** 2026-03-15  
**From:** Figma Make Project  
**To:** ChatGPT (Implementer)  
**Priority:** 🔴 CRITICAL  
**Source:** Technical spec from ChatGPT Performance Expert  
**Reference:** `/src/imports/pasted_text/ai-worker.ts` (full technical spec)

---

## ⚠️ CRITICAL ARCHITECTURE RULES (NON-NEGOTIABLE)

### ✅ Correct Architecture:

```
1 board 8×8 duy nhất
├─ Player và AI chơi chung trên cùng board
├─ UI chỉ đổi: turn owner, highlight AI, panel trái/phải
└─ KHÔNG CÓ playerBoard/enemyBoard, KHÔNG CÓ snapshot board riêng để render
```

### ❌ FORBIDDEN PATTERNS:

```typescript
// ❌ NEVER: Separate boards
state.playerBoard
state.enemyBoard

// ❌ NEVER: Async reducer
case 'AI_TAKE_TURN': {
  const move = await getBestSwap(board); // WRONG!
}

// ❌ NEVER: setTimeout cascade
setTimeout(() => dispatch(...), 300);
setTimeout(() => dispatch(...), 600);
setTimeout(() => dispatch(...), 900);

// ❌ NEVER: AI on main thread
getBestSwap(board); // blocks UI 200-500ms
```

### ✅ REQUIRED PATTERNS:

```typescript
// ✅ CORRECT: Single shared board
state.board // both players use this

// ✅ CORRECT: Reducer stays synchronous
case 'COMMIT_STABLE_BOARD': {
  return { ...state, board: action.board };
}

// ✅ CORRECT: RAF animation pipeline
pipeline.play(steps);
pipeline.onDone(() => {
  dispatch({ type: 'COMMIT_STABLE_BOARD', board: stableBoard });
  dispatch({ type: 'END_ENEMY_TURN' });
});

// ✅ CORRECT: AI in Web Worker
const move = await requestBestSwap(flatBoard); // non-blocking
```

---

## 🎯 IMPLEMENTATION GOAL

Transform current game from:
- ❌ 200-500ms UI freeze during AI turn
- ❌ Jittery animations with setTimeout cascade
- ❌ 60 re-renders/second during pointer drag

To:
- ✅ 60fps smooth animations
- ✅ AI runs in background (main thread never blocked)
- ✅ Responsive drag with < 10 re-renders per gesture

---

## 📋 PHASE 1: Web Worker for AI (CRITICAL)

### 1.1 Create `/src/app/workers/ai.worker.ts`

**Full implementation:**

```typescript
/// <reference lib="webworker" />

type TileType = number; // 0..6 mapping to resource types

type BestSwapRequest = {
  type: "BEST_SWAP";
  requestId: number;
  board: TileType[]; // flat 64-length array
  width: 8;
  height: 8;
};

type BestSwapResponse = {
  type: "BEST_SWAP_RESULT";
  requestId: number;
  move: {
    from: number; // flat index
    to: number;   // flat index
    score: number;
  } | null;
  durationMs: number;
};

const ctx: DedicatedWorkerGlobalScope = self as any;

function swap(board: TileType[], a: number, b: number): TileType[] {
  const next = board.slice();
  [next[a], next[b]] = [next[b], next[a]];
  return next;
}

function hasMatchAt(board: TileType[], width: number, height: number, idx: number): boolean {
  const x = idx % width;
  const y = (idx / width) | 0;
  const t = board[idx];
  if (t < 0) return false;

  // Check horizontal
  let count = 1;
  for (let i = x - 1; i >= 0 && board[y * width + i] === t; i--) count++;
  for (let i = x + 1; i < width && board[y * width + i] === t; i++) count++;
  if (count >= 3) return true;

  // Check vertical
  count = 1;
  for (let j = y - 1; j >= 0 && board[j * width + x] === t; j--) count++;
  for (let j = y + 1; j < height && board[j * width + x] === t; j++) count++;
  return count >= 3;
}

function findMatches(board: TileType[], width: number, height: number): Set<number> {
  const matched = new Set<number>();

  // Horizontal matches
  for (let y = 0; y < height; y++) {
    let runStart = 0;
    for (let x = 1; x <= width; x++) {
      const prev = board[y * width + (x - 1)];
      const curr = x < width ? board[y * width + x] : -999;
      if (curr !== prev) {
        const runLen = x - runStart;
        if (prev >= 0 && runLen >= 3) {
          for (let i = runStart; i < x; i++) matched.add(y * width + i);
        }
        runStart = x;
      }
    }
  }

  // Vertical matches
  for (let x = 0; x < width; x++) {
    let runStart = 0;
    for (let y = 1; y <= height; y++) {
      const prev = board[(y - 1) * width + x];
      const curr = y < height ? board[y * width + x] : -999;
      if (curr !== prev) {
        const runLen = y - runStart;
        if (prev >= 0 && runLen >= 3) {
          for (let j = runStart; j < y; j++) matched.add(j * width + x);
        }
        runStart = y;
      }
    }
  }

  return matched;
}

function scoreBoard(board: TileType[], width: number, height: number): number {
  const matches = findMatches(board, width, height);
  if (matches.size === 0) return -1;

  // Basic score: more matched cells = better
  let score = matches.size * 10;

  // Small bonus for central control
  for (const idx of matches) {
    const x = idx % width;
    const y = (idx / width) | 0;
    const dx = Math.abs(x - 3.5);
    const dy = Math.abs(y - 3.5);
    score += 4 - (dx + dy) * 0.3;
  }

  return score;
}

function getBestSwap(board: TileType[], width: number, height: number) {
  let best: { from: number; to: number; score: number } | null = null;

  for (let y = 0; y < height; y++) {
    for (let x = 0; x < width; x++) {
      const from = y * width + x;

      // Try swap right
      if (x + 1 < width) {
        const to = y * width + (x + 1);
        if (board[from] !== board[to]) {
          const next = swap(board, from, to);
          if (hasMatchAt(next, width, height, from) || hasMatchAt(next, width, height, to)) {
            const score = scoreBoard(next, width, height);
            if (!best || score > best.score) best = { from, to, score };
          }
        }
      }

      // Try swap down
      if (y + 1 < height) {
        const to = (y + 1) * width + x;
        if (board[from] !== board[to]) {
          const next = swap(board, from, to);
          if (hasMatchAt(next, width, height, from) || hasMatchAt(next, width, height, to)) {
            const score = scoreBoard(next, width, height);
            if (!best || score > best.score) best = { from, to, score };
          }
        }
      }
    }
  }

  return best;
}

ctx.onmessage = (event: MessageEvent<BestSwapRequest>) => {
  const msg = event.data;
  if (msg.type !== "BEST_SWAP") return;

  const start = performance.now();
  const move = getBestSwap(msg.board, msg.width, msg.height);
  const durationMs = performance.now() - start;

  const res: BestSwapResponse = {
    type: "BEST_SWAP_RESULT",
    requestId: msg.requestId,
    move,
    durationMs,
  };

  ctx.postMessage(res);
};
```

### 1.2 Create `/src/app/hooks/useAiWorker.ts`

**Full implementation:**

```typescript
import { useEffect, useRef, useState, useCallback } from "react";

type BestMove = { from: number; to: number; score: number } | null;

export function useAiWorker() {
  const workerRef = useRef<Worker | null>(null);
  const requestIdRef = useRef(0);
  const pendingRef = useRef(new Map<number, (move: BestMove) => void>());
  const [lastAiDurationMs, setLastAiDurationMs] = useState(0);

  useEffect(() => {
    const worker = new Worker(
      new URL("../workers/ai.worker.ts", import.meta.url), 
      { type: "module" }
    );
    workerRef.current = worker;

    worker.onmessage = (e) => {
      const { type, requestId, move, durationMs } = e.data;
      if (type === "BEST_SWAP_RESULT") {
        setLastAiDurationMs(durationMs);
        const resolve = pendingRef.current.get(requestId);
        if (resolve) {
          pendingRef.current.delete(requestId);
          resolve(move);
        }
      }
    };

    return () => {
      worker.terminate();
      workerRef.current = null;
    };
  }, []);

  const requestBestSwap = useCallback((board: number[]) => {
    return new Promise<BestMove>((resolve) => {
      const requestId = ++requestIdRef.current;
      pendingRef.current.set(requestId, resolve);
      workerRef.current?.postMessage({
        type: "BEST_SWAP",
        requestId,
        board,
        width: 8,
        height: 8,
      });
    });
  }, []);

  return { requestBestSwap, lastAiDurationMs };
}
```

### 1.3 Integrate into BattleScreen

**Pattern (pseudocode):**

```typescript
// In BattleScreen.tsx
const { requestBestSwap, lastAiDurationMs } = useAiWorker();

useEffect(() => {
  if (state.turnOwner !== 'enemy' || !state.aiThinking) return;
  
  (async () => {
    // 1. Request AI move from worker (non-blocking)
    const flatBoard = flattenBoard(state.board);
    const move = await requestBestSwap(flatBoard);
    
    if (!move) {
      dispatch({ type: 'END_ENEMY_TURN' });
      return;
    }
    
    // 2. Build animation pipeline (precompute all steps)
    const resolve = buildResolvePipeline(state.board, move.from, move.to);
    
    const steps = [
      { type: "ai-highlight", from: move.from, to: move.to, duration: 1200 },
      ...resolve.steps,
    ];
    
    // 3. Play pipeline
    pipeline.play(steps);
    
    // 4. Commit stable board ONLY when pipeline completes
    // NOT during animation
  })();
}, [state.turnOwner, state.aiThinking]);
```

---

## 📋 PHASE 2: RAF Animation Pipeline (CRITICAL)

### 2.1 Core Principle

**MOST IMPORTANT RULE:**

```
Logic state và visual animation state TÁCH NHAU

❌ WRONG: dispatch() nhiều lần trong animation
setTimeout(() => dispatch({ type: 'EXPLODE' }), 300);
setTimeout(() => dispatch({ type: 'FALL' }), 600);
setTimeout(() => dispatch({ type: 'END_TURN' }), 900);

✅ CORRECT: Precompute steps, commit once at end
const { stableBoard, steps } = buildResolvePipeline(board, from, to);
pipeline.play(steps);
pipeline.onDone(() => {
  dispatch({ type: 'COMMIT_STABLE_BOARD', board: stableBoard });
  dispatch({ type: 'END_ENEMY_TURN' });
});
```

### 2.2 Create `/src/app/animations/AnimationPipeline.ts`

**Full implementation:**

```typescript
export type PipelineStep =
  | { type: "ai-highlight"; from: number; to: number; duration: number }
  | { type: "swap"; from: number; to: number; duration: number }
  | { type: "explode"; cells: number[]; duration: number }
  | { type: "fall"; fallMap: Record<number, number>; duration: number };

export class AnimationPipeline {
  private rafId: number | null = null;
  private stepStart = 0;
  private currentStepIndex = 0;
  private running = false;

  constructor(
    private onStepStart: (step: PipelineStep) => void,
    private onStepProgress: (step: PipelineStep, progress: number) => void,
    private onStepEnd: (step: PipelineStep) => void,
    private onDone: () => void
  ) {}

  play(steps: PipelineStep[]) {
    this.stop();
    this.running = true;
    this.currentStepIndex = 0;
    this.stepStart = 0;

    const tick = (now: number) => {
      if (!this.running) return;
      
      const step = steps[this.currentStepIndex];
      if (!step) {
        this.running = false;
        this.onDone();
        return;
      }

      if (this.stepStart === 0) {
        this.stepStart = now;
        this.onStepStart(step);
      }

      const elapsed = now - this.stepStart;
      const progress = Math.min(1, elapsed / step.duration);
      this.onStepProgress(step, progress);

      if (progress >= 1) {
        this.onStepEnd(step);
        this.currentStepIndex++;
        this.stepStart = 0;
      }

      this.rafId = requestAnimationFrame(tick);
    };

    this.rafId = requestAnimationFrame(tick);
  }

  stop() {
    this.running = false;
    if (this.rafId != null) cancelAnimationFrame(this.rafId);
    this.rafId = null;
    this.stepStart = 0;
  }
}
```

### 2.3 Create `/src/app/hooks/useAnimationPipeline.ts`

**Implementation:**

```typescript
import { useRef, useState, useCallback } from 'react';
import { AnimationPipeline, PipelineStep } from '../animations/AnimationPipeline';

export function useAnimationPipeline() {
  const [currentStep, setCurrentStep] = useState<PipelineStep | null>(null);
  const [progress, setProgress] = useState(0);
  const pipelineRef = useRef<AnimationPipeline | null>(null);
  const onDoneCallbackRef = useRef<(() => void) | null>(null);

  if (!pipelineRef.current) {
    pipelineRef.current = new AnimationPipeline(
      (step) => {
        setCurrentStep(step);
        setProgress(0);
      },
      (step, progress) => {
        setProgress(progress);
      },
      (step) => {
        // Step complete
      },
      () => {
        setCurrentStep(null);
        setProgress(0);
        onDoneCallbackRef.current?.();
      }
    );
  }

  const play = useCallback((steps: PipelineStep[], onDone?: () => void) => {
    onDoneCallbackRef.current = onDone || null;
    pipelineRef.current?.play(steps);
  }, []);

  const stop = useCallback(() => {
    pipelineRef.current?.stop();
    setCurrentStep(null);
    setProgress(0);
  }, []);

  return { play, stop, currentStep, progress };
}
```

### 2.4 Add `buildResolvePipeline()` to boardEngine

**Add to `/src/app/game/boardEngine.ts`:**

```typescript
export type ResolveResult = {
  stableBoard: Cell[][];
  steps: PipelineStep[];
};

export function buildResolvePipeline(
  board: Cell[][], 
  fromRow: number, 
  fromCol: number, 
  toRow: number, 
  toCol: number
): ResolveResult {
  const steps: PipelineStep[] = [];
  let currentBoard = deepClone(board);
  
  // 1. Swap step
  const fromFlat = fromRow * 8 + fromCol;
  const toFlat = toRow * 8 + toCol;
  steps.push({ type: 'swap', from: fromFlat, to: toFlat, duration: 140 });
  
  // Apply swap to logic
  [currentBoard[fromRow][fromCol], currentBoard[toRow][toCol]] = 
  [currentBoard[toRow][toCol], currentBoard[fromRow][fromCol]];
  
  // 2. Find matches and cascade until stable
  let iterations = 0;
  while (iterations < 10) { // Safety limit
    const matches = findAllMatches(currentBoard);
    if (matches.length === 0) break;
    
    // Explode step
    const matchedIndices = matches.flatMap(cluster => 
      cluster.cells.map(cell => cell.row * 8 + cell.col)
    );
    steps.push({ type: 'explode', cells: matchedIndices, duration: 180 });
    
    // Apply removal + gravity
    const { board: nextBoard, fallMap } = removeAndFill(
      currentBoard,
      matches,
      null, // spawnPriority
      0,    // spawnPriorityCount
      [],   // presetSpawns
      false // quiDaoActive
    );
    
    // Fall step
    if (Object.keys(fallMap).length > 0) {
      steps.push({ type: 'fall', fallMap, duration: 220 });
    }
    
    currentBoard = nextBoard;
    iterations++;
  }
  
  return {
    stableBoard: currentBoard,
    steps,
  };
}
```

---

## 📋 PHASE 3: React Optimizations

### 3.1 Fix Pointer Drag (HIGH PRIORITY)

**Current problem in `/src/app/components/SharedBoard.tsx`:**

```typescript
// ❌ WRONG: setState on EVERY pointermove = 60 re-renders/sec
const handlePointerMove = (e: React.PointerEvent) => {
  setDragState({
    offsetX: e.clientX - startX,
    offsetY: e.clientY - startY,
  });
};
```

**Required fix:**

```typescript
// ✅ CORRECT: Only setState when target CELL changes

const dragRef = useRef<{
  active: boolean;
  startIndex: number | null;
  currentIndex: number | null;
}>({
  active: false,
  startIndex: null,
  currentIndex: null,
});

const [dragState, setDragState] = useState<{ 
  start: number | null; 
  target: number | null 
}>({
  start: null,
  target: null,
});

function onPointerDown(idx: number) {
  dragRef.current = { active: true, startIndex: idx, currentIndex: idx };
  setDragState({ start: idx, target: idx });
}

function onPointerMove(e: React.PointerEvent) {
  if (!dragRef.current.active) return;
  
  const idx = getCellIndexFromPoint(e.clientX, e.clientY);
  if (idx == null || idx === dragRef.current.currentIndex) return; // ✅ Skip redundant updates
  
  dragRef.current.currentIndex = idx;
  setDragState((prev) => (prev.target === idx ? prev : { ...prev, target: idx }));
}

function onPointerUp() {
  const { startIndex, currentIndex } = dragRef.current;
  dragRef.current.active = false;
  
  if (startIndex != null && currentIndex != null && startIndex !== currentIndex) {
    attemptSwap(startIndex, currentIndex);
  }
  
  setDragState({ start: null, target: null });
}

// Helper to convert pointer coords to cell index
function getCellIndexFromPoint(x: number, y: number): number | null {
  const boardRect = boardRef.current?.getBoundingClientRect();
  if (!boardRect) return null;
  
  const relX = x - boardRect.left;
  const relY = y - boardRect.top;
  
  const cellWidth = boardRect.width / 8;
  const cellHeight = boardRect.height / 8;
  
  const col = Math.floor(relX / cellWidth);
  const row = Math.floor(relY / cellHeight);
  
  if (row < 0 || row >= 8 || col < 0 || col >= 8) return null;
  return row * 8 + col;
}
```

### 3.2 Memoize Board Tiles

**Update `/src/app/components/TileIcon.tsx`:**

```typescript
type BoardTileProps = {
  idx: number;
  type: number;
  selected: boolean;
  preview: boolean;
  exploding: boolean;
  offsetY: number;
  onPointerDown: (idx: number) => void;
};

export const BoardTile = React.memo(function BoardTile({
  idx,
  type,
  selected,
  preview,
  exploding,
  offsetY,
  onPointerDown,
}: BoardTileProps) {
  return (
    <button
      className="relative w-12 h-12 md:w-16 md:h-16 touch-none"
      onPointerDown={() => onPointerDown(idx)}
      style={{
        transform: `translateY(${offsetY}px)`,
        willChange: "transform, opacity",
      }}
    >
      {/* tile visual */}
      <TileIcon type={type} />
      {exploding && <div className="absolute inset-0 animate-ping bg-yellow-400/50" />}
    </button>
  );
}, (prev, next) => {
  return (
    prev.type === next.type &&
    prev.selected === next.selected &&
    prev.preview === next.preview &&
    prev.exploding === next.exploding &&
    prev.offsetY === next.offsetY
  );
});
```

### 3.3 Memoize Derived Sets

**In SharedBoard.tsx:**

```typescript
const selectedSet = useMemo(
  () => new Set(selected != null ? [selected] : []), 
  [selected]
);

const previewSet = useMemo(
  () => new Set(preview != null ? [preview] : []), 
  [preview]
);

const explodingSet = useMemo(
  () => new Set(explodingCells), 
  [explodingCells]
);
```

### 3.4 Use startTransition for Logs

**Update log updates:**

```typescript
import { startTransition } from 'react';

// Whenever adding battle logs
startTransition(() => {
  setBattleLog((prev) => [...prev, newLogMessage]);
});
```

---

## 📋 PHASE 4: Performance Monitoring

### 4.1 Create `/src/app/hooks/useFrameMonitor.ts`

**Full implementation:**

```typescript
import { useEffect, useRef, useState } from "react";

export function useFrameMonitor() {
  const [fps, setFps] = useState(60);
  const [slowFrames, setSlowFrames] = useState(0);
  const frameCount = useRef(0);
  const slowCount = useRef(0);
  const lastTime = useRef(performance.now());
  const lastReport = useRef(performance.now());

  useEffect(() => {
    let raf = 0;

    const loop = (now: number) => {
      const dt = now - lastTime.current;
      lastTime.current = now;
      frameCount.current++;

      if (dt > 16.7) slowCount.current++;

      if (now - lastReport.current >= 1000) {
        const elapsed = now - lastReport.current;
        const currentFps = Math.round((frameCount.current * 1000) / elapsed);
        setFps(currentFps);
        setSlowFrames(slowCount.current);

        frameCount.current = 0;
        slowCount.current = 0;
        lastReport.current = now;
      }

      raf = requestAnimationFrame(loop);
    };

    raf = requestAnimationFrame(loop);
    return () => cancelAnimationFrame(raf);
  }, []);

  return { fps, slowFrames };
}
```

### 4.2 Create `/src/app/components/PerfHud.tsx`

**Full implementation:**

```typescript
export function PerfHud({
  fps,
  slowFrames,
  aiMs,
}: {
  fps: number;
  slowFrames: number;
  aiMs: number;
}) {
  return (
    <div className="fixed top-2 right-2 z-50 rounded bg-black/80 px-3 py-2 text-xs text-white font-mono">
      <div>FPS: {fps}</div>
      <div>Slow frames: {slowFrames}</div>
      <div>AI: {aiMs.toFixed(1)}ms</div>
    </div>
  );
}
```

### 4.3 Add Long Task Observer

**Create `/src/app/utils/observeLongTasks.ts`:**

```typescript
export function observeLongTasks() {
  if (!("PerformanceObserver" in window)) return;

  const observer = new PerformanceObserver((list) => {
    for (const entry of list.getEntries()) {
      console.warn("[longtask]", {
        duration: entry.duration,
        start: entry.startTime,
      });
    }
  });

  observer.observe({ entryTypes: ["longtask"] });
  return () => observer.disconnect();
}
```

### 4.4 Integrate PerfHud in BattleScreen

```typescript
import { useFrameMonitor } from '../hooks/useFrameMonitor';
import { PerfHud } from './PerfHud';

function BattleScreen() {
  const { fps, slowFrames } = useFrameMonitor();
  const { lastAiDurationMs } = useAiWorker();
  
  return (
    <>
      <PerfHud fps={fps} slowFrames={slowFrames} aiMs={lastAiDurationMs} />
      {/* rest of UI */}
    </>
  );
}
```

---

## 📂 FILES SUMMARY

### Files to CREATE:

```
/src/app/
  workers/
    └─ ai.worker.ts              ✅ Full code provided above
  hooks/
    ├─ useAiWorker.ts            ✅ Full code provided above
    ├─ useAnimationPipeline.ts   ✅ Full code provided above
    └─ useFrameMonitor.ts        ✅ Full code provided above
  animations/
    └─ AnimationPipeline.ts      ✅ Full code provided above
  components/
    └─ PerfHud.tsx               ✅ Full code provided above
  utils/
    └─ observeLongTasks.ts       ✅ Full code provided above
```

### Files to MODIFY:

```
/src/app/game/
  ├─ boardEngine.ts              ⚠️ Add buildResolvePipeline()
  └─ gameReducer.ts              ⚠️ Remove AI_TAKE_TURN, add COMMIT_STABLE_BOARD

/src/app/components/
  ├─ BattleScreen.tsx            ⚠️ Integrate worker + pipeline
  ├─ SharedBoard.tsx             ⚠️ Fix pointer tracking with refs
  └─ TileIcon.tsx                ⚠️ Add React.memo
```

---

## ✅ ACCEPTANCE CRITERIA

After implementation, verify:

- [ ] **AI does not freeze UI** - Can see smooth highlight before AI swap executes
- [ ] **Swap animation 60fps** - No jank during tile swap
- [ ] **Explosion animation 60fps** - No jank during particle effects
- [ ] **Tiles falling 60fps** - Smooth gravity animation
- [ ] **Pointer drag responsive** - < 10 re-renders during drag gesture
- [ ] **FPS counter shows 60fps** - Visible in PerfHud
- [ ] **No console errors** - Clean console during gameplay
- [ ] **Game logic still correct** - No bugs in match detection/scoring
- [ ] **Single shared board only** - No playerBoard/enemyBoard in code

---

## 🎯 IMPLEMENTATION ORDER (RECOMMENDED)

```
Priority 1 (CRITICAL - 2h):
✅ Create AI worker (ai.worker.ts + useAiWorker.ts)
✅ Integrate into BattleScreen
✅ Verify AI doesn't freeze UI

Priority 2 (HIGH - 2h):
✅ Create AnimationPipeline.ts + hook
✅ Add buildResolvePipeline() to boardEngine
✅ Replace setTimeout cascade in BattleScreen
✅ Verify smooth animations

Priority 3 (MEDIUM - 1h):
✅ Fix pointer drag with refs
✅ Add React.memo to BoardTile
✅ Add useMemo for derived sets
✅ Verify reduced re-renders

Priority 4 (POLISH - 1h):
✅ Add useFrameMonitor + PerfHud
✅ Add observeLongTasks
✅ Add startTransition for logs
✅ Verify metrics visible
```

**Total effort:** 5-6 hours  
**Expected outcome:** 60fps production-ready performance

---

## 📝 HELPER UTILITIES

### Flatten Board for Worker

```typescript
const ITEM_TYPES: BoardItemType[] = [
  'dao_kiem', 'linh_duoc', 'binh_thu', 'trong_tran', 
  'quan_ky', 'giap_tru', 'am_khi'
];

function flattenBoard(board: Cell[][]): number[] {
  return board.flat().map(cell => ITEM_TYPES.indexOf(cell.type));
}
```

### Convert Flat Index to Row/Col

```typescript
function flatToRowCol(idx: number): [number, number] {
  return [Math.floor(idx / 8), idx % 8];
}

function rowColToFlat(row: number, col: number): number {
  return row * 8 + col;
}
```

---

## 🚨 CRITICAL REMINDERS

1. **Reducer MUST stay synchronous** - No async/await in gameReducer.ts
2. **Single shared board only** - No playerBoard/enemyBoard
3. **Commit stable board ONCE at end** - Not during animation
4. **RAF for all animations** - No setTimeout for timing
5. **AI in Web Worker** - Never block main thread

---

## 📞 QUESTIONS?

If unclear, ask BEFORE implementing. Key principles:

- ✅ Async orchestration lives in **components/hooks**, not reducer
- ✅ Animation state separate from game logic state
- ✅ Precompute all steps, play pipeline, commit at end
- ✅ Use refs for high-frequency updates (pointer tracking)
- ✅ Memo components to prevent unnecessary re-renders

---

**Thank you!** 🚀  
We're excited to see 60fps smooth gameplay!

---

*Request created: 2026-03-15*  
*Source: ChatGPT Performance Expert Technical Spec*  
*Full spec reference: `/src/imports/pasted_text/ai-worker.ts`*
