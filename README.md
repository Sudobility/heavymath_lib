# @sudobility/heavymath_lib

React hooks library for the Heavymath prediction market platform. Wraps sports data fetching with user favorites from `@sudobility/heavymath_indexer_client`.

## Installation

```bash
bun add @sudobility/heavymath_lib
```

This package is published to **GitHub Packages** under the `@sudobility` scope, so
consuming projects need an `.npmrc` pointing that scope at the GitHub registry
(and an auth token with `read:packages`):

```
@sudobility:registry=https://npm.pkg.github.com
```

The `@sudobility/*` peer dependencies live in the same registry.

## Usage

Every hook fetches sports data, fetches the user's favorites, then merges them so each item has a `favorited` flag and a `setFavorited()` method.

```typescript
import { useFootballLeagues, useBasketballGames, useF1Drivers } from '@sudobility/heavymath_lib';

function Leagues() {
  const { items, isLoading, setFavorited } = useFootballLeagues(indexerClient, walletAddress);

  return items.map(league => (
    <div key={league.id}>
      {league.name} {league.favorited ? '(fav)' : ''}
      <button onClick={() => setFavorited(league.id, !league.favorited)}>Toggle</button>
    </div>
  ));
}
```

## Available Hooks

### Standard Team Sports (8 sports x 3 hooks each)

Football, Basketball, NFL, Baseball, Hockey, Rugby, Handball, Volleyball:

- `use{Sport}Leagues` / `use{Sport}Teams` / `use{Sport}Games`
- Football uses `useFootballMatches` (not Games)

### MMA

- `useMmaCategories`, `useMmaFighters`, `useMmaFights`

### Formula 1

- `useF1Drivers`, `useF1Teams`, `useF1Races`, `useF1Circuits`

### Non-sport hooks

- `useFavorites` -- all of a user's favorites enriched with resolved display names/images (24h cache)
- `useDiscussionForEntity` -- discussion metadata merged with auth state (comment count, `canPost`, `isLocked`)

## Hook Signature

```typescript
function useXxxYyy(
  indexerClient: IndexerClient,
  walletAddress: string | undefined,
  options?: { params?; enabled? }
): {
  items: XxxYyyWithFavorite[];
  isLoading: boolean;
  isError: boolean;
  error: Error | null;
  setFavorited: (id: number, favorited: boolean) => Promise<void>;
  favoritesLoading: boolean;
  addFavoritePending: boolean;
  removeFavoritePending: boolean;
}
```

## Development

```bash
bun run build        # Compile TypeScript (tsconfig.build.json) + fix ESM imports
bun run test:unit    # Run Vitest once (199 tests across 33 files)
bun run test:watch   # Run Vitest in watch mode
bun run typecheck    # TypeScript validation (tsc --noEmit)
bun run lint         # ESLint check
bun run verify       # typecheck + lint + test:unit
```

> There is no `bun run test` script; the single-run command is `bun run test:unit`.

## Peer Dependencies

Provided by the consuming app (must be installed alongside this package):

- `@sudobility/heavymath_indexer_client` -- sports data types, proxy hooks, favorites API (the hooks wrap this)
- `@sudobility/heavymath_types`, `@sudobility/types` -- shared type definitions
- `@sudobility/heavymath_contracts`, `@sudobility/auctions_contracts` -- contract type definitions
- `@tanstack/react-query` >= 5.0.0 -- data fetching/caching layer
- `react` >= 18.0.0 (works on both React web and React Native)

## Related Packages

- `@sudobility/heavymath_types` -- shared type definitions
- `@sudobility/heavymath_indexer_client` -- indexer API client
- `heavymath_app` -- frontend web application

## License

BUSL-1.1
