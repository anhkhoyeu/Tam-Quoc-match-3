# Performance Implementation Status Report
**Game:** Tam Quốc Match-3 (Figma Make)  
**Date:** 2026-03-15  
**Reporter:** AI Assistant → GPT Reviewer  
**Spec Reference:** `/src/imports/pasted_text/react-perf-spec.md`

---

## ⚠️ CRITICAL CORRECTION FROM GPT REVIEWER

**DO NOT make gameReducer async.**  
**Keep reducer pure and synchronous.**

### ✅ Correct Architecture:

```
AI Worker orchestration must live in:
├─ BattleScreen.tsx (current approach), OR
└─ useGameController.ts (preferred - cleaner separation)

Correct flow:
1. Request worker result (async, outside reducer)
2. Build RAF pipeline with precomputed steps
3. Play highlight → swap → explode → fall animations
4. Commit stable board to reducer (synchronous dispatch)
5. End enemy turn (synchronous dispatch)
```

### ❌ Wrong Approaches to Avoid:

- ❌ `async function gameReducer()` - breaks Redux principles
- ❌ `await` inside reducer cases
- ❌ Board switching / separate enemy board
- ❌ Timeout-based animation cascade
- ❌ AI compute on main thread

### ✅ Enforced Rules:

1. **Single shared board only** - no playerBoard/enemyBoard
2. **Reducer stays synchronous** - no async/await in reducer
3. **No setTimeout cascade** - use RAF pipeline
4. **No AI on main thread** - use Web Worker

---

## 📊 EXECUTIVE SUMMARY

| Category | Status | Progress |
|----------|--------|----------|
| **Architecture** | ✅ **DONE** | Single shared board implemented |
| **AI Worker** | ❌ **NOT STARTED** | Still blocking main thread |
| **Animation Pipeline** | ❌ **NOT STARTED** | Still using setTimeout cascade |
| **React Optimizations** | ⚠️ **PARTIAL** | Some memo, missing pointer ref optimization |
| **Performance Monitoring** | ❌ **NOT STARTED** | No FPS tracking, no profiling |

**Overall Progress:** 🟡 **2/7 major items complete (28%)**

---

## 0️⃣ ARCHITECTURE CORRECTION ✅ **DONE**

### ✅ Single Shared Board (COMPLETED)

**Evidence:**
```typescript
// /src/app/game/gameReducer.ts:1-11
// GAME REDUCER — TAM QUỐC MATCH-3 v2.2 (SINGLE SHARED BOARD)
// - Removed separate playerBoard/enemyBoard ✅
// - Now uses single shared state.board for BOTH players ✅
// - Player and Enemy compete on the SAME board ✅
```

**Status:** ✅ **Architecture is CORRECT**  
**No action needed** - this was already refactored.

---

## 1️⃣ WEB WORKER FOR AI ❌ **NOT STARTED**

### Current Implementation (BLOCKING MAIN THREAD)

**Location:** `/src/app/game/boardEngine.ts:289-355`

```typescript
// ❌ BLOCKING: Runs on main thread, freezes UI 200-500ms
export function getBestSwap(board: Cell[][]): [[number,number],[number,number]] | null {
  console.time('getBestSwap');
  let best: [[number,number],[number,number]] | null = null;
  let bestScore = -1;
  let swapCount = 0;
  
  // ❌ Brute-force loop: 64 cells × 2 directions × scoring = 200-500ms
  for (let r = 0; r < SIZE; r++) {
    for (let c = 0; c < SIZE; c++) {
      // ... swap simulation ...
    }
  }
  
  console.timeEnd('getBestSwap');
  return best;
}
```

**Called from:** `/src/app/game/gameReducer.ts:1481`
```typescript
case 'AI_TAKE_TURN': {
  let best: [[number,number],[number,number]] | null = null;
  try {
    best = getBestSwap(s.board); // ❌ BLOCKS UI THREAD HERE
  } catch (err) {
    console.error('❌ AI getBestSwap crashed:', err);
  }
  // ...
}
```

### Required Files (PER SPEC)

#### ❌ `/src/app/workers/ai.worker.ts` (NOT EXISTS)
Needs to be created with:
- `TileType = number` (flat array format)
- `BestSwapRequest` / `BestSwapResponse` types
- `getBestSwap()` logic ported to worker
- `ctx.onmessage` handler

#### ❌ `/src/app/hooks/useAiWorker.ts` (NOT EXISTS)
Needs to be created with:
- Worker initialization
- `requestBestSwap(board: number[])` async function
- Promise-based request/response mapping
- `lastAiDurationMs` tracking

### Integration Points

**Must modify:**
1. ❌ `/src/app/game/gameReducer.ts` - **DO NOT make async** - Remove AI_TAKE_TURN case entirely
2. ✅ `/src/app/components/BattleScreen.tsx` - Add AI worker orchestration logic
3. ✅ Optional: Create `/src/app/hooks/useGameController.ts` for cleaner separation

**Correct implementation pattern:**

```typescript
// ✅ CORRECT: BattleScreen.tsx orchestrates AI turn
useEffect(() => {
  if (!isPlayerTurn && state.aiThinking) {
    (async () => {
      // 1. Request worker result (async, outside reducer)
      const move = await requestBestSwap(flattenBoard(state.board));
      
      // 2. Build RAF pipeline with precomputed steps
      const steps = buildResolvePipeline(state.board, move);
      
      // 3. Play animations
      pipeline.play(steps);
      
      // 4. After pipeline completes, commit stable board
      pipeline.onDone(() => {
        dispatch({ type: 'COMMIT_AI_BOARD', board: stableBoard });
        dispatch({ type: 'END_ENEMY_TURN' }); // synchronous
      });
    })();
  }
}, [isPlayerTurn, state.aiThinking]);
```

```typescript
// ❌ WRONG: Do not do this in gameReducer
case 'AI_TAKE_TURN': {
  const move = await getBestSwap(s.board); // ❌ async in reducer
  // ...
}
```

**Estimated effort:** 1.5 hours

---

## 2️⃣ ANIMATION PIPELINE ❌ **NOT STARTED**

### Current Implementation (setTimeout CASCADE)

**Location:** `/src/app/components/BattleScreen.tsx:104-149`

```typescript
// ❌ CASCADING TIMEOUTS: Causes jank between animation phases

// Timer 1: AI thinking delay
aiTimerRef.current = setTimeout(() => {
  dispatch({ type: 'AI_TAKE_TURN' });
}, 150);

// Timer 2: AI swap execution
aiSwapTimerRef.current = setTimeout(() => {
  dispatch({ type: 'AI_EXECUTE_SWAP' });
}, 2000);

// Timer 3: Explosion VFX (in gameReducer)
setTimeout(() => {
  setEnemyExplodingCells(new Set());
}, 320);

// Timer 4: End turn (in gameReducer)
setTimeout(() => {
  dispatch({ type: 'END_ENEMY_TURN' });
}, 300);
```

**Problems:**
- ❌ No frame synchronization
- ❌ Timers can drift/overlap
- ❌ Hard to cancel/restart pipeline
- ❌ State updates between animations cause re-renders

### Required Files (PER SPEC)

#### ❌ `/src/app/animations/AnimationPipeline.ts` (NOT EXISTS)
Needs to be created with:
```typescript
type PipelineStep =
  | { type: "ai-highlight"; from: number; to: number; duration: number }
  | { type: "swap"; from: number; to: number; duration: number }
  | { type: "explode"; cells: number[]; duration: number }
  | { type: "fall"; fallMap: Record<number, number>; duration: number };

export class AnimationPipeline {
  private rafId: number | null = null;
  
  play(steps: PipelineStep[]) {
    // requestAnimationFrame loop
  }
  
  stop() {
    cancelAnimationFrame(this.rafId);
  }
}
```

#### ❌ `/src/app/hooks/useAnimationPipeline.ts` (NOT EXISTS)
React hook wrapper for pipeline

### Integration Points

**Must modify:**
1. `/src/app/components/BattleScreen.tsx` - Replace setTimeout with pipeline.play()
2. `/src/app/game/gameReducer.ts` - Add `buildResolvePipeline()` function

**Estimated effort:** 2 hours

---

## 3️⃣ REACT OPTIMIZATIONS ⚠️ **PARTIAL**

### 3.1 Drag Gesture Optimization ❌ **NOT DONE**

**Current Implementation:** `/src/app/components/SharedBoard.tsx:45-128`

```typescript
// ❌ PERFORMANCE ISSUE: setState on EVERY pointermove event
const [dragState, setDragState] = useState<{
  row: number;
  col: number;
  startX: number;
  startY: number;
  offsetX: number;
  offsetY: number;
} | null>(null);

const handlePointerMove = useCallback((e: React.PointerEvent) => {
  if (!dragState) return;
  e.preventDefault();
  setDragState({
    ...dragState,
    offsetX: e.clientX - dragState.startX,  // ❌ RE-RENDERS 60 TIMES/SEC
    offsetY: e.clientY - dragState.startY,
  });
}, [dragState]);
```

**Required Fix (PER SPEC):**
```typescript
// ✅ Store pointer position in ref (no re-render)
const dragRef = useRef<{
  active: boolean;
  startIndex: number | null;
  currentIndex: number | null;
}>(/* ... */);

// ✅ Only setState when target CELL changes
function onPointerMove(e: React.PointerEvent) {
  if (!dragRef.current.active) return;
  const idx = getCellIndexFromPoint(e.clientX, e.clientY);
  if (idx == null || idx === dragRef.current.currentIndex) return; // ✅ Skip redundant updates
  
  dragRef.current.currentIndex = idx;
  setDragState((prev) => (prev.target === idx ? prev : { ...prev, target: idx }));
}
```

**Status:** ❌ Not implemented  
**Estimated effort:** 30 minutes

---

### 3.2 Tile Memoization ⚠️ **PARTIAL**

**Current Implementation:** `/src/app/components/TileIcon.tsx`

```typescript
// ⚠️ BoardTile is NOT memoized
export function BoardTile({ type, selected, preview, exploding, /* ... */ }: Props) {
  // ❌ Re-renders even when props unchanged
  return <button className="..." />;
}
```

**Required Fix (PER SPEC):**
```typescript
export const BoardTile = React.memo(function BoardTile({
  idx, type, selected, preview, exploding, offsetY, onPointerDown
}: BoardTileProps) {
  return <button /* ... */ />;
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

**Status:** ❌ Not implemented  
**Estimated effort:** 45 minutes

---

### 3.3 Batch Updates ✅ **POSSIBLY DONE**

React 18 auto-batches most state updates, but need to verify async callbacks use `unstable_batchedUpdates`.

**Status:** 🟡 Needs verification  
**Estimated effort:** 15 minutes to verify

---

### 3.4 startTransition for Logs ❌ **NOT DONE**

**Current Implementation:**
```typescript
// ❌ Log updates can block gameplay state updates
dispatch({ type: 'ADD_LOG', message: '...' });
```

**Required Fix:**
```typescript
import { startTransition } from 'react';

startTransition(() => {
  setBattleLog((prev) => [...prev, "AI moved"]);
});
```

**Status:** ❌ Not implemented  
**Estimated effort:** 15 minutes

---

## 4️⃣ PERFORMANCE MONITORING ❌ **NOT STARTED**

### Required Files (PER SPEC)

#### ❌ `/src/app/hooks/useFrameMonitor.ts` (NOT EXISTS)
```typescript
export function useFrameMonitor() {
  // requestAnimationFrame loop to track FPS
  return { fps, slowFrames };
}
```

#### ❌ `/src/app/components/PerfHud.tsx` (NOT EXISTS)
```tsx
export function PerfHud({ fps, slowFrames, aiMs }: Props) {
  return (
    <div className="fixed top-2 right-2 ...">
      <div>FPS: {fps}</div>
      <div>Slow frames: {slowFrames}</div>
      <div>AI: {aiMs.toFixed(1)}ms</div>
    </div>
  );
}
```

#### ❌ Long Task Observer (NOT IMPLEMENTED)
Need to add `observeLongTasks()` utility

**Status:** ❌ Not started  
**Estimated effort:** 1 hour total

---

## 📋 PRIORITY ACTION ITEMS

### 🔴 CRITICAL (Blocks UX)
1. **Implement Web Worker for AI** (1.5h)
   - ❌ Create `/src/app/workers/ai.worker.ts`
   - ❌ Create `/src/app/hooks/useAiWorker.ts`
   - ❌ Modify `gameReducer.ts` AI_TAKE_TURN action
   - **Impact:** Eliminates 200-500ms UI freeze

### 🟡 HIGH (Quality of Life)
2. **Implement RAF Animation Pipeline** (2h)
   - ❌ Create `/src/app/animations/AnimationPipeline.ts`
   - ❌ Modify `BattleScreen.tsx` to use pipeline
   - **Impact:** Smooth 60fps animations, no jank

3. **Fix Pointer Drag Performance** (30min)
   - ❌ Refactor `SharedBoard.tsx` pointer tracking to use refs
   - **Impact:** Responsive drag gesture

### 🟢 MEDIUM (Polish)
4. **Add Tile Memoization** (45min)
   - ❌ Memo BoardTile component
   - **Impact:** Reduce re-renders from 64 → 2-10 cells

5. **Add Performance Monitoring** (1h)
   - ❌ Create useFrameMonitor hook
   - ❌ Create PerfHud component
   - **Impact:** Visibility into performance regressions

---

## 🎯 NEXT STEPS FOR GPT

**Recommended implementation order:**

```
Phase 1 (URGENT - 1.5h):
✅ Implement Web Worker for AI
   └─ Files: ai.worker.ts, useAiWorker.ts
   └─ Modify: gameReducer.ts, BattleScreen.tsx

Phase 2 (HIGH - 2h):
✅ Implement RAF Animation Pipeline
   └─ Files: AnimationPipeline.ts, useAnimationPipeline.ts
   └─ Modify: BattleScreen.tsx, gameReducer.ts

Phase 3 (POLISH - 1.5h):
✅ Fix pointer drag (30min)
✅ Add tile memo (45min)
✅ Add startTransition (15min)

Phase 4 (OPTIONAL - 1h):
✅ Add performance monitoring
```

**Total estimated effort:** 5-6 hours  
**Current completion:** 28% (2/7 items)  
**Remaining work:** 4-5 hours

---

## ✅ WHAT'S ALREADY GOOD

1. ✅ **Single shared board architecture** - correct from the start
2. ✅ **No separate enemy board rendering** - no wasted work
3. ✅ **React 18 auto-batching** - helps reduce some re-renders
4. ✅ **CSS transforms for animations** - GPU accelerated
5. ✅ **will-change hints** - already added in recent optimizations

---

## 📝 NOTES FOR GPT

- **User complaint:** "Game không mượt" at 3 points:
  1. Swap lag (pointer tracking issue)
  2. Animation giật (setTimeout cascade)
  3. AI freeze UI (main thread blocking)

- **User preference:** Willing to "đập đi xây lại" for performance

- **Constraints:**
  - Must keep Figma design (525×599 board frame)
  - Must stay React/Tailwind (no Canvas rewrite)
  - Must preserve existing UI/UX flow

- **Environment:**
  - Vite build system
  - React 18.3.1
  - TypeScript
  - No special bundler plugins needed for Web Worker (Vite handles `new Worker()`)

---

## 🚀 READY TO IMPLEMENT

All spec requirements are clear. No blockers identified.  
User is ready for full implementation following the spec.

**Awaiting GPT's implementation of Phase 1-4.**

---

*Last updated: 2026-03-15*  
*Reporter: Figma Make AI Assistant*  
*Spec author: ChatGPT Performance Expert*