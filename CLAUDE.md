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

- **Entry point**: `expo-router/entry`, routing is file-based under `src/app/`. `index.tsx` (Home), `explore.tsx` (Explore), and `about.tsx` (About, placeholder) exist, wired as tabs by `src/components/app-tabs.tsx` via `expo-router/unstable-native-tabs` (`NativeTabs`). A tab's `name` prop must match its route filename (no `.tsx`).
- **Path aliases** (`tsconfig.json`): `@/*` → `src/*`, `@/assets/*` → `assets/*`.
- **Platform-split files**: components/hooks with both `.tsx` and `.web.tsx` versions (e.g. `app-tabs.tsx`/`app-tabs.web.tsx`, `animated-icon.tsx`/`animated-icon.web.tsx`, `use-color-scheme.ts`/`.web.ts`) provide separate native vs. web implementations — Metro/webpack picks the right one automatically by platform extension. When editing one, check whether the sibling `.web.tsx` needs the same change.
- **Theming**: `src/constants/theme.ts` defines light/dark `Colors`, `Fonts`, and a `Spacing` scale as plain design tokens — no styling library (Nativewind/Tamagui/unistyles) is used. `ThemeProvider` (from `expo-router`) in `src/app/_layout.tsx` switches between `DefaultTheme`/`DarkTheme` based on `useColorScheme()`.
- **No state management, persistence, or API layer exists yet** — no Context providers beyond the router's ThemeProvider, no Redux/Zustand/Jotai/React Query, no AsyncStorage/SQLite, no `services/`/`api/`/`store/` directories, and no environment variable usage (no `.env` files). See **Data source** below for what this app is meant to connect to and mirror.
- `app.json`: deep-link scheme is `watchlistapp`; `experiments.typedRoutes` and `experiments.reactCompiler` are both enabled.

## Data source — porting an existing web app's Supabase backend

This app is a React Native/Expo port of an existing web watchlist app that lives in a **separate repo**, `F:\farudesigns\Watchlist\watchlist-app` (Vite + React + TypeScript SPA, deployed to Vercel, not accessible from this repo/machine setup in every environment — summarized here so that context isn't lost). Both apps are meant to share the same Supabase backend. Build order for the port is tracked in `ROADMAP.md` in this repo — check it for current progress before re-planning.

**Auth model — no login.** There's no Supabase Auth session. Reads are open via RLS (`for select using (true)`). Writes are gated by a shared "access code": the client sends it as an `x-app-secret` header, and a Postgres `SECURITY DEFINER` function `public.check_write_secret()` compares it against a secret in `private.app_config`; RLS insert/update/delete policies call that function. The anon key is intentionally public — RLS is the real gate. On the web app the code is cached in `localStorage`; on this app it should use `expo-secure-store` instead.

**Schema** (Postgres, applied manually via `supabase/schema.sql` in the web repo — no migrations folder):
- `public.titles` — the core table, one row per movie or show. Key columns: `id uuid`, `media_type` (`'movie'|'show'`), `tmdb_id`, `tvdb_id`, `name`, `poster_url`, `year`, `genres text[]`, `overview`, `runtime_minutes` (movies), `imdb_id` (movies), `show_status`/`watched_episode_count`/`aired_episode_count` (shows), `status` (`'watched'|'watch_later'|'following'|'dropped'`), `is_favourite bool`, `watched_at`, `last_watched_at`, `added_at`, `release_date` (movies, for Upcoming), `next_episode_air_date`/`next_episode_season`/`next_episode_number` (shows, for Upcoming). Unique on `(media_type, tmdb_id)` where `tmdb_id is not null`.
- `public.title_seasons` / `public.title_episodes` — per-episode watch tracking, FK `title_id → titles.id on delete cascade`. `title_episodes` has a `watched bool` + `watched_at`.
- `public.tmdb_upcoming_movies` — cache table for the Upcoming view, read-only to clients (no insert/update/delete policy for `anon`/`authenticated` at all — only a service-role Edge Function writes it, via daily `pg_cron`).
- `public.tmdb_swipe_skips` — tracks skipped titles for a Tinder-style "Discover" swipe feature.
- All client-facing tables share the same RLS shape: open select, `check_write_secret()`-gated writes.

**TMDB integration**: never called directly from any client. Proxied through a Supabase Edge Function (`tmdb-proxy`, in the web repo's `supabase/functions/`) which itself checks the `x-app-secret` header (against secret `APP_WRITE_SECRET`) and holds the real `TMDB_API_KEY` server-side. Supported actions: `search`, `details`, `season_episodes`, `swipe_feed`, `discover`. Call it the same way from this app: `supabase.functions.invoke('tmdb-proxy', { body: { action, ... } })` — no new backend code needed for feature parity.

**Patterns to mirror from the web app** (`web/src/` in the web repo), adapted for RN:
- `lib/supabaseClient.ts` — builds the Supabase client with the `x-app-secret` header baked in from the cached access code; exposes `getSupabase()` / `refreshSupabaseClient()`.
- `lib/withAccessCode.ts` — wraps every write call; on an RLS/401 rejection (Postgres code `42501`), clears the cached code, rebuilds the client, re-prompts for the code, retries once.
- TanStack Query hooks per concern: `useTitles`, `useMutateTitle`, `useAddTitle`, `useTmdbSearch`, `useEpisodes`, `useToggleEpisode`, `useUpcoming`, `useSwipeCandidates`/`useSwipeActions`, `useAdvanceSearch`. Reads never go through `withAccessCode` (they're open); only writes do.
- `types.ts` — hand-written TypeScript types matching the schema columns 1:1 (no generated `database.types.ts`).

**Env vars needed in this app** (none exist yet): `EXPO_PUBLIC_SUPABASE_URL`, `EXPO_PUBLIC_SUPABASE_ANON_KEY` — Expo's public-env prefix, equivalent to the web app's `VITE_SUPABASE_URL`/`VITE_SUPABASE_ANON_KEY`. Get the actual values from the web repo's `web/.env.local` or the Supabase dashboard (project ref `qlkntcqzhgfueuyecbji`).
