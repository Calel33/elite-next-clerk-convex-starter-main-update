# HANDOFF — Phase 2: Dependency Majors

**Repo:** `C:\Users\calvi\Desktop\elite-next-clerk-convex-starter-main-update`
**Created:** 2026-10-02, from the session that completed Phase 1 (safe dependency updates)
**Next-session focus:** update the remaining breaking majors (Clerk 7, Zod 4, svix 2, TanStack Table 9, lucide-react 1.x, motion 13/14) — with official migration docs and per-step verification.
**Status: ✅ COMPLETED 2026-10-02 — see §9 for the completion log and follow-ups.**

---

## 0. TL;DR

- Phase 1 (all patches/minors + safe majors) is **complete and verified; committed 2026-10-02 as `80c9d8f`** (plain git — the `gt` CLI is not installed in this environment).
- Package manager is **bun**; `bun.lock` is the only lockfile (npm/pnpm lockfiles were deleted with user approval — do not recreate).
- A **7-day release cooldown** is project policy for all installs/updates (§3).
- `bun run lint` is **red for pre-existing reasons** unrelated to dependency versions (§5). Do not chase it during Phase 2 unless the user asks.
- TypeScript 7 is **blocked** by tooling (§2). Do not include it.

## 1. Working tree state (as of handoff)

```
 M .gitignore        (user's graft-ignore block + session-added /jscpd-report.json entry)
 M bun.lock          (regenerated from scratch — Phase 1)
 M knip.json         (classMembers removed for knip v6)
 D package-lock.json (deleted with approval)
 M package.json      (Phase 1 versions + eslint dependency added)
 D pnpm-lock.yaml    (deleted with approval)
 ?? .ignore          (user's own file — leave alone)
```

- `node_modules` installed. Runtime: bun 1.3.14, Node v24.18.0.
- Phase 1 key marker versions: `next 16.3.6` (exact pin), `react/react-dom 19.3.0`, `convex 1.46.0`, `@clerk/nextjs 6.39.7`, `typescript 5.9.3`, `eslint 10.11.0`, `knip 6.38.0`. Full list: uncommitted `git diff package.json` (do not duplicate).
- **Recommended prerequisite:** get Phase 1 committed first (user left it uncommitted for review) so Phase 2 diffs stay separable. Commit only on request; repo rules prefer Graphite for stacked changes.
- Side-effect left intentionally: `prepare: "husky install"` prints a deprecation warning during install and exits 0; `.husky/` is gitignored and maintained separately.

## 2. Phase 2 scope (fresh ncu snapshot 2026-10-02, cooldown 7)

| Package | Current | Target (cooldown) | Latest | Notes / blast radius |
|---|---|---|---|---|
| `@clerk/nextjs` | ^6.39.7 | **^7.9.7** | 7.9.10 | Auth-critical, used app-wide (middleware.ts, app/layout.tsx, landing header, convex/auth.config.ts, components/custom-clerk-pricing.tsx). Read Clerk's official upgrade guide FIRST. Peers confirmed compatible with Next ^16 + React ~19.3. |
| `@clerk/backend` | ^2.33.7 | **^3.20.1** | 3.22.0 | Pair with the Clerk 7 step; session/webhook helpers live under `convex/`. |
| `zod` | ^3.25.76 | **^4.6.5** | same | Single import: `app/dashboard/data-table.tsx` (L54). Read zod v4 migration notes. |
| `svix` | ^1.99.1 | **^2.5.0** | 2.6.1 | `convex/http.ts` webhook signature verification (`verify()` ~L48-62). SECURITY-CRITICAL — do not weaken verification; review v2 API changes. |
| `@tanstack/react-table` | ^8.21.3 | **^9.2.4** | same | One consumer: `app/dashboard/data-table.tsx` (722 lines). Read v9 migration guide. |
| `lucide-react` | ^0.577.0 | **^1.48.0** | 1.50.0 | Icon imports across ~8 files (components/ui/*, landing header/hero, kokonutui). Typecheck/build catches removed/renamed exports. |
| `motion` | ^12.43.0 | **^13.4.4** | 14.0.0 | `motion/react` imports in ~7 files (magicui, motion-primitives, kokonutui, ui/text-effect). |
| `framer-motion` | ^12.43.0 | **^13.4.4** | 14.0.0 | Only `components/react-bits/text-cursor.tsx`. Opportunity: consolidate to `motion` (code change → needs user approval). |
| `typescript` | ^5.9.3 | — | 7.0.2 | **BLOCKED**: typescript-eslint 8.71 peers `typescript >=4.8.4 <6.1.0`. Do not bump until typescript-eslint widens support. |
| `@types/node` | ^24.13.6 | (26.6.2) | 26.6.4 | Deliberately on **24.x to match runtime Node v24.18.0**. Revisit only if runtime changes or the user opts in. |

Re-run `npx --yes npm-check-updates --cooldown 7` at the start of the session — versions move. `next` was not flagged (16.3.6 was the cooldown-eligible latest at handoff).

## 3. Conventions & constraints (binding)

- **bun only.** All installs/updates must pass the cooldown: `bun install --minimum-release-age=604800` / `bun update --minimum-release-age=604800`. ncu: `--cooldown 7`.
- **Gotcha (proven in Phase 1):** `bun update` (even `--force`) keeps stale transitive pins from an existing `bun.lock`. If `bun audit` flags transitives, regenerate from scratch: delete `bun.lock`, then `bun install --minimum-release-age=604800` (this cleared 8 dev advisories in Phase 1).
- **Repo rules (AGENTS.md / CLAUDE.md):** graft-first before reading files (`graft ask|grep|skeleton|callers` — repo is graft-indexed); read before edit; no deletions without explicit approval; files ≤500 lines; design tokens only.
- **middleware → proxy:** ~~keep `middleware.ts` (DEFER)~~ **Superseded 2026-10-02:** Clerk's Next-16 docs now target `proxy.ts`; migrated with user approval (rename only). See §9 and the updated `research/nextjs-16-proxy-migration-clerk.md`.
- One major at a time; verify (tsc + build + audit) between steps; user approval for code changes beyond version bumps (e.g. framer-motion consolidation).
- **No commits unless asked.** If asked, prefer the Graphite workflow (project rule).

## 4. Verification recipes

```powershell
# types
bunx tsc --noEmit

# build (no .env.local exists; build-only dummy env, create no files)
$key = 'pk_test_' + [Convert]::ToBase64String([Text.Encoding]::UTF8.GetBytes('dummy-123.clerk.accounts.dev$'))
$env:NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY=$key
$env:CLERK_SECRET_KEY='sk_test_dummy_for_build_only'
$env:NEXT_PUBLIC_CLERK_FRONTEND_API_URL='https://dummy-123.clerk.accounts.dev'
$env:NEXT_PUBLIC_CONVEX_URL='https://dummy-123.convex.cloud'
bun run build

# security
bun audit

# lint status (informational — red pre-existing, see §5)
bun run lint
```

**Phase 1 baseline to compare against:** tsc exit 0 · build 6/6 routes (`/`, `/_not-found`, `/dashboard`, `/dashboard/payment-gated`) · `bun audit` = "No vulnerabilities found" · build prints the expected middleware→proxy deprecation warning · `bun run lint` = 229 pre-existing errors.

## 5. Pre-existing reds — do not confuse with Phase 2 regressions

| Symptom | Cause | Evidence |
|---|---|---|
| `bun run lint` → 229 errors | `eslint.config.js` naming-convention forbids PascalCase for `variable`/`parameter` selectors → flags every React component; `react-hooks/exhaustive-deps` referenced without the plugin installed; `scripts/*.js` lack node globals. | Config L39-43 + Apr 2026 flat-config commit — version-independent. Needs a dedicated task (may require adding `eslint-plugin-react-hooks`). |
| `bun run lint:tech-debt` crashes | `scripts/check-tech-debt.js` is CommonJS while package is `"type": "module"`. | Pre-existing since Apr 2026. |
| `bun run lint:dead-code` exit 1 | 18 unused deps + 5 unused devDeps — identical under knip 5.88.1 (A/B verified). | Pre-existing. Several flagged packages ARE referenced in code → review knip config; do not delete deps blindly. |
| `bun run format:check` → 62 files | Formatting drift predating the update. | A/B with prettier 3.2.5 shows the same 62 files. |
| Build warning: middleware→proxy | Next 16 deprecation. | Expected; deferred per §3. |

## 6. Reference artifacts (read, don't duplicate)

- Uncommitted Phase 1 diff — `git diff` (package.json, bun.lock, knip.json, deleted lockfiles)
- `research/template-update-plan.md` — previous cycle plan (Next 15 → 16; documents why zod was deferred before)
- `research/template-update-checklist.md` — previous cycle checklist
- `research/nextjs-16-proxy-migration-clerk.md` — proxy migration research; recommendation: DEFER (refresh before acting)
- `prompts/001-update-template-dependencies.md` — original research prompt (previous cycle)
- `README.md` · `CLAUDE.md` · `AGENTS.md` — project conventions (CLAUDE.md prefers bun tooling)
- No secrets were involved in this session. The build recipe uses non-secret dummy values only; no `.env.local` exists — do not create one unless the user provides real credentials.

## 7. Suggested skills for the next session

1. `graft` — mandatory (repo is indexed; graft before any file read)
2. `incremental-implementation` — one major per increment, verify between
3. `source-driven-development` — fetch official migration docs per major before editing (Clerk, zod, svix, TanStack, lucide, motion)
4. `code-review-and-quality` — review each increment before moving on
5. `debugging-and-error-recovery` — if a bump breaks the build/runtime
6. `graphite-cli` — only if the user asks to commit/stack
7. `alignment` — optional end-of-task docs check

## 8. Done criteria for Phase 2

- [ ] Each §2 major either bumped to the latest cooldown-eligible version or explicitly deferred with a recorded reason (esp. TS 7, @types/node)
- [ ] `bun audit` clean (or documented non-fixable dev-only advisory)
- [ ] `bunx tsc --noEmit` exit 0
- [ ] `bun run build` green with the same route table
- [ ] Auth wiring verified per Clerk guidance; proxy decision documented (update the research doc)
- [ ] Changes left for user review; commit only on request

## 9. Phase 2 completion log (2026-10-02)

**Result: all in-scope majors upgraded; verification green; Phase 2 changes left uncommitted for review.**

### Commits

- Phase 1 committed first as `80c9d8f` "Update dependencies to latest safe versions (Phase 1)" (plain git; repo-local git identity set from repo history: `Calel33 <callovecrypto03@protonmail.com>`).
- Phase 2 changes remain uncommitted (per §8).

### Per-package outcomes

| Package | From | To (installed) | Code changes required |
|---|---|---|---|
| `zod` | ^3.25.76 | 4.6.5 | None (`z.object`/`z.infer` are v4-safe) |
| `lucide-react` | ^0.577.0 | 1.48.0 | None (no brand icons in use) |
| `motion` | ^12.43.0 | 13.4.4 | None (v13 breaking change only affects `@emotion/is-prop-valid` consumers) |
| `framer-motion` | ^12.43.0 | **removed** | `components/react-bits/text-cursor.tsx` now imports `motion/react` (consolidation approved) |
| `svix` | ^1.99.1 | 2.5.0 | `convex/http.ts`: v2 `Webhook.verify()` no longer returns the payload (v2.2.0 removed JSON parsing; it returns `undefined` and throws on invalid signature). Now `wh.verify(payloadString, svixHeaders)` then `JSON.parse(payloadString)`. Signature verification unchanged. |
| `@tanstack/react-table` | ^8.21.3 | 9.2.4 | `app/dashboard/data-table.tsx`: `useReactTable`→`useTable` + `features: stockFeatures`; removed `get*RowModel()` options; `table.getState().pagination`→`table.state.pagination` (3 spots); types `ColumnDef<StockFeatures, T>` / `Row<StockFeatures, T>`; `VisibilityState`→`ColumnVisibilityState`. |
| `@clerk/nextjs` | ^6.39.7 | 7.9.7 | `app/dashboard/payment-gated/page.tsx`: `<Protect condition={...}>`→`<Show when={...}>` (`Protect` is a removed stub that throws in v7). Themes migrated in `app/(landing)/header.tsx`, `app/dashboard/nav-user.tsx`, `components/custom-clerk-pricing.tsx`: `@clerk/themes` + `appearance.baseTheme` → `@clerk/ui/themes` + `appearance.theme`. |
| `@clerk/backend` | ^2.33.7 | 3.20.1 | None (`WebhookEvent`/`UserJSON` types still exported) |
| `@clerk/ui` | — | 1.36.0 (new dep) | New direct dependency for themes (per Core 3 docs) |
| `@clerk/themes` | ^2.4.64 | **removed** | No imports remain (verified via graft) |
| `middleware.ts` | — | → `proxy.ts` | File rename + `/__clerk/(.*)` matcher line (Clerk Next-16 guidance). `createRouteMatcher` kept intentionally (deprecated — see follow-ups). |

### Deferred with recorded reason

- `typescript` 7.x — blocked: `typescript-eslint` 8.x peers `typescript >=4.8.4 <6.1.0`. Revisit when typescript-eslint widens its peer range.
- `@types/node` 26.x — deliberate: runtime is Node v24.18.0; keep types on 24.x. Revisit only if runtime changes.
- `motion`/`framer-motion` 14.0.0 — not cooldown-eligible at execution time; `^13.4.4` installed. Bump to 14 after 2026-10-09 if desired.
- `@clerk/ui` 1.37+/1.38+ — cooldown; re-evaluate after their cooldown windows open (1.37.0 → 2026-10-06, 1.38.0 → 2026-10-07).

### Verification evidence (2026-10-02)

| Check | Result |
|---|---|
| `bunx tsc --noEmit` | exit 0 (run after every step) |
| `bun run build` | green; routes `/`, `/_not-found`, `/dashboard`, `/dashboard/payment-gated`; 6/6 static pages; middleware→proxy deprecation warning gone |
| `bun audit` | 2 moderate advisories (below) |
| `bun run lint` | 241 problems / **229 errors — identical to the Phase 1 baseline**; 12 warnings; pre-existing, version-independent |

**Audit detail:** both advisories come from the new `@clerk/ui` transitive chain: `@clerk/ui › @solana/wallet-adapter-react › @solana/wallet-adapter-base › @solana/web3.js › jayson › stream-json` (≤3.4.0, GHSA-528h-pc64-c93x) and `… › jayson › uuid` (<11.1.1, GHSA-w5hq-g745-h8pq). They affect an unused Solana wallet-adapter feature; no fix without unsafe major overrides of `jayson`'s pinned transitives. Re-check when Clerk bumps the chain or a newer cooldown-eligible `@clerk/ui` lands.

### Follow-ups (out of Phase 2 scope)

1. **Resource-based auth migration** for the `createRouteMatcher()` deprecation (Clerk guide: `migrate-from-create-route-matcher`), before the next Clerk major.
2. **Runtime auth smoke test** — requires real Clerk/Convex credentials and a configured instance; this session verified types + build only (no `.env.local` exists).
3. **`@clerk/ui` advisory recheck** after cooldown windows open (see deferrals).
4. Pre-existing repo-wide issues from §5 unchanged (eslint config, tech-debt script, knip config, formatting drift).
