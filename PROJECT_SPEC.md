# VE-Plan — Project Specification

VE-Plan is a virtual event planning platform. Organizers create events, publish session agendas,
approve or invite attendees, and host video meetings; attendees discover events, register or accept
invitations, and join those meetings. Video is provided by 8x8.vc (Jitsi as a Service).

This document describes what the product does today. It is a behavior spec, not a development
guide — see `CLAUDE.md` for build commands and code-level conventions.

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
it: the API checks the event's `user` field against the caller and rejects any other organizer with
`403 You are not the organizer of this event.`

## 2. Accounts and authentication

### 2.1 Sign up with email

- Sign-up collects name, email, password (minimum 6 characters, confirmed twice) and an optional
  profile photo.
- Duplicate email addresses are rejected with `409`.
- The account is created unverified, and a verification link
  (`<frontend>/verify_email?token=…`) is emailed. The token is valid for 24 hours.
- After submitting, the SPA sends the user to the "verify email" page.

### 2.2 Email verification

Opening the emailed link verifies the account, issues a 14-day JWT, stores it in the browser, and
takes the user to role selection. An invalid or expired token shows
"Verification failed. The link may be invalid or expired."

### 2.3 Login

Login fails with distinct messages for an unknown email (`404`), a wrong password (`401`) and an
account that has not verified its email yet (`401`, "Please verify your email to login."). On
success the API returns a 14-day JWT whose payload is the user document itself (minus password and
verification token), which is why the SPA can read name, role and profile photo without an extra
request. The user is routed straight to `organizer/dashboard/home` or `attendee/dashboard/home`
based on the role in that token.

### 2.4 Google and Facebook sign-in

Both providers are offered on the login screen and follow the same shape:

- If an account already exists with the same email address, the provider ID is linked to it.
- Otherwise a new account is created, already marked verified, with the provider's photo URL stored
  as the profile picture.
- The callback redirects to `<frontend>/social_login_redirect?token=…`; the SPA stores the token and
  goes to role selection, which immediately forwards users who already have a role to their
  dashboard.

Profile pictures coming from a provider are absolute URLs and are rendered as-is; uploaded pictures
are served from the API's static photo directory. Users with no picture get a placeholder image.

### 2.5 Role selection

Role selection offers "organizer" and "attendee". Setting a role returns a fresh JWT that carries it
and triggers the welcome notification (section 7). Attempting to set a role on an account that
already has one is rejected with `409 User role is already set.`

### 2.6 Password reset

Requesting a reset for an unknown email returns `404`. Otherwise a reset link
(`<frontend>/reset_password?token=…`) is emailed and is valid for one hour. Resetting requires a
password of at least 6 characters and clears the reset token.

### 2.7 Profile settings

The settings page (available to both roles) allows:

- Editing name and profile photo. Email is shown read-only and cannot be changed — it is the login
  identifier and account-recovery channel, and for social accounts it mirrors the OAuth provider. The
  `PUT /user` endpoint ignores any `email` field in the body.
- Changing the password, which requires the current password to be supplied and correct.

## 3. Events

### 3.1 What an event holds

An event has a cover image (optional), title, description, date, start time, end time, category,
visibility type, and an owning organizer. The UI offers `conference`, `meetup` and `webinar` as
categories and `public` / `private` as types.

### 3.2 Creating and editing

- Organizers create events from the calendar by clicking a day. The clicked date becomes the event
  date.
- Events cannot be created on a day that is more than 24 hours in the past — the calendar blocks it
  with an alert and the API independently rejects it with `409 Can't create an event in past days.`
- The event form requires title, description, date, start time, end time, category and type, and
  rejects a start time later than the end time.
- Editing uses the same form, pre-filled. Uploading a new cover replaces the old one; leaving it
  untouched keeps the existing cover.

### 3.3 Deleting

Deleting an event asks for confirmation and warns that everything attached to it is removed. The API
cascades the delete to the event's sessions, registrations, invitations, meeting and participant
records.

### 3.4 Visibility and discovery

- **Events list** (both roles): shows public events only, regardless of role. Filters are keyword
  search on title, time bucket (upcoming / happening / past), category and exact date; the active
  filter count is badged and each filter can be cleared individually. Results are paged 5 at a time
  with previous/next navigation, and the filter state lives in the URL query string.
- **Calendar** (home for both roles): a month view of events. Organizers see every event; attendees
  see public events only. Clicking an event opens a read-only details dialog.
- **My Events** (organizer only): the organizer's own events, filterable by all / public / private,
  loaded 5 at a time as the list is scrolled. Scroll position is restored when navigating back.

Private events do not appear in the events list; attendees reach them through an invitation, which
links directly to the event's detail page.

### 3.5 Event details page

Both roles see the event card plus its sessions. The organizer view additionally shows Edit, Delete,
and a menu linking to Registered Users, Invite Users and Meeting — but only for the organizer who
owns the event. The attendee view shows the registration/invitation state and the Join Meeting
button described in sections 4-6.

## 4. Sessions

Sessions are the agenda entries inside an event: title (required), description, speaker info, start
time and end time. Only organizers can add, edit or delete them, and the controls appear only on
events they own. Any signed-in user can read an event's sessions; the detail page shows
"There is no session found." when empty.

## 5. Registration flow (attendee-initiated)

1. On a public event, an attendee clicks **Register**. This creates a registration in the pending
   state. The button then reads "Registered" and clicking it again unregisters, after a
   confirmation prompt.
2. Register and unregister are disabled once the event's date and end time have passed.
3. The organizer opens **Registered Users** for the event: a searchable, paginated table (10 per
   page) of registrants with their approval status. Already-approved rows have no checkbox, and the
   Send Approval action is disabled once the event has ended.
4. Selecting registrants and confirming sends each of them a "Registration Approved" email and an
   in-app notification, and marks their registration approved.
5. On the attendee side the event page then shows "Register Approved" instead of the register
   button, and the event moves from the "Registered" bucket to "Register Approved" in Joined Events.

**Joined Events** (attendee) lists everything the attendee is part of, filterable by All, Registered
(still pending approval), Register Approved, and Invitation Accepted.

## 6. Invitation flow (organizer-initiated)

1. The organizer opens **Invite Users** for an event and types a keyword; the search returns
   attendee-role users matching the keyword by name or email (debounced, 500 ms).
2. The organizer selects users and confirms. If any selected user has already been invited to that
   event, the whole request is rejected with `409` and a message naming those users — no invitations
   or emails are sent.
3. Otherwise each invitee receives an "Event Invitation" email and an in-app notification, and an
   invitation record is created in the pending state.
4. The organizer can review **Invited Users** and **Accepted Users** for the event in dialogs.
5. The attendee's **Invitations** page lists pending invitations with a link to the event details
   and an **Accept** button. Accepting marks the invitation accepted; the event then appears under
   "Invitation Accepted" in Joined Events and the event page shows the "Invitation Accepted" state.

An attendee who has been invited to an event sees the invitation status on the event page instead of
the register button — invitation and registration are alternative paths into the same event.

## 7. Meetings (8x8.vc / Jitsi)

### 7.1 Starting and hosting (organizer)

- The event's **Meeting** page shows **Start Meeting** if no meeting exists yet for that event and
  organizer. Starting one mints a moderator-level 8x8.vc JWT and stores a meeting record with a
  randomly generated room name. Attempting to start a second meeting for the same event is rejected.
- Once created, the button becomes **Join Meeting** and opens a full-screen dialog embedding the
  8x8.vc conference for room `<appId>/<room name>`, joined with the stored moderator token. The
  embed enables the prejoin screen, disables in-conference invite and kick controls, and starts
  audio and video muted.
- Joining records the meeting's start time; leaving records the end time, computes the duration in
  whole minutes, closes out any participants still marked as present, and reloads the page so the
  analytics reflect the finished meeting.
- Meeting tokens expire 24 hours after they are minted. If the token has expired, opening the
  meeting shows "Can't join this meeting since meeting token is expired." and closes the dialog.
- Both Start and Join are disabled once the event's date and end time have passed.

### 7.2 Notifying attendees

From the meeting page, **View Attendees** lists the event's approved registrants and accepted
invitees, deduplicated per user, paginated 10 per page and searchable. Selecting users and confirming
sends each a "Meeting Started" email plus an in-app notification and flags them as notified.
Already-notified rows have no checkbox, and the send action is disabled once the event has ended.

### 7.3 Joining (attendee)

- The **Join Meeting** button appears on the attendee's event page only when a meeting exists for the
  event *and* the attendee has been notified as described above. It is disabled after the event has
  ended.
- The attendee joins the organizer's room with a freshly minted non-moderator token, so the organizer
  remains the only moderator.
- Joining creates the attendee's participant record with a start time; leaving records the end time
  and the stay duration in minutes. There is one participant record per attendee per event, so
  rejoining does not create a second record.

### 7.4 Meeting analytics (organizer)

The meeting page summarizes attendance for the organizer:

- Counter cards: event attendees, joined participants, approved registrants, invitation-accepted
  users.
- A doughnut chart comparing event attendees against participants who actually joined.
- A line chart of stay-time distribution, bucketed into four ranges derived from the meeting's total
  duration.
- A paginated table of joined participants with join time, leave time and stay time in minutes.

## 8. Notifications

Notifications are stored per recipient and also pushed live over a Socket.IO connection that is
authenticated with the same JWT; each user is placed in a private room so notifications are only
delivered to their own sessions. The socket connects when a dashboard is opened.

| Trigger | Recipients | Title |
| --- | --- | --- |
| Role chosen for the first time | The new user | Welcome to VE-Plan! |
| Public event created | All verified users | Event Created |
| Public event updated | All verified users | Event Updated |
| Registration approved | The approved attendees | Registration Approved |
| Invitation sent | The invited attendees | Invitation |
| Meeting started notice sent | The selected attendees | Meeting Started |

Private events do not generate created/updated notifications.

The toolbar shows a bell with an unread badge. The **Notifications** page lists notifications newest
first, merges anything arriving live while the page is open, and lets the user select entries to
mark as read or delete them. Acting with nothing selected shows a prompt to select at least one.

## 9. Transactional email

Emails are rendered from HTML templates with `{{name}}`, `{{event_title}}` and `{{link}}`
placeholders and sent over SMTP with OAuth2 credentials. The five templates in use are:

| Action | Subject | Sent when |
| --- | --- | --- |
| `email_verified` | Email Verification | A new account is registered |
| `reset_password` | Password Reset | A password reset is requested |
| `register_approved` | Registration Approved | An organizer approves registrants |
| `invitation_sent` | Event Invitation | An organizer invites attendees |
| `meeting_started` | Meeting Started | An organizer notifies attendees a meeting began |

## 10. Application structure and access control

### 10.1 Navigation

Both dashboards share a toolbar (notifications bell, profile menu with Settings, Logout, Terms and
Conditions, Privacy Policy) and a sidebar. The sidebar shows Calendar and Events to everyone,
My Events to organizers, and Invitations plus Joined Events to attendees. Logging out asks for
confirmation, then clears the stored token and returns to login.

### 10.2 Routes

Public: `/login`, `/signup`, `/role`, `/verify_email`, `/forgot_password`, `/reset_password`,
`/social_login_redirect`, `/terms_and_conditions`, `/privacy_policy`. Unknown paths outside a
dashboard fall back to login.

Organizer dashboard (`/organizer/dashboard/…`): `home`, `events`, `my_events`, `events/:id/view`,
`events/:id/registered_users`, `events/:id/invite`, `events/:id/meeting`,
`events/:id/meeting/attendees`, `setting`, `notifications`, plus a not-found page.

Attendee dashboard (`/attendee/dashboard/…`): `home`, `events`, `events/:id/view`, `invitations`,
`joined_events`, `setting`, `notifications`, plus a not-found page.

### 10.3 Client-side handling of API failures

The SPA keeps its JWT in browser local storage and attaches it as a bearer token. Global HTTP error
handling routes the user according to the failure: `401` returns to login, `403` returns to the
signed-in user's own dashboard home, and `404` shows the dashboard's not-found page. The `/role`
page additionally redirects users who are not signed in to login, and users who already have a role
straight to their dashboard.

### 10.4 Uploads

Cover images and profile photos are uploaded as multipart form data and served back as static files
under the API's `/static/covers` and `/static/profiles` paths.

## 11. API surface

All endpoints are mounted under `/api/v1`. "Any" means any authenticated user regardless of role.
"Organizer (owner)" means the request is additionally checked against the event's owning organizer,
rejecting any other organizer with `403`. Unhandled request errors fall through to a catch-all
handler that responds `500 Something went wrong.` instead of leaking the underlying error.

### Auth (`/auth`)

| Endpoint | Access | Purpose |
| --- | --- | --- |
| `POST /register` | Public | Create an account and send the verification email |
| `POST /login` | Public | Exchange credentials for a 14-day JWT |
| `POST /verify_email?token=` | Public | Verify the account and return a JWT |
| `POST /role` | Any | Set the role once and return a refreshed JWT |
| `POST /forgot_password` | Public | Email a password reset link |
| `POST /reset_password?token=` | Public | Set a new password |
| `GET /google`, `GET /google/callback` | Public | Google OAuth sign-in |
| `GET /facebook`, `GET /facebook/callback` | Public | Facebook OAuth sign-in |

### Users (`/user`)

| Endpoint | Access | Purpose |
| --- | --- | --- |
| `GET /has_role` | Any | Whether a role is set, and which |
| `GET /` | Any | The signed-in user's profile |
| `PUT /` | Any | Update name and profile photo (email is read-only) |
| `PUT /password` | Any | Change password with current-password check |
| `GET /attendees?search=` | Organizer | Search attendee accounts by name or email |

### Events (`/events`) and sessions (`/sessions`)

| Endpoint | Access | Purpose |
| --- | --- | --- |
| `POST /events` | Organizer | Create an event (optional cover upload) |
| `GET /events` | Any | Calendar feed — all events for organizers, public only for attendees |
| `GET /events/events_by_query` | Any | Public events with search/time/category/date filters and paging |
| `GET /events/own` | Organizer | The organizer's own events, filtered by type, paged |
| `GET /events/:id` | Any | One event |
| `PUT /events/:id` | Organizer (owner) | Update an event |
| `DELETE /events/:id` | Organizer (owner) | Delete an event and everything attached to it |
| `POST /sessions` | Organizer | Add a session to an event |
| `GET /sessions` | Any | Sessions for the event given in the `event-id` request header |
| `GET /sessions/:id` | Any | One session |
| `PUT /sessions/:id` | Organizer | Update a session |
| `DELETE /sessions/:id` | Organizer | Delete a session |

### Registrations (`/event_registers`)

| Endpoint | Access | Purpose |
| --- | --- | --- |
| `POST /` | Attendee | Register for an event |
| `DELETE /:id` | Attendee | Unregister from an event |
| `GET /:id` | Attendee | Whether the attendee has registered |
| `GET /:id/approved` | Attendee | Whether their registration is approved |
| `GET /events` | Attendee | Their registrations still awaiting approval |
| `GET /events/approved` | Attendee | Their approved registrations |
| `GET /:id/users` | Organizer | Registrants for an event, paged |
| `GET /:id/users/approved` | Organizer | Approved registrants for an event |
| `PUT /approve` | Organizer | Approve registrants, emailing and notifying them |
| `PUT /meeting_started` | Organizer | Notify registrants the meeting started |

### Invitations (`/event_invites`)

| Endpoint | Access | Purpose |
| --- | --- | --- |
| `POST /` | Organizer | Invite attendees, rejecting duplicates |
| `GET /:id/users` | Organizer | Everyone invited to an event |
| `GET /:id/accepted_users` | Organizer | Invitees who accepted |
| `GET /events` | Attendee | Their pending invitations |
| `GET /accepted_events` | Attendee | Invitations they accepted |
| `PUT /accept` | Attendee | Accept an invitation |
| `PUT /meeting_started` | Organizer | Notify invitees the meeting started |

### Meetings (`/meetings`) and participants (`/participants`)

| Endpoint | Access | Purpose |
| --- | --- | --- |
| `POST /meetings/token` | Any | Mint an 8x8.vc token (moderator flag from the request) |
| `POST /meetings` | Organizer | Create the event's meeting room |
| `GET /meetings/:id/is_created` | Organizer | Whether this organizer already started a meeting |
| `GET /meetings/:id` | Organizer | The organizer's meeting record for the event |
| `PUT /meetings/:id/start_time` | Organizer | Record when the meeting began |
| `PUT /meetings/:id/end_time` | Organizer | Record the end and compute the duration |
| `GET /meetings/:id/is_started` | Attendee | Whether a meeting exists and they were notified |
| `GET /meetings/:id/attendee` | Attendee | The room to join for the event |
| `GET /meetings/:id/is_expired` | Any | Whether the meeting token has expired |
| `POST /participants` | Attendee | Record joining the meeting |
| `PUT /participants/:id` | Attendee | Record leaving and compute stay time |
| `PUT /participants/:id/no_end_time` | Organizer | Close out participants still marked present |
| `GET /participants/:id` | Organizer | Participants for an event |
| `GET /participants/:id/stay_times` | Organizer | Stay-time distribution for the charts |

### Notifications (`/notifications`) and email (`/emails`)

| Endpoint | Access | Purpose |
| --- | --- | --- |
| `GET /notifications` | Any | The user's notifications, newest first |
| `GET /notifications/unread_count` | Any | Unread count for the bell badge |
| `POST /notifications/mark_as_read` | Any | Mark selected notifications read |
| `DELETE /notifications` | Any | Delete selected notifications |
| `POST /emails` | Organizer | Send one of the templated emails |

## 12. Data model

| Collection | Fields |
| --- | --- |
| User | profile, name, email, password (absent for OAuth-only accounts), role, isVerified, verificationToken(+Expires), resetPasswordToken(+Expires), googleId, facebookId |
| Event | cover, title, description, date, start_time, end_time, category, type, user (owner) |
| Session | title, description, speaker_info, start_time, end_time, event |
| EventRegister | event, user, register_approved, meeting_started |
| EventInvite | event, user, invitation_sent, invitation_accepted, meeting_started |
| Meeting | event, user (host), room_name, token, start_time, end_time, duration |
| Participant | event, user, room_name, start_time, end_time, duration |
| Notification | recipient, sender (event), type, title, message, isRead, readAt |

All documents carry creation and update timestamps. Deleting an event cascades to its sessions,
registrations, invitations, meetings and participants.

Responses follow one envelope: `{ status: "success" | "error", message, data? }`, with boolean
checks (`has_registered`, `is_register_approved`, `is_created`, `is_started`, `is_expired`,
`has_role`) returned as their own top-level flags.

## 13. Configuration and integrations

- **MongoDB** stores all application data; **Socket.IO** carries live notifications.
- **8x8.vc** provides video. The app ID and meeting domain are configured per environment, and
  meeting tokens are RS256-signed for the configured Jitsi app.
- **Google and Facebook OAuth** credentials and callback URLs are environment-configured, as are the
  SMTP/OAuth2 email credentials, sender address and template paths.
- The SPA reads the API URL, socket URL, static photo URLs, 8x8.vc settings and OAuth entry-point
  URLs from its environment file, which is swapped for the production variant at build time.
