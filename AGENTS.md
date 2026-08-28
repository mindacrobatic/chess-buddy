# AGENTS.md

## Project overview

Self-contained browser game demos. No build system, no dependencies, no
tests — each demo is a single `.html` file opened directly in a browser.

Current demos:

- `chess/index.html` — chess vs. an in-browser engine
  (~2300 lines, everything inline).

## Tech conventions

- **Single-file demos**: inline `<style>` + `<script>`. No bundlers, no npm,
  no external dependencies. Keep it that way unless the user explicitly asks
  otherwise.
- **Vanilla JS**: plain JavaScript, no frameworks. Engine code uses typed
  arrays (`Int8Array`/`Int16Array`/`Int32Array`) and bit-packed move
  encoding — preserve this style in performance-critical paths.
- **No build tooling or tests**: no test suite, linter, or formatter is
  configured. Don't add one without asking.

## Running a demo

Open the HTML file directly, or serve the repo:

```bash
python3 -m http.server 8000
# then visit http://localhost:8000/chess/
```

## `chess/index.html` architecture

The file has four parts, in order:

1. **`<style>`** (lines ~7–416) — all CSS.
2. **HTML markup** (lines ~419–500) — board container, main side panel
   (card-based: status / game setup with mode + strength / material /
   moves / buttons), a second `.chat-panel` column for the chat card,
   promotion overlay. On narrow screens the chat panel wraps below the
   main panel with a fixed-height log.
3. **Engine source, as inert text** (lines ~481–1543) —
   `<script id="engine-src" type="text/plain">`. This is the engine.
   Because its type is `text/plain`, the browser does not execute it. The
   controller extracts its `textContent`, wraps it in a `Blob`, and creates
   a Web Worker from it. **When editing the engine, you are editing text
   that will run inside a worker — it has no DOM access.**
4. **Controller / UI** (last `<script>`, lines ~1545–end) — executes on the
   main thread. Owns game state, rendering, input, and the worker bridge.

### Two independent chess implementations (important!)

Chess rules are implemented **twice, deliberately**:

- **Controller (main thread)** — an 8x8 array of `{t, c}` objects
  (`t` = `p|n|b|r|q|k`, `c` = `w|b`). Used for move validation, SAN
  generation, and game-end detection. Indexing: `board[row][col]`, row 0 =
  rank 8 (black's side). Key functions: `pseudoMoves`, `legalMoves`
  (filters pseudo-legal via clone + `inCheck`), `makeMove`, `moveToSan`,
  `insufficientMaterial`, `attacked`, `findKing`.
- **Engine (worker)** — a 10x12 mailbox board (`Int8Array(120)`) for speed.
  Used only for search. Indexing: square120 = `21 + row*10 + col`;
  `mb64`/`mb120` convert between 0–63 and mailbox indices.

When you change rules-affecting behavior (e.g. a new move type), consider
whether both implementations need the change — the UI must be able to
display/validate what the engine plays. For pure search/eval tweaks,
engine-only edits are fine.

### Controller ↔ engine protocol

The controller is the source of truth for the position. On every engine
request it sends the **full move list**; the engine replays it from
`START_FEN` (stateless search).

- Controller → worker: `{cmd: "search", id, moves: [{f, t, p}], thinkMs,
  maxDepth, noise}` where `f`/`t` are 0–63 squares and `p` is a promo code
  (`q:5, r:4, b:3, n:2`). `{cmd: "stop"}` aborts.
- Worker → controller: `{cmd: "info", id, depth, score, nodes, time, pv}`
  (progress) then `{cmd: "best", id, move, from, to, promo, score, depth,
  nodes}`. Also `{cmd: "perft"}` for debugging.
- `onEngineMsg` matches `msg.id === reqId`; stale messages are ignored.
  `stopEngine()` terminates the worker; `ensureWorker()` lazily rebuilds
  one from the embedded source.

### Engine internals (inside `engine-src`)

- **Piece codes**: white pieces 1–6 (pawn…king), black pieces 9–14.
  `(p & 8)` = color, `(p & 7)` = type.
- **Move encoding** (packed int): `from (bits 0–5) | to<<6 | promo<<12 |
  flags<<15`. Flags: `FLAG_EP=1`, `FLAG_CASTLE=2`, `FLAG_DOUBLE=4`.
- **Castling rights**: bitmask `15 = WK|WQ|BK|BQ`; `castleMask[square]`
  clears rights when king/rook squares are touched.
- **Board state**: incremental `mgScore`/`egScore`/`phase`/`hash` updated by
  `addPiece`/`removePiece`; make/unmake via undo arrays (`uMove`, `uCap`,
  `uCastle`, …) — no cloning.
- **Zobrist**: `zPiece[15*120]`, `zCastle[16]`, `zEp[120]`, `zSide`, seeded
  by `mulberry32`. `hashStack` + `isRepetition()` for draw detection.
- **TT**: fixed 2^20 entries, depth-preferred, always-replace, mate-score
  normalized by ply.
- **Search**: iterative deepening; negamax with PVS, check extensions,
  null-move pruning, LMR, futility pruning, quiescence with SEE-ish delta
  cutoff; ordering = TT move, MVV-LVA captures, killers, history. Root uses
  aspiration windows (`bestScore ± 45`, widen by 300 on fail). Time
  control throws `ABORT` at `deadline`; stops after a depth once half the
  allotted time is used.
- **Eval**: tapered over `phase` (24 max): material+PST (baked into
  `pstMgTab`/`pstEgTab`), pawn structure (doubled/isolated/passed via
  per-file min/max tracking), mobility per piece, rook on open/half-open
  file, bishop pair, king shelter. Returns relative-to-side score.
- **`perft`** is exposed over the worker interface — use it to verify move
  generation after editing `genMoves`/`make`/`unmake`.

### Controller internals

- `state` is authoritative; `history` holds `{san, snap: {state, move}}`
  snapshots for undo. Undo pops 2 ply when an engine is playing and it's
  not the engine's turn (so both sides undo), else 1 ply.
- `render()` **rebuilds the whole board DOM** on every change — don't add
  incremental DOM updates; keep the full-rebuild pattern.
- Game end checked in `evaluateGameEnd`: mate/stalemate via controller
  `legalMoves`, plus insufficient material, 50-move (`halfmove >= 100`),
  threefold repetition via string `posKeys` (independent from the engine's
  hash repetition).
- SAN is generated by `moveToSan(before, m, after, legalList)` — needs the
  pre-move legal list for disambiguation.
- Engine strength presets in `LEVELS`: Newbie (200 ms, depth 2, ±150
  noise), Okayish (800 ms, depth 5, ±20 noise), Boss (2000 ms, depth 99,
  no noise). Noise = random uniform perturbation of root scores to weaken
  play.
- Persistence via `localStorage`: keys `chessStrength` and
  `chessBoardSize`.
- **LLM chat** (`/* --- LLM opponent chat (OpenRouter) --- */` section): a
  chat card where the user pastes an OpenRouter API key to talk with the
  opponent. Key is session-only (`orKey` variable, never persisted). Calls
  `POST https://openrouter.ai/api/v1/chat/completions` with model
  `z-ai/glm-5.3-flash` directly from the controller via `fetch` — no
  worker involvement. `glm-5.3-flash` has mandatory reasoning on
  OpenRouter (cannot be disabled — `effort: "none"` is rejected with
  HTTP 400), so requests use `reasoning: { effort: "minimal",
  exclude: true }` and `max_tokens: 1024` so thinking tokens can't
  starve the visible reply; an empty reply is retried once at 2048,
  then dropped silently. Error bubbles include the
  provider's message from the JSON error body. `gameContext()` feeds the LLM live
  state (SAN move
  list, turn, check, material, game-over). The opponent only speaks on
  significant events (`describeMoveEvents`): captures (piece named),
  promotions, checks — by either side — plus eval swings of
  ≥ `CHAT_EVAL_SWING` centipawns (150) computed from the engine's `best`
  score vs. the previous one (`lastWhiteEval`, white perspective, mate
  scores clamped to ±5000), and game end (`chatAfterGameEnd`). Quiet
  moves get no commentary; game-event triggers (`chatEvent`) no-op
  unless a key is set and no request is in flight.
  Requests are serialized through a promise chain (`chatQueue`); user
  messages (`userSay`) abort any in-flight reply via an `AbortController`
  (`chatAbort`) before queueing, so user input always interrupts banter
  and is never lost. Aborted requests fail silently (no error bubble).
  `chatReset()` clears the log on New Game. Chat bubbles are built with `textContent` (never
  `innerHTML`) and appended incrementally — outside the `render()`
  full-rebuild path.
- Unicode piece glyphs (`GLYPH`) are styled with CSS text-stroke
  (`.glyph-w` / `.glyph-b`) rather than images.

## Verifying changes

There is no automated test suite. For controller changes, load the page and
play. For engine changes, sanity-check with the worker `perft` command
against known perft counts (startpos: 20, 400, 8902, 197281 at depth 1–4)
or play against it in the browser. Check the browser console for worker
errors — syntax errors inside `engine-src` only surface when the worker
blob is created.

## Guidelines for agents

- Prefer editing existing files; don't create new files unless required.
- When adding a new demo, create a subdirectory named after it with an
  `index.html` inside, and update the list above.
- Keep demos dependency-free and runnable by opening the file directly.
- Preserve the separation of concerns: UI-render in the controller,
  search in the worker. Don't move DOM calls into `engine-src` or engine
  code into the controller.
