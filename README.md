# VE-Plan

VE-Plan is a virtual event planning platform. **Organizers** create events, publish session
agendas, approve or invite attendees, and host video meetings; **attendees** discover events,
register or accept invitations, and join those meetings. Video is provided by 8x8.vc (Jitsi as a
Service).

This repository is the umbrella for the web app's two independently deployed pieces:

| Directory   | What it is | Stack |
| ----------- | ---------- | ----- |
| `backend/`  | REST API + real-time notifications | Node.js 22, Express 5, TypeScript, Mongoose (MongoDB), Socket.IO |
| `frontend/` | Single-page web client | Angular 22 (standalone, zoneless), Angular Material (Material 3), FullCalendar |

`backend/` and `frontend/` are each their own git repository with their own `package.json`,
dependencies, and lifecycle — they are not an npm/git workspace and are git-ignored by this outer
repo, which tracks only the shared documentation.

## Documentation

- **[`PROJECT_SPEC.md`](./PROJECT_SPEC.md)** — behavior specification: what every screen and endpoint
  does, the role/permission model, and the full API reference.
- **[`CLAUDE.md`](./CLAUDE.md)** — architecture and code-level conventions for both apps.
- **`backend/docs/RUNBOOK.md`** — deploy, backup/restore, incident response, and key rotation.

## Features

- **Two roles, one API.** Organizer and attendee accounts, with role enforcement on every endpoint
  and on every route the SPA loads.
- **Events & sessions.** Public or private events with cover images, timezone-aware start/end times,
  a session agenda, and search/filter/calendar views.
- **Registration & invitations.** Attendees register (with organizer approval) or accept direct
  invitations; a single merged "My Events" view tracks participation state.
- **Video meetings.** Organizers host 8x8.vc meetings; attendees join as participants, with
  attendance tracking and a meeting analytics dashboard.
- **Real-time notifications.** Socket.IO pushes notifications live, backed by a durable per-user
  store so nothing is missed across reconnects.
- **Transactional email.** HTML-templated mail (verification, password reset, approvals,
  invitations, meeting notices) sent through Gmail SMTP with a durable queue and retry sweep.
- **Auth.** Email/password with verification, Google and Facebook OAuth, 7-day JWTs with
  server-side revocation (`tokenVersion`), password reset, and rate limiting.
- **Operability.** Structured `pino` logging with request correlation ids, a Prometheus metrics
  endpoint, a security audit log, graceful shutdown, and a hand-rolled DB migration runner.

## Prerequisites

- **Node.js 22** (backend requires `^22.x`; frontend requires `^22.22.3 || ^24.15.0 || ^26.0.0`)
- **MongoDB** — a local instance or a connection string (e.g. MongoDB Atlas)
- Google & Facebook OAuth app credentials
- A Gmail account with SMTP OAuth2 credentials (for outbound email)
- 8x8.vc (JaaS) app id, API key, and RS256 private key (for meetings)

## Getting started

Clone the two app repositories into this folder as `backend/` and `frontend/`, then set each up.

### Backend

```bash
cd backend
npm install
# create backend/.env and fill it in (see "Environment" below)
npm run migrate           # build + apply pending DB migrations
npm run dev               # tsc --watch + nodemon on dist/index.js
```

The API listens on `PORT` (default from `.env`) under the `/api/v1` prefix. Uploaded files are
served from `/api/v1/static`.

### Frontend

```bash
cd frontend
npm install
# edit src/environments/environment.ts so apiUrl / socketUrl point at your backend
npm start                 # ng serve on http://localhost:4200
```

Run both at once (in two terminals) for local development.

## Environment

### Backend (`backend/.env`)

There is no committed `.env` template — create `backend/.env` with the keys below. In production
the server refuses to boot without `DB_URL`, `JWT_SECRET`, `FRONTEND_URL`, `CORS_ORIGIN`,
`PRIVATE_KEY_PATH`, and `METRICS_TOKEN` (and `CORS_ORIGIN` must not be `*`).

| Variable | Notes |
| -------- | ----- |
| `DB_URL` | MongoDB connection string |
| `JWT_SECRET` | Signs auth JWTs |
| `COOKIE_SECRET` | Optional — signs the OAuth-state cookie; falls back to `JWT_SECRET` |
| `PORT` | HTTP port the API listens on |
| `FRONTEND_URL` | Base URL the SPA is served from (used in emails and OAuth redirects) |
| `CORS_ORIGIN` | Allowed origin(s) for REST and Socket.IO, comma-separated; never `*` in production |
| `GOOGLE_CLIENT_ID` / `GOOGLE_CLIENT_SECRET` / `GOOGLE_CALLBACK_URL` | Google OAuth |
| `FACEBOOK_APP_ID` / `FACEBOOK_APP_SECRET` / `FACEBOOK_CALLBACK_URL` | Facebook OAuth (`FACEBOOK_GRAPH_VERSION` optional) |
| `SENDER` / `SMTP_USER` / `OAUTH_CLIENT_ID` / `OAUTH_CLIENT_SECRET` / `OAUTH_REFRESH_TOKEN` / `OAUTH_ACCESS_TOKEN` | Gmail SMTP OAuth2 for outbound email |
| `UNSUBSCRIBE_EMAIL` | Optional — `List-Unsubscribe` address on notification mail; falls back to `SENDER` |
| `JITSI_APP_ID` / `JITSI_API_KEY` / `PRIVATE_KEY_PATH` | 8x8.vc meeting credentials; `PRIVATE_KEY_PATH` points at the RS256 `.pem` |
| `METRICS_TOKEN` | Bearer token guarding `GET /api/v1/metrics` (required in production) |
| `LOG_LEVEL` | Optional — overrides the default (`info` in production, `debug` otherwise) |
| `NODE_ENV` | `production` disables Mongoose `autoIndex` (indexes then come only from migrations) |

### Frontend (`frontend/src/environments/`)

`environment.ts` (development), `environment.staging.ts`, and `environment.production.ts` hold
`apiUrl`, `socketUrl` / `socketPath`, the static asset URLs (`profileUrl` / `coverUrl`), the 8x8.vc
`appId` / `meeting_domain`, and the OAuth entry-point URLs. Angular swaps in the staging/production
file automatically for those build configurations.

## Scripts

### Backend

| Command | Description |
| ------- | ----------- |
| `npm run dev` | `tsc --watch` + `nodemon` (typical dev loop) |
| `npm start` | One-shot build then run |
| `npm run build` | `tsc` build + copy email templates into `dist/` |
| `npm run lint` / `lint:fix` | ESLint over `src/` |
| `npm run migrate` / `migrate:status` | Apply / list DB migrations |
| `npm run pm2:start` / `pm2:reload` / `pm2:restart` / `pm2:stop` / `pm2:logs` | Production process management (`pm2:reload` = zero-downtime) |

### Frontend

| Command | Description |
| ------- | ----------- |
| `npm start` | Dev server (`ng serve`, localhost config) |
| `npm run start:staging` / `start:prod` | Serve against the staging / production backend |
| `npm run build:staging` / `build:prod` | Production-config builds (clears `dist/` first) |
| `npm test` | Karma/Jasmine unit tests |
| `npm run lint` / `lint:fix` | ESLint over `src/` |

## Deployment

The backend runs under **PM2** (`ecosystem.config.js`). A deploy is:

```bash
cd backend
git pull
npm ci
npm run migrate        # apply any pending migrations
npm run pm2:reload     # zero-downtime restart (graceful drain)
```

Several migrations must run on their first deploy (timezone backfill, `tokenVersion` cutover,
notification TTL, audit-log indexes) — see `CLAUDE.md` and `backend/docs/RUNBOOK.md` for details.
The frontend is built with `npm run build:prod` and the contents of `dist/` are served as static
files behind the same reverse proxy that fronts the API (set `client_max_body_size` to at least
6 MB for cover/profile uploads).
