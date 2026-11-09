# CLAUDE.md - @sudobility/heavymath_lib

> **Git policy — never auto-commit or auto-push.** Leave work in the working tree.
> Run `git commit` / `git push` / `gh pr create` **only when the user asks in that turn**.
> Finishing a task is not permission to commit it. Do not use feature branches; stay on the current branch.

## What it is

Platform-agnostic React **hooks** library for the Heavymath prediction market platform
(works on React web and React Native). Every hook fetches sports data via a proxy hook from
`@sudobility/heavymath_indexer_client`, fetches the user's favorites via that client's
`useFavorites`, and merges them so each item gets a `favorited: boolean` flag and a
`setFavorited(id, bool)` method. The library adds the favorites-merge layer on top of the
indexer client — it does no network/contract work of its own.

Package `@sudobility/heavymath_lib` (BUSL-1.1, author John Huang). ESM-only (`"type": "module"`),
published to **GitHub Packages** (`@sudobility` scope → `https://npm.pkg.github.com`, see `.npmrc`).

## Stack

- TypeScript 6.0 (strict, all strict flags; target ES2020, module ESNext, `moduleResolution: bundler`, `jsx: react`)
- Vitest 4 (happy-dom env), `@testing-library/react`
- ESLint 10 (flat config) + Prettier 3
- Bun as package manager / script runner; peer deps: `@tanstack/react-query` >=5, `react` >=18

## Commands (verified against package.json)

```bash
bun run build          # tsc -p tsconfig.build.json, then node scripts/fix-esm-imports.mjs
bun run build:watch    # tsc --watch
bun run clean          # rm -rf dist
bun run test:unit      # vitest run (single pass) — THERE IS NO `bun run test`
bun run test:watch     # vitest (watch)
bun run typecheck      # tsc --noEmit
bun run lint           # eslint src
bun run lint:fix       # eslint --fix
bun run format         # prettier --write
bun run verify         # typecheck + lint + test:unit  (run before committing)
```

No `test`, `test:coverage`, or `test:run` script. For coverage run `bunx vitest run --coverage`
(v8 provider, 70% threshold for branches/functions/lines/statements, config in `vitest.config.ts`).
Scripts call `vitest`/`tsc` from `node_modules/.bin`, so `bun install` must have run first.

## Layout

```
src/
  index.ts                     # re-exports ./hooks and ./market
  hooks/
    index.ts                   # re-exports ./sports, ./useFavorites, ./useDiscussionForEntity
    useFavorites.ts            # enriches raw favorites with resolved names/images (24h staleTime+gcTime)
    useDiscussionForEntity.ts  # discussion metadata + auth state composition
    sports/
      index.ts                 # re-exports all 10 sport dirs
      football|basketball|nfl|baseball|hockey|rugby|handball|volleyball/
                               #   use{Sport}Leagues/Teams/Games(.ts) + __tests__/
      mma/                     # useMmaCategories / useMmaFighters / useMmaFights
      f1/                      # useF1Drivers / useF1Teams / useF1Races / useF1Circuits
      utils/seasons.ts         # season normalize / latest-season helpers
  market/index.ts              # empty: `export {}` (equilibrium code removed)
  __tests__/index.test.ts      # smoke test: all hooks are exported
  test/setup.ts                # vitest setup (afterEach cleanup)
```

Tests: 33 `*.test.ts` files, ~199 cases. Build config `tsconfig.build.json` extends `tsconfig.json`
but also excludes `**/__tests__/**`.

## The core pattern (every sports hook is identical)

```ts
function useXxxYyy(
  indexerClient: IndexerClient,        // for the favorites API calls
  walletAddress: string | undefined,   // undefined => no favorites fetched
  options?: { params?; enabled? }
): {
  items: XxxYyyWithFavorite[];   // sports data, each with a `favorited` flag
  isLoading: boolean;            // true if EITHER the sports OR favorites query is loading
  isError: boolean;              // from the sports query ONLY (not favorites)
  error: Error | null;           // from the sports query ONLY
  setFavorited: (id: number, favorited: boolean) => Promise<void>;
  favoritesLoading: boolean;
  addFavoritePending: boolean;
  removeFavoritePending: boolean;
}
```

Internals: call the indexer_client proxy hook for data + `useFavorites` for the user's favorites,
build a `Set` of favorited IDs for O(1) lookup, merge a `favorited` flag onto each item.
Favorites use a three-level key `category='sports'` / `subcategory=<sport>` / `type`
(e.g. football `league|team|match`; most team sports `league|team|game`;
mma `category|fighter|fight`; f1 `driver|team|race|circuit`).

## Conventions

- Files `use{Sport}{Entity}.ts`; types `{Sport}{Entity}WithFavorite`, `Use{Sport}{Entity}Options/Result`.
- Favorites constants are file-level consts: `FAVORITES_CATEGORY/SUBCATEGORY/TYPE`.
- Interface types co-located with their hook (no central `types/` dir).
- Each sport dir has an `index.ts` re-export; tests live in a sibling `__tests__/`.
- ESLint: `no-explicit-any` off; unused vars must be prefixed `_`; `prefer-const`/`no-var`/
  `object-shorthand`/`prefer-template` are errors; `react-hooks/exhaustive-deps` warns; test files
  are excluded from lint entirely (via `ignores`).
- Prettier: single quotes, semicolons, trailing commas `es5`, width 80, 2-space, `arrowParens: avoid`, LF.
- Test pattern: `vi.mock('@sudobility/heavymath_indexer_client')`, wrap `renderHook` in a
  `QueryClientProvider` (with `retry: false`), set default mocks in `beforeEach`.

## Gotchas

- **No `bun run test`** — use `bun run test:unit` (single) or `bun run test:watch`.
- **Football uses "Matches"** (`useFootballMatches`) wrapping the API's fixtures; all other
  team sports use "Games". **MMA** uses categories/fighters/fights; **F1** has four entity types.
- **MMA categories are strings**, not objects with numeric IDs.
- **ID type mismatch**: sports API returns numeric IDs, favorites store strings — hooks `String(id)` when comparing/storing.
- `isLoading` = sports OR favorites loading; `isError`/`error` = sports query only.
- Build uses `tsconfig.build.json` (not `tsconfig.json`); output `dist/**/*` only. A post-build
  step (`scripts/fix-esm-imports.mjs`) rewrites extensionless relative imports to explicit `.js`
  paths so Node's native ESM loader accepts the output. Stale `dist/market/` artifacts may remain
  until the next `clean && build`.

## Adding a new sport

1. `src/hooks/sports/{sport}/` with `use{Sport}{Entity}.ts` files following an existing sport.
2. `index.ts` re-exporting them; add `export * from './{sport}';` to `src/hooks/sports/index.ts`.
3. Add tests under `__tests__/`; run `bun run verify`.

## CI / publish

- `.github/workflows/ci-cd.yml` calls the reusable `johnqh/workflows/.github/workflows/unified-cicd.yml@main`
  with `publish-target: github` (GitHub Packages). Triggers on push/PR to `main` and `develop`.
- `prepublishOnly` runs `clean && build` automatically. `publishConfig.access: "public"`.

## Ecosystem

`heavymath_contracts` (Solidity) → `heavymath_indexer` (REST, incl. `/api/sports/*` proxy) →
`heavymath_indexer_client` (proxy + favorites hooks) → **heavymath_lib** (adds favorites merge) →
`heavymath_app` (consumer). When indexer_client adds a sports proxy hook, add a wrapper here.
