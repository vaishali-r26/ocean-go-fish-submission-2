> A full development log documenting the Ocean Go Fish browser game — bug identification, root cause analysis, implemented fixes, and key React state management learnings. Written in a format compatible with submission and code review standards.

**Project:** Ocean Go Fish — React + TypeScript + Vite
**Status:** ✅ Stable — all known bugs resolved
**Date:** April 2026

---

## Project Overview

Ocean Go Fish is a browser-based, single-player implementation of the classic *Go Fish* card game, reimagined with an ocean theme. The player competes against an AI opponent, requesting animal cards (Clownfish, Crab, Octopus, Turtle, Starfish) and collecting sets of four. The first to collect the most sets wins.

### Architecture

| Layer | Technology |
| ----- | ---------- |
| UI Framework | React 19 (functional components, hooks) |
| Language | TypeScript 5.8 (strict mode) |
| Build Tool | Vite 6 (ESM, HMR) |
| Animation | Framer Motion (`motion` v12) |
| Styling | TailwindCSS 4 + custom CSS |
| Icons | Lucide React |

### Game Domain

* **Deck:** 20 cards total — 4 cards per animal × 5 animals
* **Initial Deal:** 5 cards each (player + AI)
* **Win Condition:** Collect the most sets of 4 before all sets are claimed
* **Turn Flow:** Player asks → AI responds → cards transfer → set check → next turn

---

## Features Implemented

### Core Gameplay

* Full turn engine — player and AI take alternating turns following canonical Go Fish rules
* Card request system — player selects a card from hand to ask the AI for a matching animal
* AI opponent logic — AI randomly selects a card from its hand and asks the player
* Go Fish mechanics — when the requested animal is not held, the asker draws from the deck
* "Lucky draw" rule — if you draw the exact animal you asked for, you take another turn
* Set completion — collecting all 4 cards of the same animal removes them as a scored set
* Empty hand handling — when a player's hand empties, they draw from deck before their next ask
* Game over detection — triggers when all sets are claimed or deck + both hands are exhausted

### AI Interaction Flow

* Player response mode — when AI asks, player sees `Yes, I do!` / `No, Go Fish!` buttons
* Integrity validation — prevents player from lying (alerts if they claim "no" when they hold the card)
* Card selection UI — when player says "yes," they manually select and confirm cards to hand over
* All-or-nothing enforcement — player must give all matching cards, not just one

### UI / UX

* Ocean-themed background — animated bubbles, seaweed, light rays
* Animated card hand — Framer Motion `layoutId` transitions for card movement
* Game message board — animated message transitions with turn indicator bar
* Scoreboard — real-time player vs AI set count
* Collected sets panel — scrollable with animal thumbnails
* Game over modal — full-screen overlay with final score and "Play Again"
* Image preloading — all animal images eagerly preloaded on mount
* Graceful image fallback — shows animal initial letter if image fails to load

---

## Bugs Identified

Three distinct categories of bugs were found during development and testing.

### Bug #1 — Duplicate Sets (Critical)

**Observed:** A player or AI would show a score of `2` for a single completed set, or a previously scored animal would reappear in the sets tray.

**Steps to reproduce:**
1. Start a new game.
2. Collect 3 cards of one animal through normal asking.
3. Ask for the 4th — receive it from the AI.
4. Observe the set was counted twice (score incremented by 2, or same animal icon appeared twice).

### Bug #2 — Permanent AI Deadlock / Freeze (Critical)

**Observed:** After a player's "Go Fish!" draw, the UI displayed `"AI is drawing…"` indefinitely. No further turns occurred.

**Steps to reproduce:**
1. Ask for an animal the AI does not have.
2. Draw a card from the deck (the "Go Fish" draw).
3. Observe the game freezes — AI never takes its turn.

### Bug #3 — Stale Closure in Async Handlers (Medium)

**Observed:** Card state read inside `handleAsk` reflected the hand at the time of the function call, not current state — causing incorrect set detection after multi-step async sequences.

---

## Root Cause Analysis

### Bug #1 — React State Updater Double-Invocation

The original implementation performed set detection inside `setState` functional updaters:

```tsx
// ❌ BUGGY — set detection inside updater
// React may call this multiple times in Strict Mode / concurrent features
setPlayerHand(prev => {
  const newHand = [...prev, ...matches];
  const completed = findCompletedSets(newHand);
  if (completed.length > 0) {
    setPlayerSets(s => [...s, ...completed]); // side-effect inside updater = double execution
  }
  return newHand.filter(card => !completed.includes(card.animal));
});
```

React's documentation explicitly warns against side effects inside state updater functions. In React 18+ with `StrictMode`, updaters can be called multiple times to verify purity. Since `setPlayerSets` was invoked as a side effect inside `setPlayerHand`'s updater, it was called twice — resulting in duplicate set entries.

A parallel `useEffect` watching `playerHand` was also performing set detection:

```tsx
// ❌ Redundant fallback — fires after every hand change, duplicating set detection
useEffect(() => {
  const completed = findCompletedSets(playerHand);
  if (completed.length > 0) {
    setPlayerSets(prev => [...prev, ...completed]); // duplicate with inline detection
    setPlayerHand(prev => prev.filter(c => !completed.includes(c.animal)));
  }
}, [playerHand]);
```

This `useEffect` fired *after* the explicit set detection in the event handler, detecting sets that had already been removed — and because the sets array was not deduplicated, the same animal appeared multiple times in the score.

### Bug #2 — `isProcessing` in `useEffect` Dependency Array

The turn engine `useEffect` included `isProcessing` in its dependency array:

```tsx
// ❌ BUGGY dependency array — self-cancelling loop
useEffect(() => {
  if (isProcessing) return;

  if (turn === 'ai') {
    setIsProcessing(true); // ← triggers the effect to re-run immediately
    const t = setTimeout(() => {
      setIsProcessing(false);
    }, 800);
  }
}, [turn, status, isProcessing]); // isProcessing here causes immediate re-run
```

When the effect called `setIsProcessing(true)`, `isProcessing` being in the dependency array caused the effect to re-run immediately — before the `setTimeout` callback ever fired. The re-run saw `isProcessing === true` and returned early — permanently. The timer was orphaned.

### Bug #3 — Stale Closures in Async Handlers

`handleAsk` is an `async` function. Each `await` suspends execution and resumes in a future microtask. If state updated during that suspension, React had already re-rendered — but the handler's closure still held old `playerHand` and `aiHand` values captured at call time.

```tsx
// ❌ Stale — aiHand captured at call time, not when the await resolves
const handleAsk = async (animal) => {
  await new Promise(r => setTimeout(r, 1000));
  const matches = aiHand.filter(card => card.animal === animal); // aiHand is stale
};
```

---

## Fixes Implemented

### Fix #1 — Compute Outside Updater + Deduplicate

All set detection was moved *outside* React state updater functions. Set detection now runs as a plain computation before calling any setters. The `[...new Set([...s, ...completed])]` deduplication pattern was applied as a defense-in-depth guard.

The fallback `useEffect` watching `playerHand` / `aiHand` for set detection was completely removed.

```tsx
// ✅ FIXED — compute OUTSIDE updater, then call setters with plain values
const currentPlayerHand = playerHandRef.current;
const newHand = [...currentPlayerHand, ...matches];
const { remaining, completed } = applySetCheck(newHand); // pure computation

setPlayerHand(remaining);
if (completed.length > 0) {
  setPlayerSets(s => [...new Set([...s, ...completed])]); // deduplicated append
}
setAiHand(prev => prev.filter(card => card.animal !== animal));
```

The `applySetCheck` helper was introduced as a pure, side-effect-free function:

```tsx
const applySetCheck = useCallback((hand: Card[]): { remaining: Card[]; completed: Animal[] } => {
  const completed = findCompletedSets(hand);
  if (completed.length === 0) return { remaining: hand, completed: [] };
  return {
    remaining: hand.filter(card => !completed.includes(card.animal)),
    completed,
  };
}, [findCompletedSets]);
```

### Fix #2 — Remove `isProcessing` from Deps + Use Ref Guard

`isProcessing` was removed from the `useEffect` dependency array entirely. A `useRef` mirror (`isProcessingRef`) was introduced to allow the guard check inside the effect without it being a reactive dependency.

```tsx
// Ref mirrors isProcessing — readable inside the effect without being a dep
const isProcessingRef = useRef(isProcessing);
isProcessingRef.current = isProcessing; // kept fresh on every render

useEffect(() => {
  // ✅ Read via ref — no longer in dep array, cannot self-fire
  if (status !== 'playing' || isProcessingRef.current || playerResponseMode || aiAskingAnimal) return;

  if (turn === 'ai') {
    setIsProcessing(true); // safe — does NOT re-trigger this effect
    const t = setTimeout(() => {
      setIsProcessing(false);
    }, 800);
    return () => clearTimeout(t);
  }
  // isProcessing intentionally EXCLUDED from deps — see comment above
}, [turn, status, playerResponseMode, aiAskingAnimal, turnPulse,
    playerHand.length, aiHand.length, deck.length]);
```

A `turnPulse` counter was also added — incremented whenever the AI goes again after a lucky draw or set completion, ensuring the unified effect re-fires even when no other dependency has changed.

### Fix #3 — Read Latest State via Refs

`useRef` mirrors were added for `aiHand`, `playerHand`, and `deck`. All async handlers read current state through these refs rather than stale closure values.

```tsx
const aiHandRef = useRef(aiHand);
aiHandRef.current = aiHand;  // updated on every render

const playerHandRef = useRef(playerHand);
playerHandRef.current = playerHand;

const deckRef = useRef(deck);
deckRef.current = deck;

// Inside async handler — reads fresh state regardless of when await resolves
const handleAsk = async (animal: Animal) => {
  await new Promise(resolve => setTimeout(resolve, 1000));
  const currentAiHand = aiHandRef.current; // ✅ always current
  const matches = currentAiHand.filter(card => card.animal === animal);
};
```

---

## Key Code Snippets

### Unified Turn Engine

The turn engine is a single `useEffect` that drives all automated game progression — empty hand draws, AI thinking, and turn skipping when cards are exhausted.

```tsx
useEffect(() => {
  if (status !== 'playing' || isProcessingRef.current || playerResponseMode || aiAskingAnimal) return;

  if (turn === 'player') {
    if (playerHand.length === 0 && deck.length > 0) {
      setIsProcessing(true);
      const t = setTimeout(() => {
        const top = deckRef.current[0];
        const rest = deckRef.current.slice(1);
        if (top) { setDeck(rest); setPlayerHand([top]); }
        setIsProcessing(false);
      }, 1000);
      return () => clearTimeout(t);
    }
    return; // player has cards — wait for click
  }

  if (turn === 'ai') {
    if (aiHand.length === 0 && deck.length > 0) {
      setIsProcessing(true);
      const t = setTimeout(() => {
        const top = deckRef.current[0];
        if (top) { setDeck(deckRef.current.slice(1)); setAiHand([top]); }
        setIsProcessing(false);
      }, 1200);
      return () => clearTimeout(t);
    }
    setIsProcessing(true);
    aiThinkTimerRef.current = setTimeout(() => {
      const hand = aiHandRef.current;
      const card = hand[Math.floor(Math.random() * hand.length)];
      setAiAskingAnimal(card.animal);
      setPlayerResponseMode('deciding');
      setIsProcessing(false);
    }, 800);
  }
}, [turn, status, playerResponseMode, aiAskingAnimal, turnPulse,
    playerHand.length, aiHand.length, deck.length]);
```

### Player Ask Flow

```tsx
const handleAsk = async (animal: Animal) => {
  if (turn !== 'player' || isProcessing || status === 'game-over' || playerResponseMode) return;
  if (!playerHand.some(c => c.animal === animal)) return;

  setIsProcessing(true);
  await new Promise(resolve => setTimeout(resolve, 1000));

  const matches = aiHandRef.current.filter(card => card.animal === animal);

  if (matches.length > 0) {
    const newHand = [...playerHandRef.current, ...matches];
    const { remaining, completed } = applySetCheck(newHand);
    setPlayerHand(remaining);
    if (completed.length > 0) setPlayerSets(s => [...new Set([...s, ...completed])]);
    setAiHand(prev => prev.filter(card => card.animal !== animal));
    setIsProcessing(false);
    setTurnPulse(p => p + 1);
  } else {
    const drawn = deckRef.current[0];
    const newHand = [...playerHandRef.current, drawn];
    const { remaining, completed } = applySetCheck(newHand);
    setPlayerHand(remaining);
    if (completed.length > 0) setPlayerSets(s => [...new Set([...s, ...completed])]);
    setDeck(deckRef.current.slice(1));
    if (drawn.animal === animal) {
      setIsProcessing(false);
      setTurnPulse(p => p + 1); // lucky draw — go again
      return;
    }
    setTurn('ai');
    setIsProcessing(false);
  }
};
```

### Game Over Detection

```tsx
useEffect(() => {
  const uniquePlayerSets = new Set(playerSets).size;
  const uniqueAiSets = new Set(aiSets).size;
  const allSetsCollected = uniquePlayerSets + uniqueAiSets >= ANIMALS.length;
  const noCardsLeft = deck.length === 0 && playerHand.length === 0 && aiHand.length === 0;

  if ((allSetsCollected || noCardsLeft) && ANIMALS.length > 0 && status === 'playing') {
    setStatus('game-over');
    const winner =
      uniquePlayerSets > uniqueAiSets ? 'You win!' :
      uniquePlayerSets < uniqueAiSets ? 'AI wins!' : "It's a tie!";
    setMessage(`Game Over! ${winner}`);
  }
}, [playerSets, aiSets, deck.length, playerHand.length, aiHand.length, status]);
```

> Uses `>= ANIMALS.length` (not `===`) to guard against any edge case where a set count could exceed the exact animal count.

---

## Before / After Traces

### Duplicate Sets Bug

```
// Before (broken)
[Turn 4] Player collected Crab set
  → playerSets: ['crab', 'crab']   ← duplicate!
  → Score display: Player 2  (should be 1)

// After (fixed)
[Turn 4] Player collected Crab set
  → applySetCheck([...]) → completed: ['crab']
  → setPlayerSets(s => [...new Set([...s, 'crab'])]) → ['crab']
  → Score display: Player 1  ✓
```

### AI Deadlock Bug

```
// Before (broken)
[Turn 6] Player: "Go Fish!" — drew Octopus
  → setTurn('ai')
  → useEffect fires (turn changed)
  → setIsProcessing(true)
  → useEffect re-fires (isProcessing changed!) ← self-cancels
  → setTimeout never executes
  → UI frozen: "AI is drawing..."  ♾️

// After (fixed)
[Turn 6] Player: "Go Fish!" — drew Octopus
  → setTurn('ai')
  → useEffect fires (turn changed)
  → isProcessingRef.current = false → guard passes
  → setIsProcessing(true)  ← does NOT re-trigger effect
  → setTimeout fires after 800ms
  → AI picks card, asks player  ✓
```

### Console Log Trace (Representative)

```
[INIT] Deck shuffled. Player: 5 cards, AI: 5 cards.
[TURN] player
[PLAYER] Asked for: octopus
[AI] Has 1 octopus(s) — transferring
[SET] applySetCheck → completed: []
[PLAYER HAND] 6 cards
[TURN] player (stayed — match found)
[PLAYER] Asked for: octopus
[AI] Has 0 octopus(s) — Go Fish
[DECK] Drawing: starfish
[SET] applySetCheck → completed: []
[TURN] ai
[AI THINKING] 800ms delay
[AI] Asking player for: crab
[PLAYER] Responding: yes → giving 2 crab cards
[SET] applySetCheck → completed: ['crab']  ← AI scored a set
[TURN] ai (goes again after set)
[AI] Asking player for: turtle
...
```

---

## Steps to Run

### Prerequisites

| Requirement | Version |
| ----------- | ------- |
| Node.js | >= 18.x |
| npm | >= 9.x |

### Setup

```bash
# Navigate to the project directory
cd ocean-go-fish

# Install dependencies
npm install

# Start the development server
npm run dev
# → App available at http://localhost:3000
```

### Available Scripts

| Command | Description |
| ------- | ----------- |
| `npm run dev` | Start Vite dev server with HMR on port 3000 |
| `npm run build` | Build production bundle to `./dist` |
| `npm run preview` | Preview the production build locally |
| `npm run lint` | TypeScript type check (no-emit) |
| `npm run clean` | Remove `./dist` directory |

---

## Verified Behaviors

| Scenario | Status |
| -------- | ------ |
| Player collects set of 4 | ✅ Set scored exactly once |
| AI collects set of 4 | ✅ Set scored exactly once |
| Player asks — AI has cards | ✅ Transfer + set check + player goes again |
| Player asks — AI lacks cards | ✅ Player draws; AI turn if no lucky draw |
| AI asks — player says yes | ✅ Card selection enforced; all-or-nothing |
| AI asks — player lies | ✅ Alert shown; action blocked |
| Deck empty — hand empty | ✅ Turn skipped; game-over triggered |
| Lucky draw rule | ✅ Both player and AI re-ask after drawing asked animal |
| Game over detection | ✅ Fires on set count OR card exhaustion |
| Restart game | ✅ Full state reset, fresh shuffle |

---

## Common Mistakes & Fixes

| ❌ Mistake | ✅ Better Practice |
| --------- | ----------------- |
| Run side effects inside `setState` updaters | Compute values outside updaters; call setters with plain results |
| Include written state in `useEffect` dep array | Read written state via `useRef` mirror; keep deps to read-only values |
| Read state directly inside async handlers | Mirror mutable state to refs; read `.current` after each `await` |
| Use a single fallback `useEffect` for set detection | Use a pure helper (`applySetCheck`) called inline at each mutation site |
| Incremental set append without dedup | Use `[...new Set([...s, ...completed])]` as defense-in-depth |

---

## Learnings

**1. Never run side effects inside React state updater functions.**
Updaters must be pure. Any logic that triggers additional `setState` calls must happen *outside* the updater, computed as a plain value first.

**2. `useEffect` dependency arrays must reflect values the effect *reads*, never values it *writes*.**
Adding `isProcessing` to the deps of an effect that called `setIsProcessing` created a self-cancelling loop. Reading via a ref and writing to state cleanly separates the two surfaces.

**3. Async functions close over stale state.**
In any async handler with `await`, state values captured in the closure are frozen at call time. For values expected to change between awaits (deck, hands), refs are the correct solution.

**4. `turnPulse` as a trigger primitive.**
When a sequence ends and no observable dependency has changed, a simple incrementing counter bumped in the handler reliably re-fires the turn engine — a controlled, explicit signal rather than a workaround.

**5. Deduplication as defense-in-depth.**
`[...new Set([...s, ...completed])]` costs nothing and prevents future regressions from re-introducing duplicate set state.

---

## Glossary

| Term | Meaning |
| ---- | ------- |
| SuT | System under Test |
| Updater | The function callback form of `setState(prev => ...)` |
| Stale closure | A function holding an outdated reference to a variable captured at creation time |
| `useRef` mirror | A ref that shadows a state variable and is updated each render to stay current |
| `turnPulse` | An incrementing counter used to re-trigger a `useEffect` when no other dep changes |
| Concurrent Mode | React's internal scheduler that may run updaters multiple times for purity checks |
| Lucky draw | Drawing the exact animal you asked for, entitling you to another turn |
| `applySetCheck` | A pure function that returns completed sets and the remaining hand after removal |

---

## TL;DR

* Never call `setState` inside another `setState` updater — compute outside, then set
* Remove state variables from `useEffect` deps if the effect writes to them — use refs to read
* In async functions, read from `.current` refs after every `await` to avoid stale closures
* Use an incrementing `turnPulse` counter when you need to re-trigger an effect with no dep change
* Always deduplicate append-to-array state as defense-in-depth

## Reference

This document serves as the proof of work for the Ocean Go Fish submission — covering bug identification, root cause analysis, all fixes applied, and key learnings from the development cycle.
