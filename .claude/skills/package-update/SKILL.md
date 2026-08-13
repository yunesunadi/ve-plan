---
name: package-update
description: Use when the user asks to update/upgrade/refactor dependencies in this repo — backend (Express/Node/Mongoose) and/or frontend (Angular) — to their latest compatible versions, e.g. "update packages", "upgrade Angular", "bump Express to v5", "package-update", "refactor for the new package versions". Drives an incremental, review-friendly upgrade workflow: research real current docs via context7, plan a dependency graph, then land one small, buildable, testable change at a time instead of one big bump.
---

# Package Update Workflow

Upgrading `backend/` and `frontend/` in VE-Plan is two separate, independently-versioned projects (see root `CLAUDE.md`). Treat this as a multi-session refactor, not a single commit. The user explicitly wants **small reviewable steps**, not a big-bang rewrite — optimize for that over speed.

## Ground rules

- **One coherent change per step.** A step is something like "bump `@types/*` and other low-risk devDependencies" or "migrate Express error handling for v5" or "run `ng update @angular/core @angular/cli`" — never "update everything." Stop after each step, show what changed and why, and let the user review/test before continuing. Use `TaskCreate` to track the step list so progress survives context compaction.
- **Never guess a version or a migration detail.** Before touching a dependency, use the `context7` MCP tools (`mcp__context7__resolve-library-id` then `mcp__context7__query-docs`) to pull the actual current docs/changelog/migration guide for that library. Training data on package versions is stale by definition — this project's own CLAUDE.md was last known-accurate as of Angular 19 / Express 4.21.
- **Re-derive current state from disk, not memory.** Read `backend/package.json` and `frontend/package.json` fresh each session — don't assume the versions quoted in root `CLAUDE.md` are still current once upgrades start landing.
- **Ask before architecture-level decisions**, don't silently decide them. Examples specific to this repo: whether to move Angular to standalone components / signals (CLAUDE.md documents NgModule + `AppModule` as the deliberate current pattern), whether to go zoneless, whether Express 5's built-in async error forwarding should replace the existing per-controller `try/catch` + `{status, message, data}` envelope convention, whether Mongoose model files stay CommonJS-style (`module.exports = mongoose.model(...)`) or move to ES `export default`. Use `AskUserQuestion` for these rather than picking silently — they affect every file in a domain.
- **Respect the repo's existing conventions while upgrading.** Don't use a version bump as cover for unrelated refactors (renaming things, restructuring folders) unless the new major version *requires* it. The layered route→controller→service→model pattern, the JSON response envelope, and the four-file-per-domain structure are intentional (see root `CLAUDE.md`) — preserve them unless the user asks otherwise.
- **Commits/pushes**: only commit when the user asks, one commit per reviewed step, never `--no-verify`, never push without asking — per standing repo/session rules.

## Phase 1 — Discovery

1. Read `backend/package.json` and `frontend/package.json` in full (dependencies + devDependencies + `engines`).
2. Identify the two headline majors the user cares about (Express → latest major, Angular → latest major) plus everything that must move in lockstep with them:
   - Backend: `express` pulls along `@types/express`, possibly `body-parser` (merged into Express 5 core), `express-validator`, `multer`, `passport*` compatibility, Node engine floor.
   - Frontend: `@angular/*` pulls along `@angular/cli`, `@angular-devkit/build-angular`, `zone.js`, `typescript` (Angular pins a TS range), `rxjs`, `@angular/material`, `@angular/cdk`, and third-party Angular libs (`ngx-infinite-scroll`, `@fullcalendar/angular`) which may lag a major behind and gate the upgrade.
3. For each, note current version → candidate target version (don't invent target numbers — confirm via context7 in Phase 2).

## Phase 2 — Research (context7)

For every package being majot-bumped (not for routine patch/minor bumps of leaf deps):

1. `mcp__context7__resolve-library-id` to find the library.
2. `mcp__context7__query-docs` for: breaking changes / migration guide, minimum Node (backend) or TypeScript (frontend) version required, and any deprecated APIs this codebase actually uses (grep the codebase for the API name first so the question is concrete, e.g. "does this app use `app.del()` or `req.param()`" before asking about Express 5's removal of them).
3. Summarize findings back to the user in plain language before writing any code: what breaks, what the migration involves, roughly how large the blast radius is (how many files touch the changed API).

## Phase 3 — Plan

Turn the research into an ordered step list via `TaskCreate`, sequenced low-risk → high-risk:

1. Low-risk devDependency/type-package bumps (safe, mechanical, good warm-up step).
2. Minor/patch bumps of leaf runtime deps with no breaking changes.
3. The headline major bump itself, isolated:
   - Backend: `express` 4→5 (or whatever context7 confirms is current) — this touches routing (path-to-regexp v6 pattern syntax changed), error-handling middleware (Express 5 auto-forwards rejected promises to `next()`), and any removed/renamed APIs found in Phase 2.
   - Frontend: run `ng update @angular/core @angular/cli` (prefer the official schematic over hand-editing `package.json`, since it auto-migrates most breaking changes and codemods templates/decorators) — do this in its own step, then handle anything the schematic flags as manual follow-up as a separate step.
4. Downstream compatibility fixes surfaced by the major bump (type errors, deprecated template syntax, changed Material component APIs, etc.), grouped by module/domain rather than fixed all at once across the whole app.
5. Refactor pass applying the cross-cutting lens below, scoped to what the version bump actually touched — not a full-codebase sweep.

Present the plan to the user before starting execution; confirm before Step 3 (the major bump) specifically, since `ng update` and an Express major both have wide blast radius per this session's risk-confirmation rules.

## Phase 4 — Execute, one step at a time

For each step:

1. Make the change.
2. Verify it builds: backend `npm run build` (tsc compiles clean); frontend `npm run build:dev` and, where relevant, `ng test` for touched specs (this repo has a `*.spec.ts` per service/guard/interceptor — check for an existing one before assuming new test scaffolding is needed).
3. Apply the refactor lens below **only for code the step actually touched** — don't expand scope.
4. If the step introduced a new feature, an intentional architectural decision, or a behavioral change worth documenting (not a mechanical version bump with no user-visible effect), call the SpecMaintainer subagent to sync `PROJECT_SPEC.md`.
5. Report concisely: what changed, why, what was verified, what's left. Stop and wait — don't auto-continue to the next step in the same turn unless the user says to keep going.

## Refactor lens (apply per touched file/domain, not as a blanket pass)

When a version bump forces or enables a code change, weigh it against:

- **Scalability** — does the new API let this scale better (e.g. Express 5 streaming/error handling, Angular deferred loading `@defer` blocks for heavy dashboard views) without over-engineering a change nothing asked for.
- **Reliability/Availability** — does the change preserve existing error-handling guarantees? E.g. if Express 5 auto-forwards async errors, decide deliberately (ask, per Ground rules) whether to simplify the existing manual `try/catch` in controllers or keep it for consistency with the JSON envelope contract.
- **Performance** — bundle size / tree-shaking impact of updated Angular/Material versions, any newly-available build optimizations in the updated `@angular-devkit/build-angular`.
- **Consistency** — new code follows the same patterns as its siblings in the same domain (same four-file layering, same response envelope, same guard/interceptor style).
- **Security** — check context7's changelog for security fixes bundled in the bump; re-check CORS/JWT/passport config against any changed defaults; don't silently loosen validation while migrating `express-validator` chains.

## Notes specific to this repo

- Backend has no lint/test script configured — "verified" here means a clean `tsc` build plus a manual smoke check of the flows the change touches (e.g. auth login, event CRUD, a socket notification) via `npm run dev`.
- Frontend engines pin `node: "^22.x"` in both `package.json`s — confirm the target Express/Angular majors still support that floor before committing to a target version; surface it to the user if not.
- `backend/` and `frontend/` are independent npm projects (no workspace) — never assume a single `npm install` at the repo root affects both.
