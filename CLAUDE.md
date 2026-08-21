# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project overview

VE-Plan is a virtual event planning platform with two independent apps that are not in the same npm/git workspace — each has its own `package.json`, `node_modules`, and lifecycle:

- `backend/` — Node.js + Express + TypeScript + Mongoose (MongoDB) REST API with Socket.IO for real-time notifications.
- `frontend/` — Angular 22 SPA (standalone components, zoneless change detection) with Angular Material, Bootstrap 5, and FullCalendar.

Two user roles drive almost all functionality: **organizer** (creates events/sessions, manages registrations, invites attendees, hosts meetings) and **attendee** (discovers/registers for events, accepts invitations, joins meetings). Meetings use 8x8.vc (Jitsi-based) video integration.

## Commands

### Backend (`backend/`)

```bash
npm run dev     # tsc --watch + nodemon dist/index.js, concurrently (typical dev loop)
npm start       # one-shot tsc build then nodemon dist/index.js
npm run build   # tsc build + copy email templates into dist/
```

There is no test suite or lint script configured for the backend. There is no dedicated `dev` env — running requires `backend/.env` populated (see Environment below).

### Frontend (`frontend/`)

```bash
ng serve                              # dev server, default (development) config, uses environment.ts (localhost)
npm run start:staging                 # serve against the staging backend (environment.staging.ts)
npm run start:prod                    # serve with production configuration
npm run build:staging / npm run build:prod  # builds (rm -rf dist first)
ng test                               # Karma/Jasmine unit tests, all specs
ng test --include='**/auth.service.spec.ts'   # run a single spec file
```

Nearly every service, guard, and interceptor has a matching `*.spec.ts` — check for one before assuming a unit needs a new test file from scratch.

## Architecture

### Backend: layered, route → controller → service → model

Each domain (Event, Session, User, EventRegister, EventInvite, Meeting, Participant, Notification, Email, Auth) follows the same four-file pattern in `backend/src/{routes,controllers,services,models}`:

- **Routes** (`routes/*.ts`) wire `express-validator` `body()` validation chains and middleware directly inline — there is no separate `validators/` directory. Order matters: multer upload → validation array → `jwtAuth` → role middleware (`organizerAuth`/`attendeeAuth`) → controller handler.
- **Controllers** call `isRequestInvalid(req, res)` (in `helpers/utils.ts`) first to short-circuit on validation errors, then delegate all business/DB logic to the matching service. Every handler wraps its body in try/catch and returns a consistent JSON envelope: `{ status: "success"|"error", message, data? }`.
- **Services** contain the actual Mongoose queries/business logic and are imported as `import * as XService from "../services/XService"`.
- **Models** (`models/*.ts`) are exported via `module.exports = mongoose.model(...)` (CommonJS-style, not `export default`), even though the rest of the codebase uses ES `import`/`export`. Cascading deletes are implemented with Mongoose `pre` hooks (e.g. deleting an `Event` cascades to its Sessions, EventRegisters, EventInvites, Meetings, and Participants — see `models/Event.ts`).

Auth: `passport-jwt` validates the bearer token and attaches the decoded JWT payload as `req.user` (no DB lookup on each request — the JWT itself carries the user doc's fields). `middlewares/jwtAuth.ts` is an array `[passport.authenticate(...), checkUser]` used as a single middleware. `organizerAuth`/`attendeeAuth` check `req.user.role`; `eventOwnerAuth` additionally loads the event by `req.params.id` and rejects (`403`) any organizer who isn't the event's owner — used on `PUT`/`DELETE /events/:id`, after `organizerAuth` in the chain, and attaches the loaded event as `req.event`. Google/Facebook OAuth (`passport-google-oauth20`, `passport-facebook`) issue the same kind of JWT on successful callback.

Runs on Express 5 (built-in `express.json()`/`express.urlencoded()`, no `body-parser`) and Multer 2. A catch-all error-handling middleware (registered after the 404 handler in `index.ts`) turns any thrown/rejected error into `500 { status: "error", message: "Something went wrong." }` instead of leaking it.

Real-time notifications: `libs/socket.ts` authenticates Socket.IO connections via JWT in the handshake, joins each user to a private room (`user_<id>`), and exposes `sendToUser(userId, event, data)` for services (e.g. `NotificationService`) to push events to a specific user.

All routes are mounted under a versioned prefix in `index.ts`: `/api/v1/<resource>` (e.g. `/api/v1/events`, `/api/v1/auth`). Uploaded files (event covers, profile photos) are stored under `dist/photos/` via `multer` and served statically at `/api/v1/static`.

### Frontend: standalone components + role-based lazy-loaded feature routes

The app is fully standalone: `bootstrapApplication(AppComponent, appConfig)` in `main.ts` replaces `AppModule`/`BootstrapModule` entirely — there is no `app.module.ts` or `shared.module.ts`. Every component declares its own `imports: [...]` array of the Material/CDK/CommonModule pieces it needs. `app.config.ts`'s `appConfig` provides the router, `provideZonelessChangeDetection()` (the app runs without `zone.js` in the browser; `zone.js` remains only as a Karma test polyfill), animations, the HTTP client with interceptors, and the native date adapter.

Routing (`app.routes.ts`) splits into:
- Public routes (login, signup, role selection, password reset, email verification, etc.), and
- Two lazy-loaded, role-specific route trees gated by `canMatch: [completeAuthGuard]`: `mod-organizer/` at `organizer/dashboard/**` and `mod-attendee/` at `attendee/dashboard/**`, each exporting a `Routes` array from a `*.routes.ts` file (the old `*-routing.module.ts` + `*.module.ts` pair is gone — there's no module wrapping the routes).

Guards (`guards/`):
- `authGuard` — redirects to `/login` if no JWT in `localStorage` (key `"token"`).
- `hasRoleGuard` — used on the `/role` page; redirects already-role-assigned users straight to their dashboard.
- `completeAuthGuard` — a `CanMatchFn` gating the lazy feature routes: redirects to `/login` if not signed in, to `/role` if the account has no role yet, and to the caller's own `<role>/dashboard/home` if the matched path's role (`organizer`/`attendee`) doesn't match the JWT's role — so a signed-in attendee can no longer reach `organizer/dashboard/**` by URL, and vice versa.

Interceptors (`interceptors/`) are registered via `provideHttpClient(withInterceptors([...]))` in `app.module.ts`: `unauthenticatedInterceptor` (401 handling), `unauthorizedInterceptor` (403 handling), `notFoundInterceptor` (404 handling).

Feature-flagged config lives in `src/environments/environment*.ts` — `apiUrl`, `socketUrl`, static asset URLs (`profileUrl`/`coverUrl`), the 8x8.vc `appId`/`meeting_domain`, and OAuth redirect URLs. `environment.production.ts` is swapped in automatically by the `production` build configuration (see `angular.json` `fileReplacements`).

Services under `services/` are thin HTTP wrappers (one per backend resource, mirroring the backend route names) that return typed `Observable`s using shared response types from `models/Utils.ts`. `caches/` holds services that memoize cross-cutting state used by guards (e.g. `DashboardCacheService.has_role`).

### Environment configuration

Backend expects a `.env` in `backend/` with (non-exhaustive): `DB_URL`, `JWT_SECRET`, `FRONTEND_URL`, `CORS_ORIGIN`, `PORT`, Google/Facebook OAuth credentials (`GOOGLE_CLIENT_ID`/`SECRET`/`CALLBACK_URL`, `FACEBOOK_APP_ID`/`SECRET`/`CALLBACK_URL`), SMTP/SendGrid email credentials (`SENDER`, `SMTP_SERVICE`, `SMTP_USER`, `SENDGRID_API_KEY`, `OAUTH_*`), 8x8.vc Jitsi credentials (`JITSI_APP_ID`, `JITSI_API_KEY`), and per-template HTML file paths (`*_TEMPLATE` vars pointing into `src/email_templates/`).
