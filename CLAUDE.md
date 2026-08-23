# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Important

Expo SDK 57 is very recent — its APIs differ from older SDKs you may know. Before writing any code, check the exact versioned docs at https://docs.expo.dev/versions/v57.0.0/ rather than relying on memory.

## Commands

- `npm start` — start the Expo dev server (Metro)
- `npm run android` / `npm run ios` / `npm run web` — start the dev server targeting a specific platform
- `npm run lint` — runs `expo lint`; **not yet initialized** (no ESLint/Prettier config or packages installed). First run will trigger Expo's interactive setup rather than actually lint.
- `npm run reset-project` — stock create-expo-app script that moves the current `src/app` scaffold to `app-example/` and creates a blank `src/app`. Only relevant if intentionally discarding the starter template.
- There is no test script and no test runner installed (no Jest, no `__tests__`). Set this up before assuming any test command exists.

## Architecture

This is currently the stock create-expo-app SDK 57 template (`src/`-rooted variant, typed routes) with the app renamed to "watchlist-app" — no watchlist domain logic, state management, persistence, or API layer has been built yet. Expect to design those from scratch.

- **Entry point**: `expo-router/entry`, routing is file-based under `src/app/`. Only `index.tsx` (Home) and `explore.tsx` (Explore) exist, wired as tabs by `src/components/app-tabs.tsx` via `expo-router/unstable-native-tabs` (`NativeTabs`).
- **Path aliases** (`tsconfig.json`): `@/*` → `src/*`, `@/assets/*` → `assets/*`.
- **Platform-split files**: components/hooks with both `.tsx` and `.web.tsx` versions (e.g. `app-tabs.tsx`/`app-tabs.web.tsx`, `animated-icon.tsx`/`animated-icon.web.tsx`, `use-color-scheme.ts`/`.web.ts`) provide separate native vs. web implementations — Metro/webpack picks the right one automatically by platform extension. When editing one, check whether the sibling `.web.tsx` needs the same change.
- **Theming**: `src/constants/theme.ts` defines light/dark `Colors`, `Fonts`, and a `Spacing` scale as plain design tokens — no styling library (Nativewind/Tamagui/unistyles) is used. `ThemeProvider` (from `expo-router`) in `src/app/_layout.tsx` switches between `DefaultTheme`/`DarkTheme` based on `useColorScheme()`.
- **No state management, persistence, or API layer exists** — no Context providers beyond the router's ThemeProvider, no Redux/Zustand/Jotai/React Query, no AsyncStorage/SQLite, no `services/`/`api/`/`store/` directories, and no environment variable usage (no `.env` files). These will need to be introduced when watchlist data (movies/shows, persistence, any backend) is actually built.
- `app.json`: deep-link scheme is `watchlistapp`; `experiments.typedRoutes` and `experiments.reactCompiler` are both enabled.
