# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Git and remotes

This is the owner's personal fork. All work happens against the fork.

- `origin` = `KhanxLab/kasirgratisan` (the personal fork). Every push, branch and pull request goes here.
- `upstream` = `jipraks/kasirgratisan` (the original project). Never push to it, open PRs against it, or otherwise write to it.
- Commit identity is `KhanxLab <192100190+KhanxLab@users.noreply.github.com>`, set in this repo's local git config. Never commit with any work account (e.g. `@konecta.com`). Check `git config user.email` before committing.
- Use the `KhanxLab` account for `gh` operations.

## Project memory

Use Engram (project `kasirgratisan`) as the memory for this project: read it with `mem_context` / `mem_search` at the start of work and save decisions, fixes and conventions with `mem_save`. Don't keep separate Claude Code file memories for this project.

## Commands

```bash
npm run dev            # Vite dev server on http://localhost:8080
npm run build          # bumps version.json (scripts/bump-version.js), then vite build
npm run build:dev      # same, in development mode
npm run preview        # serve the production build
npm run lint           # eslint . (flat config in eslint.config.js)
npm test               # vitest run (one-shot)
npm run test:watch     # vitest in watch mode
npx vitest run src/test/sync.test.ts          # single test file
npx vitest run -t "name of test"              # single test by name
npm run cap:sync       # build + cap sync (Android)
npm run cap:android    # build + cap sync + open Android Studio
npm run cap:run        # build + cap sync + run on device/emulator
```

- `npm run build` and `build:dev` modify `version.json`. Don't commit that change by accident when you only meant to verify a build; restore it if it is unrelated to the task.
- Two lockfiles exist (`package-lock.json` and `bun.lock`). Keep both in sync when changing dependencies.
- Test setup is `src/test/setup.ts` (jsdom, globals, `fake-indexeddb` for Dexie). Tests live in `src/test/` only, matching `src/**/*.{test,spec}.{ts,tsx}`.
- Android release builds (`./gradlew bundleRelease` in `android/`) need JDK 21 and a signing config; see README.

## Architecture

**Fully local POS, one codebase, two shells.** The same React app ships as a PWA (vite-plugin-pwa, Workbox) and as an Android app (Capacitor 8). There is no backend: no cloud sync, no accounts, no billing, no analytics or push. Don't add network calls; everything must work offline.

**Local data is the only source of truth.** All business data lives in IndexedDB through `PosDatabase` in `src/lib/db.ts`. Schema changes go in a new `this.version(N).stores(...)` block. Never edit an existing version block, because Dexie runs them as migrations for users already on older versions. The latest is version 14. Fields such as `syncedAt`, `cloudStoreId` and the `deletedRecords` table are legacy leftovers from the removed cloud sync; they stay in the schema for existing data but nothing uses them. `setupTimestampHooks` keeps `updatedAt` current on create/update. Backup is a local JSON file (`src/lib/backup.ts`).

**Auth.** `src/hooks/use-auth.tsx` handles optional multi-user login (username + PIN hashed in `src/lib/auth.ts`). Owner and staff roles have per-permission flags. Gate sensitive UI and actions with `can()` from `useAuth()`, and only when multi-user is enabled.

**Routing and layout.** All routes are declared in `src/App.tsx`, nested under `AppLayout` (`src/components/layout/`). Each route wraps its page in an `ErrorBoundary`. Pages are in `src/pages/`; settings sub-pages are in `src/pages/settings/`. Reactive data binding uses `useLiveQuery()` from `dexie-react-hooks`, not manual state syncing.

**Conventions that matter here.**
- UI strings go through i18next. Two locales: `es` (Spanish, default and fallback) and `en`, each with JSON namespaces (`common`, `settings`, `products`, `reports`, `dashboard`, `onboarding`). Add keys to both locales. Use `useTranslation('namespace')` in components; plain modules (e.g. `src/lib/printer.ts`) use `i18n.getFixedT(resolveLanguage(...), ns, prefix)` from `@/i18n`. Only an explicit user choice switches away from Spanish (no browser-language detection).
- Dates use `getDateLocale(i18n.language)` from `@/i18n`; don't build per-file date-fns locale maps.
- The only currency is Colombian peso (COP), stored as integer pesos (no decimals). Format with `src/lib/currency.ts` (`CURRENCY_SYMBOL`, `NUMBER_LOCALE` = `es-CO`, `formatCurrency`) rather than hard-coded symbols or locales.
- Use the existing shadcn/ui components in `src/components/ui/`.

**Known doc drift.** The "Project Structure" section of `README.md` is outdated: it omits Customers, Debts, Expenses, and the settings sub-pages. Trust the code over that section.
