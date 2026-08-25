# Watchlist Mobile App — Roadmap & Learning Log

Porting the existing web watchlist app (`F:\farudesigns\Watchlist\watchlist-app`, Vite + React + Supabase) to this Expo/React Native app, same Supabase backend. Written as a step-by-step build order for someone learning React Native by building — each phase introduces a small set of new concepts before moving on.

## How to use this file

- Tasks are checkboxes. Tick `[ ]` → `[x]` as each is done.
- Work top to bottom — each phase assumes the previous one works.
- Ask for a walkthrough of any single task before starting it; don't need to plan ahead.
- When a phase finishes, add a dated entry under **Progress Log** at the bottom (what you built, anything that tripped you up) — same idea as the web app's own ROADMAP.md.

---

## Phase 0 — Orientation (Expo & this project, no Supabase yet)

Goal: get comfortable navigating and running the existing scaffold before adding anything.

- [x] Run `npm start` and open the app in Expo Go (or a simulator) — confirm it launches
- [ ] Read `src/app/_layout.tsx` and `src/components/app-tabs.tsx` — understand how the two tabs (Home/Explore) are wired via `expo-router`'s file-based routing
- [ ] Read `src/app/index.tsx` and `src/app/explore.tsx` — see how a screen component is structured
- [ ] Make a trivial edit (change some text) in `index.tsx`, save, see it hot-reload on device
- [ ] Skim `src/constants/theme.ts` and `src/hooks/use-theme.ts` — understand the light/dark token system
- [ ] Understand the `@/*` path alias (`tsconfig.json`) and the `.tsx` vs `.web.tsx` platform-split file pattern

## Phase 1 — Supabase connection

Goal: talk to the same Supabase project the web app uses, prove a read works.

- [ ] Install `@supabase/supabase-js`
- [ ] Get the Supabase URL + anon key from the web app's `web/.env.local` (or Supabase dashboard) and add them to this project as `EXPO_PUBLIC_SUPABASE_URL` / `EXPO_PUBLIC_SUPABASE_ANON_KEY` (Expo's public-env prefix, equivalent to Vite's `VITE_` prefix)
- [ ] Create a Supabase client module (mirrors `web/src/lib/supabaseClient.ts`)
- [ ] Write a throwaway test: fetch all rows from `titles` and `console.log` them — confirm the connection works end to end
- [ ] Understand *why* there's no Supabase Auth session here (the web app's access-code + RLS model) before building anything that writes

## Phase 2 — Display the watchlist (read-only)

Goal: real UI showing real data — the first "it feels like an app" milestone.

- [ ] Install `@tanstack/react-query` and set up a `QueryClientProvider` (mirrors the web app's data-fetching pattern)
- [ ] Learn `FlatList` (React Native's list-rendering component — the RN equivalent of mapping over an array in a CSS grid on web)
- [ ] Build a `useTitles` hook mirroring `web/src/hooks/useTitles.ts` (query by `media_type`, ordered by `added_at`)
- [ ] Render a basic list screen: title name, poster image, year
- [ ] Add loading and error states
- [ ] Add the movie/show type switch and status tabs (mirrors `TypeSwitch`/`StatusTabs` on web), matching the tab order in the web app's `TYPE_CONFIG`

## Phase 3 — Access-code write gate

Goal: implement the write-side auth pattern and make your first successful write.

- [ ] Install `expo-secure-store` (RN's equivalent of `localStorage` for the access code — more secure, since it's on-device encrypted storage)
- [ ] Build an access-code prompt (modal) mirroring `AccessCodePrompt`/`AccessCodeContext` on web
- [ ] Port `withAccessCode()` — the wrapper that catches an RLS/401 rejection, clears the cached code, re-prompts, retries once
- [ ] Wire the Supabase client to send the `x-app-secret` header on writes, same as the web client
- [ ] Test it: toggle a title's `is_favourite` from the app and confirm it updates in Supabase

## Phase 4 — Add a title via TMDB

Goal: the full add-new-title flow, including the server-proxied search.

- [ ] Call the existing `tmdb-proxy` Edge Function from RN (`supabase.functions.invoke`, same as web — no new backend code needed)
- [ ] Build a search UI (text input + results list)
- [ ] Implement "add title" — insert into `titles`, and for shows, prefetch `title_seasons`/`title_episodes` (mirrors `useAddTitle.ts`)

## Phase 5 — Title detail & episode tracking

- [ ] Build a title detail screen (navigate to it from the list — new `expo-router` concept: dynamic routes)
- [ ] Show season/episode list for shows, with watched toggles (mirrors `useEpisodes`/`useToggleEpisode`)

## Phase 6 — Secondary features (only after 1–5 feel solid)

- [ ] Upcoming view (`release_date` / `next_episode_air_date`, mirrors `useUpcoming`)
- [ ] Discover swipe page (mirrors `useSwipeCandidates`/`useSwipeActions`) — new concept: gesture handling with `react-native-gesture-handler`/`react-native-reanimated` (already installed in this project)
- [ ] Advanced search page (mirrors `useAdvanceSearch`)

## Phase 7 — Polish & tooling

- [ ] Set up ESLint (`npm run lint` currently unconfigured — see `CLAUDE.md`)
- [ ] App icon / splash review for the mobile-specific branding
- [ ] Decide if/what needs a test setup (none exists yet)

---

## Progress Log

_Add an entry each time a phase (or a meaningful chunk of one) is completed — date, what got built, what was learned or got stuck._

<!-- Example:
### 2026-08-24 — Phase 0 done
Ran the scaffold, understood tab routing and the theme system. Confused for a bit about why `explore.tsx` maps to a route automatically — got it once I understood expo-router's file-based convention.
-->
