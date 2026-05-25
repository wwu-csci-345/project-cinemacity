# CSCI 345 — Object-Oriented Design
## Programming Assignment: CinemaCity

---

## 1. Project Overview

**CinemaCity** is a browser-playable, multiplayer board game set in a fictional movie studio. Players move between studio locations, take acting roles in film productions, rehearse and act to remove shot counters from scenes, earn credits and reputation, and upgrade their rank. The player with the highest score when the last scene wraps wins.

This assignment is your capstone exercise in **object-oriented design**: you will implement a working game engine — from the lowest-level domain objects up through the command-pattern action layer — while the factory, event, and UI layers are handed to you fully working.

### Learning Objectives

After completing this assignment you will be able to:

1. **Apply the Single Responsibility Principle** — each class in the domain layer has a clearly defined, narrow responsibility.
2. **Implement the Command Pattern** — each player action is encapsulated in its own class with `validate()` and `execute()` methods.
3. **Implement the Observer Pattern** — domain events are emitted by the `GameEventEmitter`; the UI reacts without being coupled to game logic.
4. **Implement the Factory Pattern** — factories construct complex object graphs from raw data, keeping domain classes free of parsing concerns.
5. **Practice test-driven development** — your task is to write all 58 test bodies **first**, then make them pass by implementing the domain classes.
6. **Reason about object graph ownership** — understand how the `Game` facade coordinates `Board`, `Players`, `SceneDeck`, and `TurnManager` without any single class becoming a "god object".

---

## 2. Game Rules Summary

### 2.1 Setup
- 2–4 players. Each player starts at the **Trailer Park** with **rank 1** and **2 credits**.
- One scene card is dealt face-up to each of the 6 filming locations.
- The remaining scene cards form the draw pile (deck).

### 2.2 Turn Structure
Each turn a player may perform **any combination** of the following actions, subject to constraints:
- **Move** — at most once. Move to an adjacent location. Cannot move while on a role.
- **Take Role** — at most once. Assign yourself to an available role at your current location. Cannot take a role while already on one.
- **Act OR Rehearse** — at most one of these per turn. Must be on a role. Must have an active scene at the location.
- **Upgrade** — at most once per turn, at the Casting Agency only. Cannot upgrade while on a role.
- **End Turn** — always legal. Passes control to the next player.

### 2.3 Acting Mechanic
When a player acts:
1. Roll one six-sided die.
2. Add your current **rehearsal tokens** as a flat bonus.
3. Compare total to the scene's **budget**:
   - **Success** (total ≥ budget): Remove one shot counter from the scene. Rewards depend on role type:
     - **On-card role**: +1 reputation. (Wrap bonus delivered later, not now.)
     - **Off-card role**: +2 credits, +1 reputation.
   - **Failure** (total < budget): No reward on on-card roles. Off-card roles earn +1 credit as a consolation ("showed up for the day").

### 2.4 Rehearsal
Gaining a rehearsal token adds +1 to your next (and all future) die rolls while you hold the same role. Tokens reset to 0 when you clear a role (on scene wrap or voluntarily).

### 2.5 Scene Wrap
A scene wraps when its last shot counter is removed. When a scene wraps:
1. **Wrap bonuses** are distributed to players holding on-card roles: `pay` credits + 2 reputation per on-card role player.
2. **All player roles** at that location are cleared (role ID reset, rehearsal tokens reset to 0).
3. **All role slots** on the scene card and at the location are vacated (so they can be reused by a future scene).
4. The scene card is removed from the location.
5. The **completed scenes** counter is incremented.
6. If the deck is not empty, a new scene card is dealt to that location.
7. The **game-over condition** is checked (see §2.7).

### 2.6 Rank Upgrades
- Upgrades happen at the **Casting Agency** (the upgrade location).
- You may only upgrade **one rank at a time** (rank 1 → 2, not rank 1 → 3).
- Maximum rank is **6**.
- You pay with **either** credits or reputation (not both) according to the cost table:

| To Rank | Credit Cost | Reputation Cost |
|---------|-------------|-----------------|
| 2       | 4           | 2               |
| 3       | 10          | 4               |
| 4       | 18          | 6               |
| 5       | 28          | 8               |
| 6       | 40          | 12              |

### 2.7 Game-Over Condition
The game ends when **the deck is empty AND no filming location has an active scene**. (The deck becoming empty alone does not end the game; players must finish all remaining scenes first.)

### 2.8 Scoring
Final score = **reputation × 2 + credits + rank**

The player with the highest final score wins.

---

## 3. Architecture: Class-by-Class Responsibility

Study this section carefully before writing any code. Understanding what each class is responsible for — and what it is *not* responsible for — is the most important design skill this assignment develops.

```
src/
  domain/
    types.ts             — Shared type definitions (no logic)
    Role.ts              — A single acting role; knows who holds it
    SceneCard.ts         — A scene card; tracks shot counters and on-card roles
    SceneDeck.ts         — The draw pile; provides cards on demand
    Location.ts          — A named board space; holds a scene and off-card roles
    Board.ts             — The board; indexes and queries locations
    Player.ts            — A player; owns all player state
    TurnManager.ts       — Tracks whose turn it is and per-turn action flags
    Game.ts              — Facade; coordinates all domain objects
    events/
      GameEvents.ts      — (given) Discriminated union of all event types
      GameEventEmitter.ts — (given) Typed publish/subscribe event bus
    actions/
      Action.ts          — (given) Command interface: validate() + execute()
      MoveAction.ts      — Command: move to adjacent location
      TakeRoleAction.ts  — Command: assign player to a role
      ActAction.ts       — Command: act in a scene (die roll mechanic)
      RehearseAction.ts  — Command: gain one rehearsal token
      UpgradeAction.ts   — Command: upgrade rank at Casting Agency
      EndTurnAction.ts   — Command: end the current player's turn
    factories/
      BoardFactory.ts    — (given) Builds Board + Locations + Roles from JSON
      SceneDeckFactory.ts — (given) Builds SceneDeck + SceneCards from JSON
      GameFactory.ts     — (given) Assembles the complete Game object graph
```

### Role.ts
Owns a single acting role. Tracks who currently holds it (`takenByPlayerId`). Knows whether the role is `isOnCard` (a scene-card role) or an off-card location role. Does **not** know game rules or rewards.

Key methods:
- `isAvailable()` → `boolean` — true when no player holds this role.
- `assign(playerId)` — records the player who took this role.
- `vacate()` — clears the player assignment (called on scene wrap).

### SceneCard.ts
Tracks remaining shot counters and on-card roles. Does **not** know about players, rewards, or the board.

Key methods:
- `removeShot()` → `boolean` — decrements the shot counter; returns `true` when the counter reaches zero (i.e., the scene is now wrapped).
- `getAvailableRoles()` → `Role[]` — returns on-card roles that are not yet taken.

### SceneDeck.ts
A simple draw pile. Holds scene cards in order and pops them off one at a time. Does **not** shuffle (that is the factory's job).

Key methods:
- `draw()` → `SceneCard | null` — removes and returns the top card; returns `null` if the deck is empty.

### Location.ts
Represents a named place on the board. Holds its current scene card (if any) and its permanent off-card roles.

Key methods:
- `setScene(card)` — place a scene card here.
- `clearScene()` — remove the current scene.
- `hasScene()` → `boolean` — true when a scene is active.
- `getAvailableRoles()` → `Role[]` — **critical**: returns `[]` when no scene is active. When a scene is active, returns both available on-card roles (from the scene) and available off-card roles (from the location).
- `isNeighborOf(locationId)` → `boolean` — true when the given ID appears in this location's neighbor list.

### Board.ts
An indexed collection of `Location` objects. Answers adjacency and filtering queries. Does **not** know about players, turns, or rules.

Key methods:
- `getLocation(id)` → `Location` — returns the location or throws if the ID is unknown.
- `getNeighbors(locationId)` → `Location[]` — returns the `Location` objects adjacent to the given location.
- `isNeighbor(fromId, toId)` → `boolean` — true when `toId` is in `fromId`'s neighbor list.
- `getSceneLocations()` → `Location[]` — returns all locations that are not upgrade locations.
- `getUpgradeLocation()` → `Location` — returns the single upgrade location (throws if none).

### Player.ts
Owns all player state: identity, position, rank, credits, reputation, current role, and rehearsal tokens. Provides mutating methods that enforce simple preconditions (e.g., `spendCredits` throws if insufficient). Does **not** know about the board, other players, or game rules.

Key methods:
- `moveTo(locationId)` — update the player's location.
- `takeRole(roleId, isOnCard)` — record the role and reset rehearsal tokens.
- `clearRole()` — reset role ID, `isOnCard` flag, and rehearsal tokens.
- `spendCredits(amount)` — deduct credits; **throw** if insufficient.
- `spendReputation(amount)` — deduct reputation; **throw** if insufficient.
- `upgradeRank(toRank)` — set the player's rank.
- `addRehearsalToken()` — increment the token count.
- `calculateScore()` → `number` — returns `reputation × 2 + credits + rank`.

### TurnManager.ts
Tracks whose turn it is (round-robin over the player array) and per-turn action flags. Flags are reset when `advanceTurn()` is called.

Key methods:
- `recordMove()`, `recordTakeRole()`, `recordAct()`, `recordRehearse()`, `recordUpgrade()` — set the corresponding boolean flag to `true`.
- `advanceTurn()` — advance to the next player (wrap around with modulo); increment `turnNumber` when the index wraps back to 0; reset all five per-turn flags.

### Action classes (MoveAction, TakeRoleAction, ActAction, RehearseAction, UpgradeAction, EndTurnAction)
Each implements the `Action` interface with two methods:
- `validate()` → `ActionResult` — checks whether the action is legal given the current state. **Does not mutate state.**
- `execute()` → `ActionResult` — calls `validate()` first; returns the validation result if the action is illegal; otherwise, mutates state, emits events, and returns a descriptive `ActionResult`.

**MoveAction**:
- Validate: player must not be on a role; player must not have already moved this turn; target must be a neighbor.
- Execute: call `player.moveTo()`, `turnManager.recordMove()`, emit `playerMoved` and `stateChanged`.

**TakeRoleAction**:
- Validate: player must not already have a role; player must not have taken a role this turn; the role must be available at the player's location; player's rank must meet the role's required rank.
- Execute: `role.assign(player.id)`, `player.takeRole(...)`, `turnManager.recordTakeRole()`, emit `roleTaken` and `stateChanged`.

**ActAction**:
- Validate: player must have a role; player must not have already acted or rehearsed this turn; the location must have an active scene.
- Execute: roll the die, add rehearsal tokens. On success, remove a shot and give rewards (on-card: +1 rep; off-card: +2 cr, +1 rep). On failure, give consolation reward (off-card only: +1 cr). Record act. Emit `actPerformed`. If the scene wrapped, call `onSceneWrap` callback and emit `sceneWrapped`. Emit `stateChanged`.

**RehearseAction**:
- Validate: player must have a role; player must not have already acted or rehearsed this turn; location must have an active scene.
- Execute: `player.addRehearsalToken()`, `turnManager.recordRehearse()`, emit `rehearsed` and `stateChanged`.

**UpgradeAction**:
- Validate: player must not be on a role; player must not have already upgraded this turn; player must be at the upgrade location; target rank must be `current + 1`; target rank must be ≤ 6; player must have sufficient credits or reputation.
- Execute: deduct currency, call `player.upgradeRank()`, `turnManager.recordUpgrade()`, emit `rankUpgraded` and `stateChanged`.

**EndTurnAction**:
- Validate: always returns `{ success: true }`.
- Execute: `turnManager.advanceTurn()`, emit `turnEnded` and `stateChanged`.

### Game.ts
The **Facade** for the entire domain. Holds references to `Board`, `Players`, `SceneDeck`, `TurnManager`, and `GameEventEmitter`. Provides simple public methods (`move()`, `takeRole()`, `act()`, `rehearse()`, `upgrade()`, `endTurn()`) that construct command objects and call `execute()`. Also provides `can*()` validation helpers for the UI.

Key private method: `_handleSceneWrap(scene, location)` — implements the scene-wrap sequence (see §2.5). This is called as a callback from `ActAction` to avoid a circular import. It returns the list of rewards so `ActAction` can emit the `sceneWrapped` event.

---

## 4. What Is Given vs. What You Must Implement

### Given (fully working — do not modify)

| File | Why given |
|------|-----------|
| `package.json`, `tsconfig.json`, `vite.config.ts`, `vitest.config.ts` | Build and test tooling |
| `index.html`, `src/style.css` | HTML shell and all styles |
| `src/main.ts` | App entry point; wires startup screen and factory |
| `src/ui/GameRenderer.ts` | Full DOM rendering (Observer view) |
| `src/ui/ActionHandler.ts` | DOM event controller |
| `src/data/locations.json`, `src/data/scenecards.json` | Game data |
| `src/domain/types.ts` | All shared type definitions |
| `src/domain/events/GameEvents.ts` | Typed event union |
| `src/domain/events/GameEventEmitter.ts` | Event bus (Observer pattern) |
| `src/domain/actions/Action.ts` | Command interface |
| `src/domain/factories/BoardFactory.ts` | Factory: JSON → Board |
| `src/domain/factories/SceneDeckFactory.ts` | Factory: JSON → SceneDeck |
| `src/domain/factories/GameFactory.ts` | Factory: assembles Game |
| `src/tests/helpers.ts` | Test helper functions |
| `docs/ai-use-log-example.md` | Model AI use log |

### You Must Implement

| File | Stubbed methods |
|------|-----------------|
| `src/domain/Role.ts` | `assign()`, `vacate()`, `isAvailable()` |
| `src/domain/SceneCard.ts` | `removeShot()` |
| `src/domain/SceneDeck.ts` | `draw()` |
| `src/domain/Location.ts` | `setScene()`, `clearScene()`, `hasScene()`, `getAvailableRoles()`, `isNeighborOf()` |
| `src/domain/Board.ts` | `getLocation()`, `getNeighbors()`, `isNeighbor()`, `getSceneLocations()`, `getUpgradeLocation()` |
| `src/domain/Player.ts` | `moveTo()`, `takeRole()`, `clearRole()`, `spendCredits()`, `spendReputation()`, `upgradeRank()`, `addRehearsalToken()`, `calculateScore()` |
| `src/domain/TurnManager.ts` | `recordMove()`, `recordTakeRole()`, `recordAct()`, `recordRehearse()`, `recordUpgrade()`, `advanceTurn()` |
| `src/domain/actions/MoveAction.ts` | `validate()`, `execute()` |
| `src/domain/actions/TakeRoleAction.ts` | `validate()`, `execute()` |
| `src/domain/actions/ActAction.ts` | `validate()`, `execute()` |
| `src/domain/actions/RehearseAction.ts` | `validate()`, `execute()` |
| `src/domain/actions/UpgradeAction.ts` | `validate()`, `execute()` |
| `src/domain/actions/EndTurnAction.ts` | `validate()`, `execute()` |
| `src/domain/Game.ts` | All method bodies, including `_handleSceneWrap()` |
| `src/tests/*.test.ts` | All 58 individual test bodies |

---

## 5. Method Requirements

This section specifies what each stubbed method must do, with enough detail to write tests first.

### 5.1 Role

```typescript
isAvailable(): boolean
```
Returns `true` iff `_takenByPlayerId` is `null`.

```typescript
assign(playerId: string): void
```
Sets `_takenByPlayerId` to `playerId`.

```typescript
vacate(): void
```
Sets `_takenByPlayerId` to `null`.

---

### 5.2 SceneCard

```typescript
removeShot(): boolean
```
- Decrements `_remainingShots` by 1 (do not go below 0).
- If `_remainingShots` reaches 0, set `_isWrapped = true`.
- Returns `_isWrapped` (i.e., `true` if the scene just wrapped).

---

### 5.3 SceneDeck

```typescript
draw(): SceneCard | null
```
Removes and returns the first element from `_cards`. Returns `null` if the array is empty. Do **not** mutate the original array passed to the constructor (make a shallow copy in the constructor).

---

### 5.4 Location

```typescript
setScene(card: SceneCard): void
```
Sets `_currentScene` to `card`.

```typescript
clearScene(): void
```
Sets `_currentScene` to `null`.

```typescript
hasScene(): boolean
```
Returns `true` iff `_currentScene` is not `null`.

```typescript
getAvailableRoles(): Role[]
```
**If no scene is active, return `[]`.** Otherwise, return the concatenation of:
- `_currentScene.getAvailableRoles()` (on-card roles not yet taken)
- `offCardRoles.filter(r => r.isAvailable())` (off-card roles not yet taken)

```typescript
isNeighborOf(locationId: string): boolean
```
Returns `true` iff `locationId` appears in `neighborIds`.

---

### 5.5 Board

```typescript
getLocation(id: string): Location
```
Looks up `id` in the internal `Map<string, Location>`. If not found, **throw** `new Error(`Unknown location ID: "${id}"`)`.

```typescript
getNeighbors(locationId: string): Location[]
```
Returns the `Location` objects whose IDs appear in the neighbor list of the location identified by `locationId`.

```typescript
isNeighbor(fromId: string, toId: string): boolean
```
Returns `true` iff `toId` appears in `fromId`'s neighbor list. (Delegates to `Location.isNeighborOf`.)

```typescript
getSceneLocations(): Location[]
```
Returns all locations where `isUpgradeLocation === false`.

```typescript
getUpgradeLocation(): Location
```
Returns the first location where `isUpgradeLocation === true`. Throws if none exists.

---

### 5.6 Player

```typescript
moveTo(locationId: string): void
```
Sets `_locationId` to `locationId`.

```typescript
takeRole(roleId: string, isOnCard: boolean): void
```
Sets `_currentRoleId`, `_currentRoleIsOnCard`, and resets `_rehearsalTokens` to 0.

```typescript
clearRole(): void
```
Sets `_currentRoleId` to `null`, `_currentRoleIsOnCard` to `false`, and `_rehearsalTokens` to 0.

```typescript
spendCredits(amount: number): void
```
If `amount > _credits`, throw `new Error('Not enough credits: ...')`. Otherwise deduct.

```typescript
spendReputation(amount: number): void
```
If `amount > _reputation`, throw `new Error('Not enough reputation: ...')`. Otherwise deduct.

```typescript
upgradeRank(toRank: number): void
```
Sets `_rank` to `toRank`.

```typescript
addRehearsalToken(): void
```
Increments `_rehearsalTokens` by 1.

```typescript
calculateScore(): number
```
Returns `_reputation * 2 + _credits + _rank`.

---

### 5.7 TurnManager

```typescript
recordMove(): void    // sets _hasMoved = true
recordTakeRole(): void // sets _hasTakenRole = true
recordAct(): void     // sets _hasActed = true
recordRehearse(): void // sets _hasRehearsed = true
recordUpgrade(): void  // sets _hasUpgraded = true
```

```typescript
advanceTurn(): void
```
- Advance `_currentIndex` to `(_currentIndex + 1) % _players.length`.
- If the new `_currentIndex === 0`, increment `_turnNumber` (a full round completed).
- Reset all five per-turn flags to `false`.

---

### 5.8 Action classes

Each action class must implement `validate()` and `execute()` as described in §3 above. All `execute()` methods must call `validate()` first and return the validation result unchanged if invalid.

Key invariant: **`validate()` must not mutate state**. It may read any field but must not call any setter, `record*()` method, or `emit()`.

---

### 5.9 Game.ts

`Game` creates one `TurnManager` (in the constructor) and wires together all domain objects. Each public method (`move`, `takeRole`, `act`, `rehearse`, `upgrade`, `endTurn`) constructs the corresponding action object and calls `execute()`.

`_handleSceneWrap(scene, location)` must:
1. Distribute wrap bonuses to on-card role players (`pay` credits + 2 reputation each).
2. Clear role state for all players located at `location.id` who have a role.
3. Vacate all role slots on the scene card and on the location's off-card roles.
4. Call `location.clearScene()`.
5. Increment `_completedScenes`.
6. If the deck is not empty, draw a new card and call `location.setScene(card)`.
7. Call `_checkGameOver()` to test whether the game should end.
8. Return the rewards array (so `ActAction` can emit the `sceneWrapped` event).

---

## 6. Tests

All test files have their `describe` blocks and `it()` titles in place. Your job is to **fill in the body** of each test so that it passes once your implementation is correct.

### Recommended order of implementation (bottom-up)

1. `Role` → `SceneCard` → `SceneDeck` (lowest-level, no dependencies)
2. `Location` (depends on Role, SceneCard)
3. `Board` (depends on Location)
4. `Player` (independent of board)
5. `TurnManager` (depends on Player)
6. Action classes (depend on all of the above)
7. `Game` (depends on everything)

Start by writing tests for each class **before** implementing it. Run `npm test` to see failures. Implement until all tests pass.

### Test files and test count by area

| File | # Tests | Area |
|------|---------|------|
| `movement.test.ts` | 6 | Movement rules |
| `roles.test.ts` | 8 | Role-taking rules |
| `acting.test.ts` | 9 | Act/rehearse mechanic |
| `scenes.test.ts` | 7 | Scene completion and wrap |
| `upgrades.test.ts` | 10 | Upgrade rules and cost table |
| `turns.test.ts` | 8 | Turn order and flag resets |
| `game.test.ts` | 13 | Integration: game-over, scoring, events |
| **Total** | **61** | |

### Using the helpers

`src/tests/helpers.ts` exports two factory functions for tests:

```typescript
// Minimal 2-player game with a hand-crafted 3-location board. Fast and deterministic.
makeMinimalGame(overrides?): { game, locA, locB, upgrade, alice, bob }

// Full 2-player game using the real JSON data files. Used for integration tests.
makeGame(playerNames?, rollDie?): Game
```

You may add additional helpers to `helpers.ts` if needed, but do not remove existing ones.

---

## 7. Grading Criteria

| Area | Points | Requirement |
|------|--------|-------------|
| Movement | 10 | Move/reject rules pass; location updated; can move again next turn |
| Roles | 15 | Take/reject rules pass; rank check; already-taken check; scene-required check |
| Acting — success/failure rewards | 15 | Correct credits/reputation for on-card vs off-card; die roll + tokens applied |
| Rehearsal | 5 | Token accumulates; mutual exclusion with act; resets on role clear |
| Scene wrap | 15 | Wrap bonuses; role clear; role vacate; scene removed; new card dealt; counter incremented |
| Upgrades | 10 | At agency only; one rank at a time; cost table correct; credits/reputation choice |
| Turn management | 5 | Round-robin order; flag resets; turn number increments |
| Game-over | 10 | Correct condition; scoring formula; winner determined; `isOver` blocks further actions |
| TypeScript (zero errors) | 5 | `npm run typecheck` passes with no errors |
| AI use log | 10 | Log documents ≥ 3 interactions; includes what was accepted/rejected and why |
| **Total** | **100** | |

---

## 8. Running the Project

### Install dependencies (first time only)
```bash
cd deadwood-template   # or wherever you cloned/extracted the project
npm install
```

### Start the dev server
```bash
npm run dev
```
Opens a Vite dev server at `http://localhost:5173`. The game UI is live-reloaded when you save files. You can play the game in the browser to verify your implementation interactively.

> **Note:** The UI will load but actions will throw `"Not implemented"` errors until you implement the corresponding domain classes. Implement bottom-up (Role → SceneCard → … → Game) to see the UI progressively come to life.

### Run tests
```bash
npm test              # single run, shows pass/fail counts
npm run test:watch    # re-runs on file save (useful during development)
```

### Type-check
```bash
npm run typecheck     # reports TypeScript type errors (no compilation output)
```

All three commands must succeed (zero errors, all 58 tests passing) for full credit.

---

## 9. AI Use Policy

You **may** use AI coding assistants (GitHub Copilot, Claude, ChatGPT, etc.) on this assignment, subject to the following conditions:

1. **You must maintain an AI use log** in `docs/ai-use-log.md`. For each substantive AI interaction, record:
   - The date and tool used.
   - The exact prompt (or a faithful paraphrase).
   - A summary of what the AI produced.
   - What you accepted, what you rejected, and **why**.
   - Any modifications you made to the AI-generated code.

2. **Minimum log entries:** You must document at least **3 distinct AI interactions**. Interactions for trivial completions ("add a semicolon") do not count; document design-level or logic-level interactions.

3. **See `docs/ai-use-log-example.md`** for a model of what a good log entry looks like.

4. **You are responsible for understanding every line you submit.** If you cannot explain a piece of code in a code review, it will be treated as academic dishonesty regardless of its source.

5. **Do not ask AI to write your test bodies.** The point of writing tests is to deepen your understanding of the specification. Tests that are AI-generated without student-authored intent will receive no credit.

---

## 10. Submission Checklist

Before submitting, verify:

- [ ] `npm run typecheck` reports zero errors.
- [ ] `npm test` shows all 61 tests passing.
- [ ] `npm run dev` launches the game and all six actions work end-to-end.
- [ ] `docs/ai-use-log.md` has at least 3 substantive entries.
- [ ] You have not modified any files in the **Given** list (§4).
- [ ] You have not modified `src/tests/helpers.ts` in a breaking way.
- [ ] The `dist/` and `node_modules/` directories are not in your submission.
