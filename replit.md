# The Doubleganger

A 2D top-down multiplayer browser game with three modes — Friend or Foe (social deduction), Infection (survival), and Shadow (singleplayer vs AI Hunter) — all set in a shared "Lab Complex" map.

## Run & Operate

- `pnpm --filter @workspace/api-server run dev` — run the API server (port 5000), hosts the game WebSocket at `/api/ws`
- `pnpm --filter @workspace/doubleganger run dev` — run the game client
- `pnpm run typecheck` — full typecheck across all packages
- `pnpm run build` — typecheck + build all packages
- `pnpm --filter @workspace/api-spec run codegen` — regenerate API hooks and Zod schemas from the OpenAPI spec (not used for game logic, only for any future REST endpoints)
- `pnpm --filter @workspace/db run push` — push DB schema changes (dev only)
- Required env: `DATABASE_URL` — Postgres connection string

## Stack

- pnpm workspaces, Node.js 24, TypeScript 5.9
- API: Express 5 + `ws` (raw WebSocketServer mounted on the same HTTP server at `/api/ws`)
- DB: PostgreSQL + Drizzle ORM
- Validation: Zod (`zod/v4`), `drizzle-zod`
- API codegen: Orval (from OpenAPI spec) — not used for real-time game messages
- Build: esbuild (CJS bundle)

## Where things live

- `lib/game-shared` — shared constants, types, the Lab Complex map definition, and geometry helpers (line-of-sight, zone lookup, distance) used by both server and client
- `artifacts/api-server/src/game/` — server-side game logic
  - `rooms.ts` — room/lobby lifecycle (create/join/leave, room codes)
  - `engine.ts` — fixed-tick game loop, input handling, broadcasting state snapshots
  - `socket.ts` — WebSocket message routing, wired into `index.ts` via `http.createServer(app)`
  - `modes/{friendOrFoe,infection,shadow}.ts` — per-mode rules (roles, win conditions, AI hunter behavior)
- `artifacts/doubleganger/src/`
  - `net/socket.ts` — client WebSocket wrapper
  - `game/{input,render}.ts` — input capture and canvas rendering
  - `game/render.ts` — preloads character sprite PNGs (`assets/characters/`) and draws them rotated to `facing`, falling back to a colored circle if an image hasn't loaded yet
  - `pages/{Home,Lobby,Game,Doubleganger,LoadingScreen}.tsx` — screens, wired into `App.tsx`

## Architecture decisions

- Game state/messages are pure WebSocket, not REST/OpenAPI — real-time gameplay doesn't fit the request/response codegen model used elsewhere in the repo.
- Rendering uses top-down character sprite PNGs (rotated to facing direction) + name tags, with a colored-circle fallback if a sprite hasn't loaded yet.
- Shadow mode's AI Hunter has a `HUNTER_GRACE_MS` (6s) detection-immunity window after match start — without it the hunter's patrol path passes the stationary player spawn almost immediately, ending the match before the player can react.
- A short loading screen (`LoadingScreen.tsx`) shows on first load while character sprite images preload, before the menu (`Home.tsx`) is shown.
- Player name and skin choice are persisted to `localStorage` (`Home.tsx`) so they pre-fill on the next visit.

## Product

- **Friend or Foe**: 4-10 player social deduction in the Lab Complex; lobby with room codes, host-started match.
- **Infection**: all-vs-infected survival; infection spreads via proximity touch, infected players move faster.
- **Shadow**: singleplayer vs an AI Hunter that patrols, chases on sight/sound (line-of-sight + FOV + hearing radius), with a brief post-start grace period.
- Shared "Lab Complex" map: main hall, corridors, shadow/light zones (affect Hunter view distance), vents.

## User preferences

- User wants all three game modes built simultaneously as basic/functional versions, not one at a time.

## Gotchas

- When debugging the WebSocket server outside the browser (e.g. via code execution), the sandbox lacks global `WebSocket`/`process.env` — connect using the dev domain directly (`wss://<dev-domain>/api/ws`) and the `WebSocket` global is available in that sandbox's `code_execution` environment.
- In multi-socket test scripts, attach all `message` listeners *before* triggering a broadcast (e.g. `start-game`) — attaching them sequentially inside an `await` loop races against near-simultaneous server broadcasts and causes false "timeout" failures.

## Pointers

- See the `pnpm-workspace` skill for workspace structure, TypeScript setup, and package details
