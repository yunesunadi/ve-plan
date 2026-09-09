# VE-Plan — Project Specification

VE-Plan is a virtual event planning platform. Organizers create events, publish session agendas,
approve or invite attendees, and host video meetings; attendees discover events, register or accept
invitations, and join those meetings. Video is provided by 8x8.vc (Jitsi as a Service).

This document describes what the product does today. It is a behavior spec, not a development
guide — see `CLAUDE.md` for build commands and code-level conventions.

The product has two clients on one REST API: a web SPA (Angular) and a native mobile app
(Flutter). The mobile app is **Android only** for now — iOS is planned but not yet set up. This
spec focuses on API and web behavior; where it refers to a "mobile client" or "native mobile app",
that means the Android app.

## 1. Roles and permissions

Every account has exactly one role, chosen once after the account is created and never changed
afterwards. The role determines which dashboard the user lands in and which API calls are permitted.

| Capability | Organizer | Attendee |
| --- | --- | --- |
| Browse public events, calendar, event details, sessions | Yes | Yes |
| See all events (including private ones) on the calendar | Yes | No — public only |
| Create / edit / delete events and sessions | Yes | No |
| See registrants for an event and approve them | Yes | No |
| Search attendees and send event invitations | Yes | No |
| Start and host a meeting, view meeting analytics | Yes | No |
| Notify selected attendees that the meeting has started | Yes | No |
| Register / unregister for an event | No | Yes |
| Accept an invitation | No | Yes |
| Join a meeting as a participant | No | Yes |
| Notifications, profile settings, password change | Yes | Yes |

Role enforcement is server-side: the API rejects a wrong-role request with `403` and the message
`Unauthorized access.`, and the SPA reacts by sending the user back to their own dashboard home.
Requests without a valid bearer token get `401 Unauthenticated access.` and the SPA redirects to
`/login`. The SPA also enforces this before a route loads: the dashboard guard blocks matching an
`organizer/dashboard/**` or `attendee/dashboard/**` path against a JWT of the other role, redirecting
to the signed-in user's own dashboard home instead; a signed-in user with no role yet is sent to
`/role`, and a signed-out user to `/login`.

Beyond role, editing or deleting a specific event is further restricted to the organizer who owns
it: the API loads the event by id (returning `404 There is no event with this ID.` if it is gone),
checks the event's `user` field against the caller, and rejects any other organizer with `403 You are
not the organizer of this event.`

## 2. Accounts and authentication

### 2.1 Sign up with email

- Sign-up collects name, email, password (minimum 8 characters, confirmed twice) and an optional
  profile photo. The same rule guards every password *write* — sign-up, password change, and
  password reset: at least 8 characters and not on a common-password list
  (`400 "This password is too common. Please choose a stronger one."`). It is **not retroactive** —
  login enforces no length or list check, so accounts created under the old 6-character minimum keep
  working until their next password change. The sign-up form shows a live strength hint as the
  password is typed; it is advisory only, and the server's rule is what actually gates the account.
- Duplicate email addresses are rejected with `409`, enforced by the unique index rather than only a
  pre-check, so two concurrent sign-ups for the same address cannot both succeed. The one exception:
  if the address belongs to an account that never verified and whose 24-hour window has lapsed, the
  new sign-up overwrites that abandoned record and a fresh verification link is sent, so an expired
  unverified account no longer squats an address permanently.
- The account is created unverified, and a verification link
  (`<frontend>/verify_email?token=…`) is emailed. The token is valid for 24 hours. The verification
  email is best-effort — a send failure does not fail the sign-up, and `POST /auth/resend_verification`
  (rate-limited, and answering the same way whether or not the address needs verifying) issues a new
  link.
- After submitting, the SPA sends the user to the "verify email" page.

### 2.2 Email verification

Opening the emailed link verifies the account, issues a 7-day JWT, stores it in the browser, and
takes the user to role selection. Clicking the same link again is harmless — an already-verified
account just gets a fresh token (`200`). An unrecognised token is `400`; a token whose 24-hour
window has passed is `410`, which the SPA turns into a "request a new link" prompt.

### 2.3 Login

An unknown email and a wrong password both fail the same way — `401 Email or password is incorrect.`
— so login cannot be used to tell which addresses are registered; the "no such user" path still runs
a throwaway hash so it takes the same time as a real check. A correct password on an unverified
account is a distinct `403` asking the user to verify first (this only appears after the password
matched, so it leaks nothing). A correct password against a social-only account is `400` pointing to
Google/Facebook. On
success the API returns a 7-day JWT whose payload carries only the account id, its role and a
`tokenVersion` counter — no name, email or photo (the SPA reads those from `GET /user`). Every
authenticated request re-loads the account from the database and rejects the token with
`401 Unauthenticated access.` if the account is gone or `401 Your session has expired. Please sign
in again.` if the token's `tokenVersion` no longer matches the stored one, so a password change,
reset or account deletion invalidates every previously-issued token immediately. The user is routed
straight to `organizer/dashboard/home` or `attendee/dashboard/home` based on the role in that token.

### 2.4 Google and Facebook sign-in

Both providers are offered on the login screen and follow the same shape:

- If an account already exists with the same email address, the provider ID is linked to it only
  when ownership of the address is not in doubt: a Google sign-in links when Google reports the
  email as verified; a Facebook sign-in links only to an account that has no password of its own
  (Facebook gives no "verified" signal). A refused link returns `409` telling the user to sign in
  with their password and link the provider from settings.
- Otherwise a new account is created, already marked verified, with the provider's photo URL stored
  as the profile picture.
- The OAuth entry point mints a random nonce, stores it in a short-lived (10-minute) signed
  `HttpOnly` `SameSite=Lax` cookie, and carries it together with the `client` (`web`/`mobile`) hint
  through the provider `state` as base64url-JSON. On the callback the API rejects the response with
  `400 Invalid OAuth state.` unless the returned nonce matches the cookie (the standard OAuth CSRF
  defence), then redirects — web clients to `<frontend>/social_login_redirect?token=…`, mobile
  clients to the `veplanauth://oauth?token=…` deep link — with `Referrer-Policy: no-referrer` set so
  the token is not leaked onward through the redirect. The SPA stores the token, strips it from the
  URL, and goes to role selection, which immediately forwards users who already have a role to their
  dashboard.
- Native mobile apps can also skip the browser redirect entirely: `POST /auth/facebook/token` takes a
  Facebook access token, verifies it against the Graph API (`debug_token` + `/me`), upserts the
  matching account, and returns the same 7-day JWT. A Facebook account with no email address is
  rejected with `400`.

Profile pictures coming from a provider are absolute URLs and are rendered as-is; uploaded pictures
are served from the API's static photo directory. Users with no picture get a placeholder image.

### 2.5 Role selection

Role selection offers "organizer" and "attendee" — any other value is rejected with `400`. The two
roles are presented as pickable cards, the copy states that the choice is permanent, and confirming
goes through a dialog rather than a plain submit. Setting a
role returns a fresh 7-day JWT that carries it and triggers the welcome notification (section 7). The write
is atomic and conditional on the role not already being set, so a duplicate or concurrent call is
rejected with `409 User role is already set.`

### 2.6 Password reset

A reset request always returns `200` with the same "if an account exists, a link has been sent"
message, whether the address is unknown, social-only, or real — no enumeration. When it is a real
password account, a reset link (`<frontend>/reset_password?token=…`) valid for one hour is emailed
(best-effort), replacing any previous outstanding token. Requesting a new one is rate-limited per
address and per IP.

Resetting requires a password that meets the §2.1 rule (at least 8 characters, not common) and is
not the account's current password (`400` if it is). An unrecognised token is `400`; an expired one is `410`. A successful reset proves
the person controls the address, so it also marks the account verified and clears its verification
token. A successful reset advances the account's `tokenVersion`, so every previously-issued JWT is
invalidated and any live socket is dropped — the person logs in fresh. The reset step still rejects a
social-only account with `400 This account uses social login and has no password.`

### 2.7 Profile settings

The settings page (available to both roles) is organised as Profile, Security and Danger zone, and
allows:

- Editing name (1–80 characters) and profile photo. Email is shown read-only and cannot be changed —
  it is the login identifier and account-recovery channel, and for social accounts it mirrors the
  OAuth provider. The `PUT /user` endpoint accepts only `name` and the photo. A chosen photo is
  checked in the browser for type and size before upload (JPEG/PNG/WebP, up to 5 MB — the same rule
  the API enforces), shown as a preview before saving, and its upload reports progress.
- Changing the password, which requires the current password to be supplied and correct and rejects
  reuse of that same password. `GET /user` returns a `hasPassword` flag; for a social-login account
  with no password of its own the settings page replaces the form with a note saying which provider
  the account signs in with and where to manage it, and
  `PUT /user/password` independently rejects them with `400 This account uses social login and has
  no password.` A successful change advances the account's `tokenVersion` (invalidating every other
  session) and drops its sockets, and returns a fresh `token` in the response body so the caller's
  own session survives its own change — the SPA swaps the stored token for it.
- **Deleting the account** (`DELETE /user`, rate-limited). A password account must re-enter its
  password; a social-only account must type its email address to confirm. An organizer is told up
  front how many of their events will be removed, and a live meeting they own is surfaced as a
  persistent warning with a link to that meeting, which stops the submit before it is sent. Deletion
  is refused with
  `409` while the caller owns an event whose meeting is live. It erases the user's owned events (with
  the usual cascade to their sessions, registrations, invitations, meetings and participants), the
  user's own registrations / invitations / participant rows / notifications, and the profile photo,
  then removes the account and disconnects its sockets. A second call is `401` — the deleted
  account's JWT is now actively rejected on the next request (the per-request account lookup finds
  nothing) as well as on the socket.

## 3. Events

### 3.1 What an event holds

An event has a cover image (optional), title, description, date, start time, end time, category,
visibility type, and an owning organizer. Category is one of `conference`, `meetup`, `webinar` and
type is `public` or `private` — the API rejects any other value (`400`), and the schema enforces the
same set. Title is 3–140 characters, description at most 5000; date and times must be valid ISO
timestamps.

An event also carries an IANA `timezone` (e.g. `Asia/Bangkok`). Clients submit it on create and
edit (the browser/device zone by default); the API falls back to the server's zone when it is absent
or invalid. From the calendar day, the start/end time-of-day and the timezone the server derives two
absolute UTC instants, `starts_at` and `ends_at`, and stores them on the event on every write.
Pre-existing events were backfilled with the server's zone and the matching instants by migration
`003-event-timezone-backfill`. These instants are being adopted as the single source of truth for
"has this event started / ended" and for time filtering and sorting, replacing the older per-reader
local-timezone recomposition.

### 3.2 Creating and editing

- Organizers create events either from the **Create event** action on their home dashboard or by
  clicking a day on the calendar, which pre-fills that date. Both open the same form, which carries
  an editable date field of its own.
- An event that has already ended cannot be created — the API rejects it with
  `409 Can't create an event that has already ended.`, comparing the derived `ends_at` instant
  against now. The form's date field independently refuses a date in the past and says so inline, so
  clicking a past day on the calendar opens the form with the problem already flagged rather than
  silently doing nothing. An event already under way can still be created.
- The event form requires title, description, date, start time, end time, category and type. The API
  rejects an end time that is not after the start time (`400`), independently of the client check.
- An edit that would push the event's end instant into the past is refused with `409` while a meeting
  for that event is still live, so attendees are not dropped from a room the app then treats as
  ended.
- Editing uses the same form, pre-filled. Uploading a new cover replaces the old one; leaving it
  untouched keeps the existing cover.

### 3.3 Deleting

Deleting an event asks for confirmation and warns that everything attached to it is removed. The API
cascades the delete to the event's sessions, registrations, invitations, meeting and participant
records. Deletion is refused with `409 "End the meeting before deleting this event."` while a
non-ended meeting exists for the event, so a live Jitsi room is never destroyed out from under its
participants. The cascade legs run independently (a failed leg does not abort the others or the
delete itself); a periodic reconciliation sweep removes any child rows left pointing at an event
that no longer exists.

### 3.4 Visibility and discovery

- **Home** (both roles): the role's dashboard and the landing screen after sign-in. The organizer
  overview is one call (`GET /events/organizer_summary`) and shows a Create-event action, a banner
  when one of their meetings is live, counts of upcoming events / registrations awaiting approval /
  pending invitations, the next upcoming event, and a recent-activity list (registrations, accepted
  invitations and started meetings across their events). The attendee feed
  (`GET /events/attendee_summary`) shows Happening now, Starting soon (within the next 24 hours),
  Invitations awaiting a response — with an inline Accept — and Recommended for you, upcoming public
  events the attendee is not already part of. Empty sections are hidden; a completely empty feed
  shows one empty state with a Browse-events call to action.
- **Events list** (both roles): shows public events only, regardless of role. Filters are keyword
  search on title, time bucket (upcoming / happening / past), category and calendar day; the time
  bucket and day filters run on the event's derived `starts_at` / `ends_at` instants (the day
  resolved in the server's default timezone), so "upcoming" keeps an event happening later today.
  The filter fields are always visible, each active filter also appears as a removable chip, and a
  "Clear all" action shows once at least one is set. Results are paged 5 at a time, and the filter
  state lives in the URL query string.
- **Calendar** (both roles, its own destination): a month view of events. Organizers see every
  event; attendees see public events only. Clicking an event opens a read-only details dialog, and a
  legend distinguishes public from private ones. Below the tablet breakpoint the month grid is
  replaced by a list of the month's events; at wider widths the user can switch between the two.
- **My Events** (organizer): the organizer's own events, filterable by all / public / private,
  loaded 5 at a time as the list is scrolled. Scroll position is restored when navigating back, and
  a failed "load more" keeps the rows already on screen and offers a retry. Attendees have their own
  My Events — the merged membership list described in section 5.

Private events do not appear in the events list; attendees reach them through an invitation, which
links directly to the event's detail page. Fetching one event by id enforces this too: a private
event is returned only to its owner or to an attendee who holds an invitation or a registration for
it — everyone else gets `404`, the same response as for an id that does not exist.

### 3.5 Event details page

Event detail is a persistent header plus tabs, and the tab is part of the URL
(`…/events/:id/view/overview`, `…/agenda`, and for organizers `…/people`) so a tab can be linked to
and survives a reload. The header carries the event title, breadcrumbs and the role's actions;
switching tabs does not re-fetch the event, and an edit made from the header updates every tab in
place.

- **Organizer**: Edit and Delete in the header — only for the organizer who owns the event — and the
  tabs **Overview** (the event card), **Agenda** (the sessions list with add / edit / delete) and
  **People** (links to Registered Users and Invite Users). A fourth tab-strip entry, **Meeting**,
  links out to the event's meeting page.
- **Attendee**: the tabs **Overview** and **Agenda** (read-only), plus a participation panel pinned
  above them so it is visible on every tab. That panel shows the registration/invitation state as a
  status chip with a short explanation and the Register / Cancel registration / Join Meeting actions
  described in sections 4–7.

The event card renders the date and times in the event's own timezone and names that zone, and falls
back to a category icon when the event has no cover image.

For the attendee, that state is not derived client-side from a fan-out of calls: `GET /events/:id`
returns a single server-computed `participation` object in `data` —
`{ state: "none" | "registered" | "registration_approved" | "invited" | "invitation_accepted",
meeting_started }`. An invitation outranks a registration for display (an attendee holding both sees
the invitation state); `meeting_started` is orthogonal (a Join Meeting control can appear alongside
any state) and is true once the meeting is **live** — the host is present in the room and it has not
ended (§7.3), regardless of whether this attendee was individually notified. `participation` is
`null` for organizers.

## 4. Sessions

Sessions are the agenda entries inside an event: title (3–140 characters), description, speaker info,
start time and end time. The API rejects an end time that is not after the start time, and a session
whose times fall outside its event's start/end window (`400`); overlapping sessions are allowed
(parallel tracks). Only organizers can add, edit or delete them, and both the controls and the API
enforce this: creating a session requires the caller to own the event named in the body, and editing
or deleting one is checked against the owner of the session's parent event — anyone else gets `403`,
and an unknown session id `404`. An edit cannot move a session to a different event; only the title,
description, speaker info and times are writable.

Any user who can see the event can read its sessions, ordered by start time — `GET /events/:id/sessions`
is the canonical route (the older `GET /sessions` with an `event-id` header is kept as an alias). A
private event's sessions are visible only to its owner and to attendees with a registration or
invitation, the same rule as the event itself.

## 5. Registration flow (attendee-initiated)

1. On a public event, an attendee clicks **Register**. This creates a registration in the pending
   state; a repeat click while already registered is idempotent (no second row). The button then
   reads "Registered" and clicking it again unregisters, after a confirmation prompt — unregistering
   when there is nothing to remove returns `404`. A **private** event can only be registered for by
   an attendee who currently holds an invitation for it; anyone else gets
   `403 "This is a private event. You need an invitation to register."`
2. Register and unregister are disabled in the UI once the event's date and end time have passed, and
   the API independently rejects a late registration with `400 This event has already ended and can no
   longer be registered for.` Unregistering is also refused with `409` while the event's meeting is
   live, so a participant record cannot be orphaned mid-call. When an **approved** attendee
   unregisters (once the meeting has ended), the owning organizer receives a best-effort in-app
   notification; unregistering before approval stays silent.
3. The organizer opens **Registered Users** for the event: a searchable, paginated table (10 per
   page) of registrants with their approval status. Already-approved rows have no checkbox, and the
   Send Approval action is disabled once the event has ended. The registrant-list endpoints, like
   the approval endpoint, are restricted to the organizer who owns the event (`403` otherwise).
4. Selecting registrants and confirming sends each of them a "Registration Approved" email and an
   in-app notification, and marks their registration approved. The server first narrows the request
   to users who actually hold a *pending* registration for that event, so emails and notifications
   never reach anyone else; every other selection (already approved, or never registered) is
   **skipped, not re-processed**. The request succeeds with `200` and a
   `data: { approved: [...], skipped: [...] }` body listing which registrations were newly approved
   and which were left untouched, so a repeat approval is idempotent. The dialog reports that
   outcome per row — approved, or already approved — rather than a single summary message.
5. On the attendee side the event page then shows "Register Approved" instead of the register
   button, and the event moves from the "Registered" segment to "Approved" in My Events.

**My Events** (attendee) is one screen listing everything the attendee is part of, segmented into
All / Invited / Registered (still awaiting approval) / Approved / Attending (invitation accepted).
The segment lives in the URL, rows carry the server-computed state as a status chip, an invited row
offers Accept inline, and the list loads a page at a time behind a "Load more" action. An attendee
holding both an invitation and a registration for the same event is listed once, under the
invitation state.

## 6. Invitation flow (organizer-initiated)

1. The organizer opens **Invite Users** for an event and types a keyword; the search returns
   attendee-role users matching the keyword by name or email (debounced, 500 ms). The keyword is
   matched as a literal (regex metacharacters are escaped), fewer than two characters returns nothing
   rather than matching everyone, results are capped at 50 per page and sorted by name, and each hit
   carries only the id, name, profile photo and a **masked** email — never the full address or any
   account token.
2. The organizer selects users and confirms. Only the organizer who owns the event may invite to it
   (`403` otherwise). Inviting to an event that has already ended is rejected with `400 This event
   has already ended and can no longer be invited to.` The selection is validated: 1–200 entries,
   each a valid id resolving to a **verified attendee** account — any unknown id or non-attendee is
   rejected with `400` naming the offenders (an organizer cannot be invited). Already-invited
   selections are **skipped, not rejected**: the request succeeds with `200` and a
   `data: { invited: [...], skipped: [...] }` body listing which selections were newly invited and
   which already had an invitation; the dialog reports that outcome per row rather than as a single
   summary message. The invite write upserts per `(event, user)`, so a concurrent double-submit
   cannot create duplicate invitations.
3. Each newly-invited attendee receives an "Event Invitation" email and an in-app notification, and
   an invitation record is created in the pending state.
4. The organizer can review **Invited Users** and **Accepted Users** for the event in dialogs — both
   lists are restricted to the event's owner.
5. The attendee finds pending invitations in two places: the **Invited** segment of My Events, and
   an "Invitations awaiting your response" section on their home feed. Both link to the event
   details and offer **Accept** inline. Accepting marks the invitation accepted; the event then
   moves to the "Attending" segment and the event page shows the "Invitation Accepted" state.
   Accepting when no invitation exists (revoked, or never sent) returns `404`; accepting an
   invitation to an event that has already ended returns `400`; re-accepting an already-accepted
   invitation is an idempotent `200`.

Once an event has ended, an invitation alone no longer grants read access to a private event (a
registration still does), so a stale invite is not a permanent access grant.

An attendee who has been invited to an event sees the invitation status on the event page instead of
the register button — invitation and registration are alternative paths into the same event.

## 7. Meetings (8x8.vc / Jitsi)

### 7.1 Starting and hosting (organizer)

- The event's **Meeting** page leads with the meeting's status — Not started, Scheduled, Live now or
  Ended — plus a one-line explanation, and lays out Status, Meeting controls, Analytics and Joined
  participants in that order. A control that cannot be used right now (Start / Join / End / Reopen)
  is shown disabled with the reason stated — the join window has not opened yet, the join window has
  closed, the event has already ended — rather than silently greyed out. Re-opening an ended meeting
  returns it to Scheduled or Live now; "reopened" is not a distinct status.
- The page shows **Start Meeting** if no meeting exists yet for that event and
  organizer. Starting one stores a meeting record whose room name is generated by the server — the
  request names only the event, and a client-supplied room name or token is rejected. Attempting to
  start a second meeting for the same event is rejected with `409 A meeting already exists for this
  event.`, and an organizer who does not own the event with `403 You are not the organizer of this
  event.`
- Once created, the button becomes **Join Meeting** and opens a full-screen dialog embedding the
  8x8.vc conference for room `<appId>/<room name>`. The dialog asks the API for a token for that
  event as it opens; the server grants moderator rights because the caller owns the event, and
  returns the room name to join alongside the token. The embed enables the prejoin screen, disables
  in-conference invite and kick controls, and starts audio and video muted.
- The 8x8.vc `external_api.js` library (which defines the `JitsiMeetExternalAPI` global) is **not**
  loaded on every page. `MeetingService.loadExternalApi()` injects a single `<script>` the first
  time a meeting dialog opens and memoises the load promise, so other routes (login, dashboards)
  never fetch it and a second meeting reuses the already-injected script. If that script fails to
  load the meeting dialog reports it on an overlay panel with Try again and Close, rather than
  closing itself.
- The organizer joining the room marks the meeting **host-present** and records its start instant.
  Both timestamps are stamped by the server — the start-time and end-time updates take no time in
  their request body. When no start time was ever recorded the duration is left unset rather than
  computed from a missing value.
- Every meeting-record route (`GET /meetings/:id`, the start/end-time updates, end and reopen) is
  scoped to the organizer who owns both the event and its meeting; anyone else gets `404 There is no
  meeting for this event.`
- Meeting tokens are never stored on the meeting record. One is minted each time a join dialog
  opens, is scoped to that meeting's room only, and is valid for two hours. A token request must
  name a well-formed event id (`400` otherwise) that exists (`404 There is no event with this ID.`)
  and already has a meeting (`404 There is no meeting for this event.`). If the API refuses the
  request, the dialog stays open and shows the returned message on an overlay panel over the
  conference frame, with Try again and Close.
- **Join window.** A meeting can be started, a token minted, or a participant recorded only from 15
  minutes before the event's start time until 30 minutes after its end time. Outside that window the
  API answers `403` naming the window; creating a meeting after it is `409 This event has ended, so a
  meeting can no longer be created.` (Creating one earlier is still allowed.)

Once a meeting exists, the meeting page also offers **End Meeting**. Ending it, after a confirmation
prompt, is one server-side transactional call: it flags the meeting ended (recording `ended_at`),
stamps the end time and duration, drops the host-present flag, closes out every participant still
marked present, and sends every attendee who was notified that the meeting started a "Meeting Ended"
email and in-app notification. The organizer dialog shows a spinner and a real error state instead of
reloading the page. While a meeting is ended the page shows "This meeting has ended. Attendees can no
longer join until it is re-opened." and replaces the button with **Reopen**; reopening clears the
ended flag and re-sends the "Meeting Started" email and notification to the same attendees. End and
Reopen are idempotent — the flag is flipped only if it is not already in the requested state, so a
repeated or cross-tab call succeeds without sending the emails and notifications a second time.

### 7.2 Notifying attendees

From the meeting page, **View Attendees** lists the event's approved registrants and accepted
invitees, deduplicated per user, paginated 10 per page and searchable. Selecting users and confirming
sends each a "Meeting Started" email plus an in-app notification and records that they were notified.
Already-notified rows have no checkbox, and the send action is disabled once the event has ended.
Only the event's own organizer can send it — the notify endpoints reject a non-owner with `403 You
are not the organizer of this event.` and require a non-empty list of user ids and a well-formed
event id. Notifying is a courtesy ping and a record of who was told — it does **not** gate joining
(see 7.3): a meeting is joinable once the host is present in the room, for every approved
participant.

### 7.3 Joining (attendee)

- The **Join Meeting** button appears on the attendee's event page only when the meeting is **live** —
  the host is present in the room and it has not been ended (`participation.meeting_started` on
  `GET /events/:id`). It is disabled after the event has ended. If the meeting has already been ended
  by the time the dialog opens, the dialog says so on its own panel and offers Close.
- The dialog first asks the API for the meeting's room name, ended flag and start time — only those
  three fields, never the meeting record — and then mints a token for that room. The token is
  non-moderator, because the attendee does not own the event, so the organizer remains the only
  moderator.
- The API issues that token to any participant of the event — an approved registrant or an accepted
  invitee — while the meeting is live, answering `403 You are not a participant of this event.` or
  `403 This meeting has ended.` otherwise. It also refuses a join before the organizer is actually in
  the room (`409 The host hasn't started this meeting yet.`), outside the join window (`403`), or
  once the meeting is at capacity — 25 concurrent participants (`409 This meeting is full.`). Whether
  the attendee was individually "notified" (7.2) does not affect this. The dialog surfaces whichever
  message came back on its overlay panel, with Try again and Close, so a "host hasn't started yet" or
  "meeting is full" can simply be retried.
- Joining creates the attendee's participant record; leaving adds that session's minutes to the stay
  duration. Both the join and leave instants are stamped by the server — neither call takes a
  timestamp from the client. There is one participant record per attendee per event (enforced by a
  unique index): rejoining reopens the same record and clears its end time, and the duration
  accumulates across sessions rather than being overwritten. A duplicate "leave" is a no-op. The join
  call is accepted only from an approved registrant or accepted invitee of an event that has a live
  (not ended) meeting; the room name is set by the server, not the client.
- The organizer's participant list and stay-time breakdown for an event, and the "close out everyone
  still present" action, are all restricted to the event's owner. A background sweep also closes out
  participants left marked present on meetings that have already ended, capping the session at the
  meeting's `ended_at`.

### 7.4 Meeting analytics (organizer)

The meeting page summarizes attendance for the organizer:

- Counter cards: event attendees, joined participants, approved registrants, invitation-accepted
  users.
- A doughnut chart comparing event attendees against participants who actually joined.
- A line chart of stay-time distribution, bucketed into four ranges sized from the meeting's total
  duration (or, when that is unknown, the longest recorded stay); the last bucket is open-ended so
  long stays are never dropped. Before the meeting ends, or when no stay times were recorded, the
  chart is replaced by an empty state ("No stay-time data yet") and the endpoint returns `data: []`
  with a `meta.available` flag.
- A paginated table of joined participants with join time, leave time and stay time in minutes.

Until the meeting has run at all, the whole analytics block is a single "Meeting not started" empty
state. Each chart is mirrored by a visually-hidden table carrying the same numbers, so the figures
are readable without seeing the canvas.

## 8. Notifications

Notifications are stored per recipient and also pushed live over a Socket.IO connection that is
authenticated with the same JWT; each user is placed in a private room so notifications are only
delivered to their own sessions. Each client opens a single connection for the whole signed-in
session — owned by the app shell (web) and by a session listener (mobile), not by the notifications
screen — and closes it on logout. The handshake re-runs on every connection (including reconnects):
it re-loads the account and rejects the connection if the account is gone or the token's
`tokenVersion` no longer matches, so a socket cannot outlive a logout, password change or account
deletion; after a password change the client reconnects with the fresh token. Because a live emit
to an offline socket is lost, the clients treat the stored notifications as the source of truth and
the socket only as an accelerator: on every connect and reconnect they refetch the list and the
unread count so nothing that arrived during a drop is missed, and merge live events by id so a
notification is never shown twice. The connection CORS origin is restricted to the configured
origins (never `*`), a single account may hold at most five concurrent sockets (the oldest is
dropped when a sixth connects), and every connect and disconnect is logged with the user and socket
id.

| Trigger | Recipients | Title |
| --- | --- | --- |
| Role chosen for the first time | The new user | Welcome to VE-Plan! |
| Public event created | All verified users | Event Created |
| Public event updated (title/date/time changed) | All verified users | Event Updated |
| Private event updated (title/date/time changed) | Its registered and invited attendees | Event Updated |
| Registration approved | The approved attendees | Registration Approved |
| Invitation sent | The invited attendees | Invitation |
| Meeting started notice sent | The selected attendees | Meeting Started |
| Meeting ended (or reopened) | The attendees already notified for that meeting | Meeting Ended / Meeting Started |
| Approved attendee unregisters | The event's owning organizer | Attendee left |

An event edit only notifies when a field attendees care about — title, date, start or end time —
actually changed; a description-only edit is silent. The public "event updated" broadcast is
debounced to at most once per hour per event. Every event-related notification carries a reference
to its event (`sender`), which the clients use to deep-link. The broadcast fan-out is done in
batches over an id-only projection rather than loading every user document.

Notifications are stored per recipient and also pushed live over a Socket.IO connection; the stored
copy is the source of truth and the socket is an accelerator.

`GET /notifications` is paginated (`?offset=&limit=`, default 20, max 50) and returns
`meta: { total, unread, offset, limit }`. The toolbar shows a bell with an unread badge.
`POST /notifications/mark_all_read` clears the badge in one call; `POST /notifications/mark_as_read`
takes a capped array of ids and only touches rows that are still unread. Deleting takes the id array
in the request body (a JSON string in the `notification_id_list` query param is still accepted for
older clients). Notification history is bounded server-side: rows are removed 90 days after creation
by a TTL index (both clients page the recent set newest-first and never read further back).

The notifications screen groups rows by day (Today, Yesterday, then the date) and offers an
All / Unread filter over the loaded list, with more pages fetched as the user scrolls. Unread rows
are marked by an accent bar, a dot and a "New" label rather than by colour alone. Opening a
notification marks it read before navigating to its event, and it is a real link, so the usual
open-in-new-tab gestures work. Selecting rows raises a bar offering Mark read, Delete and Clear
selection; Mark all read stays in the header and is disabled when nothing is unread.

## 9. Transactional email

Emails are rendered from HTML templates named after the action (`src/email_templates/<action>.html`),
with `{{name}}`, `{{event_title}}` and `{{link}}` placeholders substituted in; `name` and
`event_title` are HTML-escaped since they are user-controlled, while `link` is a URL the server
builds. A missing or unreadable template is an error, not a silently blank email, and all six
templates are checked at boot (fatal in production). The event and meeting notification mails
(registration approved, invitation sent, meeting started, meeting ended) carry a `mailto:`
`List-Unsubscribe` header (RFC 2369) pointing at `UNSUBSCRIBE_EMAIL` (or the sender address when
that is unset); the purely transactional mails — email verification and password reset — do not,
since there is nothing to unsubscribe from.

Delivery goes over Gmail SMTP with OAuth2 through a single pooled connection. `EmailLog` is a
durable queue: each row carries everything needed to (re)send itself — the action, the rendered
input vars (`meta`), the `from` address, and (for organizer fan-out actions) the `event` it belongs
to — and moves `pending` → `sent` / `failed`, tracking a cumulative attempt count, the last error,
and whether the last failure is `retryable`. On insert an eager background send fires immediately
(the HTTP response is not held for it) and retries up to three times with exponential backoff on
transient errors (timeouts, connection resets, expired OAuth token); a hard SMTP rejection is not
retried and is marked non-retryable. A periodic retry sweep (every 5 minutes) recovers stragglers:
rows stuck `pending` past 10 minutes (restart orphans) and `retryable` failures that have cooled
down, are under the cumulative-attempt ceiling, and are less than 24 hours old — after that the
organizer owns the failure via the status endpoint below. Rows are retained 30 days (TTL index),
which bounds the collection now that rows hold rendered-input PII.

For the organizer actions that fan out to a list of attendees — approving registrants, sending
invitations, notifying that a meeting started — the state change is written first, then the
recipients are resolved in one query and the whole batch is queued with a single `EmailLog`
`insertMany`, each row tagged with the `event` id. A delivery failure never fails the request or
rolls back the state change. The `PUT /event_registers/approve` and `POST /event_invites`
responses carry `data.email = { queued: <n> }` (the number of rows queued); the meeting-started
notify responses stay bare. The organizer can poll `GET /events/:id/email_status` for the
event-level rollup (`{ sent, pending, failed, retryableFailed }`) and trigger
`POST /events/:id/email_retry` to flip that event's retryable failures back to `pending` and
re-deliver them (`data.requeued = <n>`). Both accept an optional `action`
(`register_approved` | `invitation_sent` | `meeting_started`) to scope the rollup / retry to a
single email type — the `<app-email-delivery-status>` widget passes it so the Registered Users,
Invite Users, and Event Attendees screens each show only the delivery status for the mail that
screen sends.

The six templates in use are:

| Action | Subject | Sent when |
| --- | --- | --- |
| `email_verified` | Email Verification | A new account is registered |
| `reset_password` | Password Reset | A password reset is requested |
| `register_approved` | Registration Approved | An organizer approves registrants |
| `invitation_sent` | Event Invitation | An organizer invites attendees |
| `meeting_started` | Meeting Started | An organizer notifies attendees a meeting began, or reopens an ended meeting |
| `meeting_ended` | Meeting Ended | An organizer ends a meeting |

## 10. Application structure and access control

### 10.1 Navigation

Both dashboards share a toolbar (notifications bell with an unread count, and a profile menu with
Appearance and Logout) and the same four primary destinations for both roles: **Home, Events, My
Events, Calendar** — Home being the role's dashboard, not the calendar. Settings, Terms and
Conditions and Privacy Policy sit in a secondary group at the bottom of the navigation drawer. The
shell is responsive across one codebase rather than a separate mobile layout: a bottom navigation
bar below the tablet breakpoint, a collapsed icon-only rail at tablet width, and a fully labelled
side drawer at desktop width and up; the drawer's expanded/collapsed preference persists across
sessions on the widths where the user has a choice. Logging out asks for confirmation via the same
themed confirmation dialog used elsewhere in the app (see 10.5), then clears the stored token and
returns to login.

### 10.2 Routes

Public: `/login`, `/signup`, `/role`, `/verify_email`, `/forgot_password`, `/reset_password`,
`/social_login_redirect`, `/terms_and_conditions`, `/privacy_policy`, `/error`. Unknown paths
outside a dashboard fall back to login.

Organizer dashboard (`/organizer/dashboard/…`): `home`, `events`, `my_events`, `calendar`,
`events/:id/view` (with the child tabs `view/overview`, `view/agenda`, `view/people`; bare `view`
redirects to `view/overview`), `events/:id/registered_users`, `events/:id/invite`,
`events/:id/meeting`, `events/:id/meeting/attendees`, `setting`, `notifications`, plus a not-found
page.

Attendee dashboard (`/attendee/dashboard/…`): `home`, `events`, `my_events`, `calendar`,
`events/:id/view` (with the child tabs `view/overview` and `view/agenda`), `setting`,
`notifications`, plus a not-found page. The former `invitations` and `joined_events` routes still
resolve — they redirect into `my_events?filter=invited` / `?filter=registered` — so old links and
bookmarks keep working.

### 10.3 Client-side handling of API failures

On the server, the bearer token is first verified for signature and expiry, then every authenticated
request loads the caller's account by id (an indexed lookup) to confirm it still exists and that the
token's `tokenVersion` still matches the stored one; `role`, `name` and `email` used downstream are
taken from that fresh record rather than from the token. This is the trade for the slim, PII-free
token and for immediate revocation on password change, reset and account deletion.

The SPA keeps its JWT in browser local storage and attaches it as a bearer token. Global HTTP error
handling routes the user according to the failure: a `401` clears the stored token and returns to
login (skipped if there was no token to begin with — e.g. a public-page call). A `403`/`404` never
redirects the user away from their current page — a background request failing shouldn't throw
someone off a screen they're actively using; the screen instead shows an inline error state with a
retry action. The event-scoped organizer/attendee screens (registered-users, invite, event-attendees,
meeting, event-view) resolve their event before the route activates: a missing event sends the user
to the dashboard's not-found page as part of navigation itself, and any other resolution failure
surfaces as an error toast without leaving a half-rendered screen behind. The `/role` page
additionally redirects users who are not signed in to login, and users who already have a role
straight to their dashboard.

Errors the SPA cannot handle at all — a component that throws, an unexpected runtime failure — reach
an app-wide error handler. It always logs to the browser console, and — de-duplicated over a
five-second window and throttled to five reports a minute — posts the message, stack, page URL and
user agent to the API's client-error sink, so the failure lands in the server logs (with the user's
id attached when a token was present). Errors that already came from the API are not re-reported,
since the server logged them itself. When the backend answers `500` it returns a request id, and the
error toast quotes it — `… (ref: <request id>)` — so a user can hand it to support and it can be
matched to the exact server log line.

### 10.4 Uploads

Cover images and profile photos are uploaded as multipart form data and served back as static files
under the API's `/static/covers` and `/static/profiles` paths (with `X-Content-Type-Options:
nosniff` and `Cross-Origin-Resource-Policy: cross-origin`, so the SPA and the mobile app can load
them from another origin). An upload must be a JPEG, PNG, or WebP of at most 5 MB — the declared
content type is checked against the file's magic bytes, and a mismatch is `415`, an oversize file
`413` (returned as the normal JSON envelope by a dedicated upload-error middleware rather than
bubbling to the generic `500`; a reverse proxy in front of the API must be configured to allow a
body at least that large, or it rejects the request with its own HTML `413` first). Replacing a
cover or profile photo deletes the previous file, and deleting an event deletes its cover.

### 10.5 Shared UI conventions

The frontend draws every screen from one small shared component vocabulary (`src/app/shared/ui/`)
instead of each screen inventing its own presentation, so the same state always looks and reads the
same way everywhere it appears:

- **Status labels.** Every status a user sees anywhere in the app comes from one fixed vocabulary,
  always shown as an icon plus text (never colour alone): Registered, Approved, Invited, Accepted,
  Live now, Ended, Public, Private, Upcoming, Happening now, Past.
- **Loading, empty, and error states.** A list or detail screen that is loading shows a placeholder
  shaped like the content about to arrive (not a blank screen or a generic spinner); a list with
  nothing in it shows an icon, a short headline, supporting copy, and — where there's a next step —
  a call to action (e.g. Notifications' "You're all caught up" when there's nothing new); a failed
  request shows what went wrong and a "Try again" action, except when retrying can't help (an
  access-denied or not-found response), where the retry action is hidden instead.
- **Toasts.** Background actions confirm or fail via a toast in one of four severities — success,
  info, warning, error — each with its own colour, icon, and dismiss timing; an error toast never
  auto-dismisses, so the user has time to read it.
- **Confirmation.** Every destructive or state-changing action (logging out, deleting an event,
  ending a meeting) confirms through the same themed dialog rather than the browser's native
  confirm/alert prompt, and a destructive action that can't be undone (e.g. deleting an event)
  additionally requires typing the item's name before the confirm button enables.
- **Tables and bulk actions.** The paginated people lists — registrants, invitees, meeting attendees
  — share one table component: rows are reachable and moved between with the arrow keys (plus Home
  and End), Enter or Space toggles selection, and once anything is selected the bulk-action bar
  sticks to the top of the list while scrolling. Below the tablet breakpoint the same rows render as
  cards instead of table columns.
- **Appearance.** The app follows the operating system's light/dark setting by default. The profile
  menu's **Appearance** submenu pins it to Light or Dark, or returns it to System, and that choice
  persists on the browser. The chosen scheme is applied before the first paint, so the app never
  flashes the wrong one on load.

## 11. API surface

All endpoints are mounted under `/api/v1`. "Any" means any authenticated user regardless of role.
"Organizer (owner)" / "Organizer (event owner)" means the request is additionally checked against
the event's owning organizer, rejecting any other organizer with `403` (or `404` where confirming
existence would itself leak information). Every handler responds `500 { status: "error", message:
"Something went wrong." }` on an unexpected error — the underlying error object is logged server-side
and never serialised into the response.

Paged list endpoints (`GET /events/events_by_query`, `GET /events/own`, `GET /events/my`,
`GET /user/attendees`, `GET /event_registers/:id/users`, `GET /event_registers/events`,
`GET /event_registers/events/approved`, `GET /event_invites/events`,
`GET /event_invites/accepted_events`, `GET /participants/:id`, `GET /notifications`) accept
`?offset=&limit=` and return `data: [...]` alongside `meta: { total, offset, limit }` (the
notifications endpoint additionally reports `unread`). `offset` and `limit` are clamped, never
rejected: `offset` floors to 0, `limit` is bounded to 1–50 and falls back to the endpoint's
default when absent or non-numeric. The attendee's own registration / invitation lists are sorted
newest-first and omit rows whose event has been deleted.

### Platform

| Endpoint | Access | Purpose |
| --- | --- | --- |
| `GET /health` | Public | Liveness — process uptime and MongoDB connection state (`200` connected, `503` otherwise) |
| `GET /metrics` | Operator (bearer `METRICS_TOKEN`) | Prometheus-format metrics — `503` if no token is configured, `401` if the presented one is wrong |
| `POST /client_errors` | Public (rate-limited, optional auth) | Sink for unhandled web-client errors; logs one structured line and returns `204` — nothing is stored |

`GET /metrics` is not part of the product API: it is for whoever operates the deployment. The token
is compared in constant time, and the endpoint is unavailable rather than open when `METRICS_TOKEN`
is unset. `POST /client_errors` accepts `message` (required) plus optional `stack`, `url` and
`userAgent`, trims over-long fields instead of rejecting them, and is capped at 30 reports per
minute per IP. It reads a bearer token if one is present — to attribute the report to a user — but
never rejects a request for lacking one, since the page that failed may not have a valid session.

The API also sends `helmet()` security headers, caps request bodies at 100 kB, and rate-limits the
credential endpoints (`/auth/login`, `/register`, `/resend_verification`, `/forgot_password`,
`/reset_password`, `/facebook/token`) per IP and per email address. In production it refuses to
start if `DB_URL`, `JWT_SECRET`, `FRONTEND_URL`, `CORS_ORIGIN`, `PRIVATE_KEY_PATH`, or
`METRICS_TOKEN` is unset or if `CORS_ORIGIN` is `*`.

### Auth (`/auth`)

| Endpoint | Access | Purpose |
| --- | --- | --- |
| `POST /register` | Public (rate-limited) | Create an account and send the verification email |
| `POST /login` | Public | Exchange credentials for a 7-day JWT |
| `POST /verify_email?token=` | Public | Verify the account and return a JWT (idempotent; `410` on an expired link) |
| `POST /resend_verification` | Public (rate-limited) | Re-send the verification link; non-enumerating response |
| `POST /role` | Any | Set the role once and return a refreshed JWT |
| `POST /forgot_password` | Public (rate-limited) | Email a password reset link; non-enumerating response |
| `POST /reset_password?token=` | Public (rate-limited) | Set a new password (`410` on an expired link) |
| `GET /google`, `GET /google/callback` | Public | Google OAuth sign-in (web or mobile via `client=mobile`) |
| `GET /facebook`, `GET /facebook/callback` | Public | Facebook OAuth sign-in (web or mobile via `client=mobile`) |
| `POST /facebook/token` | Public | Exchange a Facebook access token for a JWT (native mobile) |

### Users (`/user`)

| Endpoint | Access | Purpose |
| --- | --- | --- |
| `GET /has_role` | Any | Whether a role is set, and which |
| `GET /` | Any | The signed-in user's profile (no password or account tokens), plus a `hasPassword` flag |
| `PUT /` | Any | Update name (1–80 chars) and profile photo (email is read-only) |
| `PUT /password` | Any | Change password — current-password check, rejects reuse of that password; revokes other sessions and returns a fresh `token` for the caller |
| `DELETE /` | Any (rate-limited) | Delete the account and all its data after re-auth (password, or type-to-confirm email for social accounts); `409` while owning a live meeting |
| `GET /attendees?search=&offset=&limit=` | Organizer | Search attendee accounts by name or email — literal match, ≥2 chars, paged (default 20, max 50) with `meta: { total, offset, limit }`, id/name/profile and a masked email only |

### Events (`/events`) and sessions (`/sessions`)

| Endpoint | Access | Purpose |
| --- | --- | --- |
| `POST /events` | Organizer | Create an event (optional cover upload) |
| `GET /events` | Any | Calendar feed — all events for organizers, public only for attendees |
| `GET /events/events_by_query` | Any | Public events with search/time/category/date filters — paged, with `meta: { total, offset, limit }` (default 5) |
| `GET /events/own` | Organizer | The organizer's own events, filtered by type — paged, with `meta: { total, offset, limit }` (default 5) |
| `GET /events/organizer_summary` | Organizer | Home dashboard in one call — upcoming / awaiting-approval / pending-invitation counts, `next_event`, `live_meeting`, and a merged `recent_activity` list |
| `GET /events/attendee_summary` | Attendee | Home feed in one call — `happening_now`, `starting_soon` (next 24 h), `pending_invitations`, and `recommended` (upcoming public events they aren't already part of) |
| `GET /events/my?filter=` | Attendee | Everything the attendee is part of, deduped across invitations and registrations, each row carrying `participation_state` — `filter` is `all` (default) / `invited` / `registered` / `approved` / `attending`, an unknown value falls back to `all`; paged (default 10) with `meta: { total, offset, limit }` |
| `GET /events/:id` | Any (private events: owner or an invited/registered attendee only) | One event; for an attendee, `data.participation` carries the server-computed `{ state, meeting_started }` (see §3.5), `null` for organizers |
| `PUT /events/:id` | Organizer (owner) | Update an event — title, description, date, times, category, type and cover only |
| `DELETE /events/:id` | Organizer (owner) | Delete an event and everything attached to it — `409` while a meeting for it is live |
| `GET /events/:id/email_status` | Organizer (owner) | Event-level transactional-email rollup — `data: { sent, pending, failed, retryableFailed }`; optional `?action=` scopes it to one email type |
| `POST /events/:id/email_retry` | Organizer (owner) | Requeue this event's retryable email failures for re-delivery — `data: { requeued }`; optional `action` in the body scopes the retry |
| `GET /events/:id/sessions` | Any who can see the event | Sessions for the event, ordered by start time |
| `POST /sessions` | Organizer (event owner) | Add a session to an event |
| `GET /sessions` | Any who can see the event | Alias of `GET /events/:id/sessions`; event id in the `event-id` request header |
| `GET /sessions/:id` | Any | One session |
| `PUT /sessions/:id` | Organizer (event owner) | Update a session — title, description, speaker info and times only |
| `DELETE /sessions/:id` | Organizer (event owner) | Delete a session |

### Registrations (`/event_registers`)

| Endpoint | Access | Purpose |
| --- | --- | --- |
| `POST /` | Attendee | Register for an event — `403` for a private event without an invitation |
| `DELETE /:id` | Attendee | Unregister from an event — `409` while its meeting is live; notifies the organizer when an approved attendee leaves |
| `GET /:id` | Attendee | Whether the attendee has registered |
| `GET /:id/approved` | Attendee | Whether their registration is approved |
| `GET /events` | Attendee | Their registrations still awaiting approval — paged (default 20), newest-first, orphan-free, with `meta: { total, offset, limit }` |
| `GET /events/approved` | Attendee | Their approved registrations — paged (default 20), newest-first, orphan-free, with `meta: { total, offset, limit }` |
| `GET /:id/users` | Organizer (event owner) | Registrants for an event — paged (default 10), with `meta: { total, offset, limit }` |
| `GET /:id/users/approved` | Organizer (event owner) | Approved registrants for an event |
| `PUT /approve` | Organizer (event owner) | Approve the still-pending registrants among those named, emailing and notifying them; returns `data: { approved, skipped, email: { queued } }` (already-approved ids are skipped, idempotent) |
| `PUT /meeting_started` | Organizer (event owner) | Notify registrants the meeting started |

### Invitations (`/event_invites`)

| Endpoint | Access | Purpose |
| --- | --- | --- |
| `POST /` | Organizer (event owner) | Invite 1–200 verified attendees; skips already-invited ones and returns `data: { invited, skipped, email: { queued } }` |
| `GET /:id/users` | Organizer (event owner) | Everyone invited to an event |
| `GET /:id/accepted_users` | Organizer (event owner) | Invitees who accepted |
| `GET /events` | Attendee | Their pending invitations — paged (default 20), newest-first, orphan-free, with `meta: { total, offset, limit }` |
| `GET /accepted_events` | Attendee | Invitations they accepted — paged (default 20), newest-first, orphan-free, with `meta: { total, offset, limit }` |
| `PUT /accept` | Attendee | Accept an invitation — `404` if none, `400` if the event ended, idempotent `200` if already accepted |
| `PUT /meeting_started` | Organizer (event owner) | Notify invitees the meeting started |

### Meetings (`/meetings`) and participants (`/participants`)

| Endpoint | Access | Purpose |
| --- | --- | --- |
| `POST /meetings/token` | Any | Mint a two-hour 8x8.vc token for the event's meeting room, returned with that room's name (moderator flag derived from event ownership) |
| `POST /meetings` | Organizer | Create the event's meeting room with a server-generated room name |
| `GET /meetings/:id/is_created` | Organizer | Whether this organizer already started a meeting |
| `GET /meetings/:id` | Organizer | The organizer's meeting record for the event |
| `PUT /meetings/:id/start_time` | Organizer | Mark the host present and record the start instant (server-stamped; body ignored) |
| `PUT /meetings/:id/end_time` | Organizer | Record the end instant and compute the duration (server-stamped; body ignored) |
| `PUT /meetings/:id/end` | Organizer | End the meeting: flag ended, stamp end time + duration, close dangling participants, notify attendees |
| `PUT /meetings/:id/reopen` | Organizer | Clear the ended flag and re-notify attendees |
| `GET /meetings/:id/is_started` | Attendee | Whether a meeting exists, they were notified, and it is not ended |
| `GET /meetings/:id/attendee` | Attendee | The room name, ended flag and start time to join for the event |
| `GET /meetings/:id/is_expired` | Owner or participant | `{ exists, is_expired }` — `is_expired` is true once the meeting is ended or the join window has closed |
| `POST /meetings/token` — additional refusals | | `403` outside the join window, `409` if the host is not yet present, `409 This meeting is full.` at 25 concurrent participants |
| `POST /participants` | Attendee (participant of a live meeting) | Record joining — server sets the room name and join instant; refused outside the join window, before the host is present, or at capacity |
| `PUT /participants/:id` | Attendee | Record leaving and add the session's minutes to the stay duration (server-stamped; body ignored) |
| `PUT /participants/:id/no_end_time` | Organizer (event owner) | Close out participants still marked present |
| `GET /participants/:id` | Organizer (event owner) | Participants for an event, oldest join first — paged (default 20), with `meta: { total, offset, limit }` |
| `GET /participants/:id/stay_times` | Organizer (event owner) | Stay-time distribution for the charts |

### Notifications (`/notifications`)

| Endpoint | Access | Purpose |
| --- | --- | --- |
| `GET /notifications` | Any | The user's notifications, newest first — paged (`?offset=&limit=`, default 20/max 50), with `meta: { total, unread, offset, limit }` |
| `GET /notifications/unread_count` | Any | Unread count for the bell badge |
| `POST /notifications/mark_as_read` | Any | Mark the given notifications read (capped id array; only unread rows are touched) |
| `POST /notifications/mark_all_read` | Any | Mark every unread notification read |
| `DELETE /notifications` | Any | Delete the given notifications (id array in the body; JSON string in a query param still accepted) |

There is no client-facing email endpoint. Templated email is sent only as a side effect of the
actions that own it (verification, password reset, registration approval, invitation, meeting
start/end).

## 12. Data model

| Collection | Fields |
| --- | --- |
| User | profile, name, email, password (absent for OAuth-only accounts), role, tokenVersion (session-revocation counter), isVerified, verificationToken(+Expires), resetPasswordToken(+Expires), googleId, facebookId — the password and both token pairs are stripped from every serialised form of a user, including populated `user` sub-documents; the auth JWT payload carries only `_id`, `role` and `tokenVersion` |
| Event | cover, title, description, date, start_time, end_time, timezone (IANA), starts_at / ends_at (UTC instants derived server-side from date + times + timezone), category (`conference`/`meetup`/`webinar`), type (`public`/`private`), user (owner), updateNotifiedAt |
| Session | title, description, speaker_info, start_time, end_time, event |
| EventRegister | event, user, register_approved, meeting_started — unique per `(event, user)` |
| EventInvite | event, user, invitation_sent, invitation_accepted, meeting_started — unique per `(event, user)` |
| Meeting | event, user (host), room_name, start_time, end_time, duration, ended, ended_at — unique per `event` |
| Participant | event, user, room_name, start_time, end_time, duration — unique per `(event, user)`; `duration` accumulates across rejoin sessions |
| Notification | recipient, sender (event), type (`first_time_register`, `event_created`, `event_updated`, `register_approved`, `event_invited`, `meeting_started`, `meeting_ended`, `attendee_unregistered`), title, message, isRead, readAt — rows expire 90 days after creation via a TTL index, keeping the collection bounded |
| EmailLog | to, action, subject, from, meta (rendered `{ name, event_title, link }` vars), event (for organizer fan-out actions), status (`pending`/`sent`/`failed`), attempts (cumulative across retry sweeps), retryable (whether the last failure is worth retrying), lastError, sentAt — a durable one-row-per-email queue; rows expire 30 days after creation via a TTL index |
| AuditLog | actor (id, role and email, denormalised so the row survives the account being deleted), action, target (type + id), metadata (action-specific), requestId (matches the request's `x-request-id`), ip — append-only; rows expire 400 days after creation via a TTL index |

**Audit trail.** Security-sensitive actions write one `AuditLog` row each: `role.set`,
`register.approve`, `invite.send`, `meeting.create`, `meeting.end`, `event.delete`,
`account.delete`, `password.change` and `password.reset`. The write happens after the action has
already succeeded and is best-effort — a failed audit write is logged and swallowed, and never
fails or delays the request that caused it. Because the row carries the request id, an audit entry
can be lined up with the request's log line. There is no in-app view of the audit log; it is read
directly from the database during an investigation.

Schema changes ship with hand-rolled ordered migrations in `backend/src/migrations/` (`npm run
migrate`, recorded in a `_migrations` collection): among them, `003-event-timezone-backfill` seeds
`timezone` / `starts_at` / `ends_at` on pre-existing events, `005-user-token-version` seeds
`tokenVersion: 0` on every pre-existing account, and `008-audit-log-indexes` builds the audit log's
lookup and retention indexes. The `tokenVersion` cutover rejects any token issued
before it, so every user signs in again once the API deploys.

All documents carry creation and update timestamps. The `(event, user)` uniqueness on the three join
tables (and one meeting per event) is enforced by a unique index, so registering, being invited, or
joining a meeting twice updates the existing row instead of creating a duplicate. Deleting an event
cascades to its sessions, registrations, invitations, meetings and participants; the cascade is bound
to every Mongoose query-level delete method (`findOneAndDelete`, `deleteOne`, `deleteMany`), not just
the one the current code path uses. The cascade legs run independently so one failure cannot leave
the rest undone, and a background reconciliation sweep (90 s after boot, then every 30 min) deletes
any join/meeting/participant/session row whose parent event no longer exists. A meeting holds no 8x8.vc token: tokens are minted per join and
never persisted or returned as part of a meeting record.

Responses follow one envelope: `{ status: "success" | "error", message, data? }`, with boolean
checks (`has_registered`, `is_register_approved`, `is_created`, `is_started`, `is_expired`,
`has_role`) returned as their own top-level flags, and a minted meeting token returned as a
top-level `token` next to the `room_name` it is scoped to.

## 13. Configuration and integrations

- **MongoDB** stores all application data; **Socket.IO** carries live notifications.
- **Structured logging** uses `pino` (with `pino-http` replacing `morgan`): one structured JSON
  log line per request in production (pretty-printed locally), tagged `service: "ve-plan-api"`.
  Every request carries a correlation id — an inbound `x-request-id` header is honoured when
  present (trusted from the proxy, trimmed, capped at 128 chars), otherwise a UUID is minted —
  and the id is echoed back on the response `x-request-id` header. Request log level follows the
  status code (`5xx` → error, `4xx` → warn, else info). Unhandled errors are logged against the
  request's id and the `500` response body includes it as `data.requestId` for support
  correlation. Level is `info` in production, `debug` otherwise; override with the optional
  `LOG_LEVEL` env. Unhandled server errors are tagged `event: "unhandled_error"` and logged with
  their stack; reports from the web client arrive tagged `event: "client_error"`, so both classes
  can be filtered out of the request stream.
- **Metrics** are exposed in Prometheus text format on the token-guarded `GET /api/v1/metrics`:
  standard process metrics (event-loop lag, heap, GC) plus request duration and request count
  (labelled by method, matched route and status code — the route label is the mounted route pattern,
  never the raw URL, so it stays low-cardinality), and gauges for connected sockets, email queue
  depth (pending plus retryable failures) and MongoDB reachability. The three gauges are refreshed
  every 30 seconds. Nothing scrapes the endpoint by default; it is there for whatever collector the
  deployment adds.
- **Shutdown and process safety.** `SIGTERM` / `SIGINT` start an ordered drain — stop accepting HTTP
  connections, close the Socket.IO server, close the MongoDB connection, exit — with a 10-second cap
  after which the process exits non-zero anyway. This makes `npm run pm2:reload` a zero-downtime
  deploy, as opposed to `pm2:restart`, which drops in-flight requests and sockets. An unhandled
  promise rejection is logged and the process keeps serving; an uncaught exception is logged at fatal
  level and then drains and exits non-zero so the process manager restarts a known-good process. If
  the initial database connection fails the process exits rather than serving broken requests.
- **Backup and operations.** The backend repository carries `scripts/mongo-backup.sh` (a timestamped
  gzip `mongodump`, pruned to a retention window and appended to a backup log) and
  `scripts/mongo-restore.sh` (a `mongorestore` that requires typing `RESTORE`, or an explicit flag
  when run non-interactively). Uploaded photos live on local disk and are not part of the database
  dump. `docs/RUNBOOK.md` documents the deploy and rollback procedure, which migrations must run on
  deploy, backup/restore including a restore rehearsal log, incident response (where the logs are,
  correlating by request id, reading the metrics, querying the audit log), health and metric norms,
  common failures, and key rotation.
- **8x8.vc** provides video. The app ID and meeting domain are configured per environment, and
  meeting tokens are RS256-signed with a private key file (`PRIVATE_KEY_PATH`) for the configured
  Jitsi app and API key. Each token names one room, carries a two-hour expiry, and denies
  livestreaming, recording, transcription and outbound calls.
- **Google and Facebook OAuth** credentials and callback URLs are environment-configured, as are the
  Gmail SMTP OAuth2 email credentials and sender address. The OAuth-state cookie is signed with
  `COOKIE_SECRET`, falling back to `JWT_SECRET` when it is unset.
- The SPA reads the API URL, socket URL and path, static photo URLs, 8x8.vc settings and OAuth
  entry-point URLs from its environment file, which is swapped for the production variant at build
  time.
- The backend ships a `pm2` ecosystem config for production deployment, with a kill timeout longer
  than the shutdown drain so a reload is never cut short.
