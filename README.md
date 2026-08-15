# Social Network – 2-Person Work Split (v2)

---

## Core Principles

1. **Split by end-to-end feature, not by frontend/backend.** Each person owns their features from migration → Go handler → API → UI.
2. **One implementation per concept.** Posts are posts, whether they live in a group or not. No parallel `posts` and `group_posts` systems.
3. **Shared infrastructure is built first, together.** Anything both people call into (auth middleware, WebSocket hub, notifications, API client, frontend shell) gets built in Phase 1 by both, then frozen.
4. **Agree on contracts before writing code.** Table names, error shape, endpoint naming, WebSocket message format.

---

# Phase 1 – Foundation (Both, Together)

Nothing in Phase 2 starts until this is merged into `develop`.

## Backend

- Repository setup, Go module, folder structure
- SQLite connection (`pkg/db/sqlite/sqlite.go`)
- Migration system wired to `file://backend/pkg/db/migrations/sqlite`
- `users` and `sessions` tables
- Registration, login, logout (enough to get a session cookie)
- Auth middleware — every later handler depends on knowing the current user
- Standard error response format
- **WebSocket hub skeleton**: connect, authenticate via session cookie, route a message to a given user ID, handle disconnect
- **Notifications table + generic `CreateNotification()` helper**

## Frontend

- Framework setup (Next.js / Vue / Svelte — pick one, commit to it)
- App shell: layout, navigation, routing
- Auth/session context (who is the current user)
- API client wrapper (base URL, credentials, error handling)
- WebSocket client wrapper (connect, reconnect, subscribe by message type)
- Notification bell component in the nav

## Docker

- `Dockerfile.backend`, `Dockerfile.frontend`, `docker-compose.yml`
- Backend port exposed, frontend port exposed, volume for uploads

## Database Design

Do the ER diagram together on paper/whiteboard before writing a single migration.

Agree on and write down:

- Table names and column names
- Primary key format (INTEGER autoincrement vs UUID — pick one, use everywhere)
- Timestamp format and column names (`created_at`, `updated_at`)
- Foreign keys and `ON DELETE` behaviour
- Nullable vs NOT NULL conventions

---

# Contracts (Lock These Down in Phase 1)

## Error Response Format

Every error from every endpoint:

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Email is already registered"
  }
}
```

Agreed codes: `VALIDATION_ERROR`, `UNAUTHORIZED`, `FORBIDDEN`, `NOT_FOUND`, `CONFLICT`, `INTERNAL_ERROR`.

## Current User Payload

Returned by `GET /api/auth/me` and embedded wherever a user is referenced:

```json
{
  "id": 1,
  "email": "user@example.com",
  "first_name": "Ali",
  "last_name": "Hassan",
  "nickname": "ali",
  "avatar_path": "/uploads/avatars/abc123.jpg",
  "is_public": true
}
```

## API Naming

```text
POST   /api/auth/register
POST   /api/auth/login
POST   /api/auth/logout
GET    /api/auth/me

GET    /api/users/:id
PATCH  /api/users/:id
POST   /api/users/:id/follow
DELETE /api/users/:id/follow
GET    /api/users/:id/followers
GET    /api/users/:id/following

GET    /api/posts
POST   /api/posts
GET    /api/posts/:id
POST   /api/posts/:id/comments
GET    /api/posts/:id/comments

GET    /api/groups
POST   /api/groups
GET    /api/groups/:id
POST   /api/groups/:id/join
POST   /api/groups/:id/invite
GET    /api/groups/:id/posts
POST   /api/groups/:id/events

GET    /api/notifications
POST   /api/notifications/:id/read

GET    /ws
```

Never: `/getUsers`, `/create_post`, `/groups/create`, `/api/getMessages`.

## WebSocket Message Envelope

Every message in both directions:

```json
{
  "type": "private_message",
  "payload": { }
}
```

Agreed types:

```text
private_message
group_message
notification
typing
error
```

Person 2 owns chat payload shapes. Person 1 owns notification payload shape. Both must fit this envelope.

---

# Phase 2 – Parallel Feature Development

## Person 1 – Identity, Content & Notifications

### Profiles

- Profile page (info, activity, followers, following)
- Public/private toggle
- Visibility enforcement: private profile shows full info only to accepted followers

### Followers

```text
User A follows User B
        ↓
Is User B public?
   ┌────┴────┐
  Yes        No
   │          │
Follow      Send Request
Directly       │
          Accept / Decline
```

- Follow, unfollow
- Follow request, accept, decline
- Auto-follow for public profiles
- Prevent duplicate requests

### Posts & Comments Engine

**This is one system that serves both regular posts and group posts.**

- `posts` table has a **nullable `group_id`**
- `group_id IS NULL` → normal post, privacy rules apply
- `group_id IS NOT NULL` → group post, membership check applies (Person 2 supplies that check)
- One `comments` table, one comment UI, one create-comment endpoint

Privacy for non-group posts:

1. **Public** — all users
2. **Almost private** — followers only
3. **Private** — selected followers only (`post_allowed_users`)

### Image Handling

One upload pipeline used by posts, comments, and avatars.

```text
Uploaded File
     ↓
Validate MIME + extension (JPEG, PNG, GIF)
     ↓
Enforce size limit
     ↓
Save to /uploads with generated filename
     ↓
Store path in SQLite
```

### Notifications

- Notification bell, dropdown, unread count
- Read/unread state
- Renders any notification type by `type` field
- Emits: `follow_request`
- Person 2's handlers call `CreateNotification()` for their own types

### Chat Frontend

Person 1 builds the chat **UI**, Person 2 builds the chat **backend**.

- Conversation list
- Message thread view
- Message input + emoji picker
- Consumes `private_message` / `group_message` over the shared WebSocket client
- Contract is the message envelope agreed in Phase 1

### Person 1 Tables

```text
users              (Phase 1)
sessions           (Phase 1)
notifications      (Phase 1)
followers
follow_requests
posts              (includes nullable group_id)
post_allowed_users
comments
```

---

## Person 2 – Communities & Real-Time Backend

### Groups

- Create group (title, description)
- Browse all groups
- Membership
- Invite users → accept / reject
- Request to join → creator accepts / rejects
- Existing members can also invite

```text
User
 ↓
Member of Group?
 ┌──────┴──────┐
Yes            No
 │              │
Access        Denied
```

- Emits notifications: `group_invitation`, `group_join_request`

### Group Content Gating

Person 2 does **not** rewrite posts. Person 2 provides:

```go
func IsGroupMember(userID, groupID int) (bool, error)
```

Person 1's post handlers call this when `group_id` is set. Agree on this signature in Phase 1.

### Events

- Create event (title, description, day/time)
- Responses: Going / Not Going
- Members only
- Emits notification: `group_event_created`

### WebSocket Hub (full implementation)

Builds on the Phase 1 skeleton.

```text
User A
  │ WebSocket
  ▼
Backend WebSocket Hub
  │ WebSocket
  ▼
User B
```

- Connection registry per user
- Room/broadcast support for groups
- Message persistence to SQLite
- Delivery of `notification` messages on behalf of Person 1

### Private Chat Backend

- Allowed only when at least one user follows the other
- Persist to `private_messages`
- Deliver live if recipient connected, otherwise available on next fetch
- `GET /api/messages/:userId` for history

### Group Chat Backend

```text
         Group WebSocket Room
        /        |         \
   User A     User B      User C
```

- Only members can join the room
- Persist to `group_messages`
- Broadcast to all connected members

### Person 2 Tables

```text
groups
group_members
group_invitations
group_requests
events
event_responses
private_messages
group_messages
```

---

# Ownership Summary

| Area | Person 1 | Person 2 |
|---|:---:|:---:|
| Auth, sessions, cookies | Lead | Support |
| Profiles | Lead | |
| Followers & follow requests | Lead | |
| Posts & comments engine | Lead | |
| Post privacy | Lead | |
| Image uploads | Lead | |
| Feed | Lead | |
| Notifications UI + delivery | Lead | Emits events |
| Chat frontend (UI, emoji) | Lead | |
| Groups & membership | | Lead |
| Invitations & join requests | | Lead |
| Group content gating helper | Consumes | Lead |
| Events & responses | | Lead |
| WebSocket hub | Support | Lead |
| Private chat backend | | Lead |
| Group chat backend | | Lead |
| DB design & ER diagram | Both | Both |
| Migration strategy | Both | Both |
| Docker | Both | Both |
| Integration testing | Both | Both |

---

# Phase 3 – Integration

Test complete cross-feature flows, not individual endpoints.

```text
Private Profile
      ↓
Follow Request
      ↓
Notification appears live
      ↓
Accept
      ↓
Follower can see almost-private posts
      ↓
Private chat becomes available
```

```text
Browse Groups
    ↓
Request to Join
    ↓
Creator gets notification
    ↓
Accepted
    ↓
Group posts visible
    ↓
Event created → notification to all members
    ↓
Group chat works
```

Also test:

- Logout mid-WebSocket-session
- Two browsers, same user
- Non-member hitting group endpoints directly
- Private post visible to selected follower, invisible to other followers
- GIF upload in a comment
- Refresh with an open chat

---

# Migrations

Do **not** pre-allocate blocks of numbers — you will collide the first time you branch simultaneously.

**Rule:** claim a number in the shared chat before creating the file. Format:

```text
claiming 000011_add_post_privacy
```

Alternative if you prefer zero coordination: timestamp prefixes (`20250315143000_create_groups.up.sql`).

Phase 1 migrations (written together):

```text
000001_create_users
000002_create_sessions
000003_create_notifications
```

Everything after that is claimed on demand.

**Once a migration is pushed to `develop`, it is frozen.** Fix it with a new migration, never by editing.

---

# Files That Cause Merge Conflicts

Touch these only after telling your partner:

```text
backend/server.go              (route registration)
backend/pkg/db/sqlite/*        (connection & migration runner)
backend/websocket/hub.go       (shared infrastructure)
frontend/ layout & nav
frontend/ api client
frontend/ websocket client
docker-compose.yml
```

Rule of thumb: if you're editing a Phase 1 file during Phase 2, pause and ask.

---

# Git Workflow

```text
main
│
└── develop
     ├── feature/profiles
     ├── feature/followers
     ├── feature/posts
     ├── feature/notifications
     ├── feature/groups
     ├── feature/events
     └── feature/chat
```

```bash
git checkout develop
git pull
git checkout -b feature/posts
# work
git add .
git commit -m "feat: implement post creation"
git push -u origin feature/posts
```

Then open a PR into `develop`. Partner reviews. Merge. Integration test.

Merge `develop` into your feature branch daily — do not let a branch drift for a week.

## Commit Style

```text
feat: add user registration
feat: implement follow requests
feat: add post privacy
fix: prevent duplicate follow requests
refactor: move queries into repository package
docs: add API documentation
```

Not: `update`, `stuff`, `fix`, `final`, `final2`, `working`.

---

# Project Structure

```text
social-network/
│
├── backend/
│   ├── handlers/
│   ├── middleware/
│   ├── models/
│   ├── services/
│   ├── websocket/
│   │
│   ├── pkg/
│   │   └── db/
│   │       ├── migrations/
│   │       │   └── sqlite/
│   │       └── sqlite/
│   │
│   └── server.go
│
├── frontend/
│
├── uploads/
│
├── docker-compose.yml
├── Dockerfile.backend
└── Dockerfile.frontend
```

---

# Allowed Packages

- Standard Go library
- `gorilla/websocket`
- `golang-migrate` (or `sql-migrate` / `Boostport/migration`)
- `mattn/go-sqlite3`
- `golang.org/x/crypto/bcrypt`
- `gofrs/uuid` or `google/uuid`

---

# What Changed From v1

| v1 | v2 | Why |
|---|---|---|
| Person 2 owned groups, events, both chats, WebSockets **and** notifications | Notifications moved to Person 1; chat frontend moved to Person 1 | Person 2's load was ~2x and contained the two hardest subsystems |
| Separate `group_posts` / `group_comments` tables | One `posts` table with nullable `group_id` | Avoids two divergent implementations of the same feature |
| WebSocket hub built in Phase 3 | Hub skeleton built in Phase 1 | Notifications need it; Phase 3 start would block Person 1 |
| Notifications owned by Person 2 | Shared infrastructure, each person emits their own types | Triggers are split across both people's features |
| Frontend shell unassigned | Explicitly Phase 1, both | Highest merge-conflict surface in the repo |
| Migration numbers pre-allocated in blocks | Claim-before-write or timestamps | Blocks collide as soon as you branch in parallel |
| No error format agreed | Error shape and current-user payload locked in Phase 1 | Retrofitting these at integration time is painful |
| 4 phases | 3 phases | Old Phase 2/3 split existed only because of the chat bottleneck |
