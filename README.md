# Cipher — a Next Round game module

Cipher is a turn-based codebreaker deduction race for the
[Next Round](https://github.com/lewismatt/next-round) platform: a secret code
is generated for the match, and players race to crack it with private guesses
and Exact/Misplaced feedback. It is a standalone game module — this repository
is the whole game — designed against the platform's game SDK
(`next-round-game/0.1`).

Designed by Larry and Jimothy Von Bottington the 2nd (design v0.1, jointly
endorsed 2026-10-24).

## Rules

- A secret code is generated deterministically at match creation from
  server-side entropy. It lives only in server state and is never sent to a
  client before the match ends.
- Presets:
  - **Cadet** — code length 4, 6 symbols, no repeats, 12 guesses
  - **Standard** — code length 4, 6 symbols, repeats allowed, 10 guesses
  - **Expert** — code length 5, 8 symbols, repeats allowed, 12 guesses
- One guess per turn in strict rotation. Feedback per guess: **Exact** (right
  symbol, right position) and **Misplaced** (right symbol, wrong position),
  computed two-pass so no symbol is double-counted.
- Guesses are **private** until the match ends; opponents see only your
  guesses-used count. At the end, the code and every player's board are
  revealed side by side.
- The first exact crack wins instantly. If everyone exhausts the guess cap,
  players are ranked by their best single guess (most Exact, then most
  Misplaced); an exact tie is a draw. In solo, exhausting the cap records as
  uncracked.
- In no-repeats presets, guesses must also be repeat-free (design amendment
  A1, 2026-10-24). Invalid guesses are rejected without consuming the turn.
  An optional turn
  timer (30 seconds to 7 days, or none) skips a missed turn; membership loss
  pauses the match with the deadline preserved.

## Layout

- `src/manifest.ts` — module manifest (modes, capabilities, JSON schemas)
- `src/server.ts` — pure rules reducer (`GameServer`): code generation,
  feedback, turn rotation, win/draw resolution, pause/resume
- `src/client.ts` — browser client (vanilla DOM over the platform's
  `GameBridge`); it only ever receives the projected view, never the code
- `src/server.test.ts` — rules tests; every transition is schema-validated

## Development

The platform packages (`@next-round/contracts`, `@next-round/game-sdk`,
`@next-round/game-client`) live in the core repository and are not published
to a registry. Development links them from a pinned checkout:

```sh
pnpm setup   # clones lewismatt/next-round into .core/ at a pinned ref,
             # installs, and builds the platform packages
pnpm build
pnpm typecheck
pnpm test
```

`CORE_REF` (see `scripts/fetch-core.sh`) pins the platform commit this module
is developed and verified against. CI does the same.

## Bundling into a Next Round deployment

Per the platform's module model, an operator adds a reviewed package to the
server build registry and redeploys API/web/worker together. The mechanism
below is the one verified end-to-end against the core platform (draft PR
`lewismatt/next-round#46`, CI green on all jobs, 2026-10-24):

1. **Fetch this repo into the core checkout as a workspace package.** The
   core repo's `scripts/fetch-cipher.sh` clones this repository into
   `games/cipher/` (gitignored) at a pinned commit; the `games/*` workspace
   glob picks it up and `apps/api` / `apps/worker` depend on
   `@next-round/game-cipher` via `workspace:*`, so the monorepo build
   compiles it alongside the platform's own packages and its dependencies
   on `@next-round/contracts` / `game-sdk` / `game-client` resolve to the
   platform's copies. A registry- or git-sourced dependency does **not**
   work: this package ships sources only (no committed `dist/`, no
   `prepare` script) and a platform build never compiles a dependency's
   sources — workspace inclusion is what makes it build. Keep the fetched
   tree byte-identical to the pinned upstream commit.
2. Register it next to the bundled games — a `cipher-module.ts` in each app
   that calls `registry.register(manifest, server, { clientRoot, isDefault: true })`
   (mirroring `trivia-module.ts`), invoked from the API server and worker
   startup, plus a `cipher` entry in `install-bundled-games.ts`.
3. Add the client bundle input `games/cipher/0.1.0/client` → this package's
   `src/client.ts` in the web app's Vite config, plus match creation
   settings UI for `preset` / `turnSeconds`.
4. CI and the deployment image build run the fetch step before
   `pnpm install` / the Docker build.

The core repository intentionally carries no Cipher code; deployments that
don't want the game simply never bundle it.

## License

Apache-2.0.
