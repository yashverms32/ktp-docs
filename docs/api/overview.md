---
sidebar_position: 1
---

# Overview

The **KTP API** (`ktp-api`) is the backend for the KTP Georgia web platform. It's a Node.js/Express REST API that manages member profiles, events, photos, and provides protected data to the website and iOS app.

---

## Architecture

| Layer | Technology |
|-------|-----------|
| Runtime | Node.js + Express |
| Database | PostgreSQL (raw SQL via `pg`, no ORM) |
| Auth | Authentik OIDC — JWT Bearer tokens (verified via JWKS) |
| Photos & video | Immich — fully integrated |
| Documents | ktp-api's own disk (`uploads/documents/`), not Immich — arbitrary file types |

**Internal address:** `http://10.0.0.53:4000` (LXC 119)  
**Public address:** `https://api2.ugaktp.com` (routed through Traefik on LXC 100)

The website (LXC 116) always hits the API via the internal address. The iOS app hits `https://api2.ugaktp.com`.

---

## Authentication

All protected routes require a valid **Authentik JWT** as a Bearer token in the `Authorization` header:

```
Authorization: Bearer <access_token>
```

The middleware (`middleware/auth.js`) validates the token by fetching the JWKS from Authentik:

```
AUTHENTIK_JWKS_URL=http://10.0.0.4:9000/application/o/ktpapp/jwks/
```

After verification, `req.user` is set to:

```js
{
  authentik_id: string,   // UUID — primary key in the DB
  username: string,
  groups: string[],       // e.g. ["active", "eboard"]
  authentik_pk: number    // Authentik's internal integer PK
}
```

Group-based access control (e.g., eboard-only routes) is enforced by additional middleware that checks `req.user.groups`.

---

## Database Schema

PostgreSQL database: `ugaktp_db` on LXC 118 (`10.0.0.54:5432`)

### `users` table

| Column | Type | Notes |
|--------|------|-------|
| `authentik_id` | `UUID PRIMARY KEY` | From Authentik JWT `sub` claim |
| `username` | `TEXT NOT NULL` | From Authentik — NOT UNIQUE |
| `authentik_pk` | `INTEGER` | Authentik's internal integer PK (for webhook deletion) |
| `first_name` | `TEXT` | |
| `last_name` | `TEXT` | |
| `preferred_name` | `TEXT` | |
| `dob` | `DATE` | |
| `major` | `TEXT` | |
| `graduation_date` | `TEXT` | Format: "Spring 2026" or "Fall 2026" |
| `phone` | `TEXT` | |
| `email` | `TEXT UNIQUE` | |
| `linkedin_url` | `TEXT` | |
| `pledge_class` | `TEXT` | e.g. "Alpha", "Beta" |
| `profile_picture_asset_id` | `TEXT` | Immich asset ID |
| `member_group` | `TEXT` | One of: `active`, `pledge`, `eboard`, `chair`, `alumni` |
| `profile_complete` | `BOOLEAN DEFAULT FALSE` | |
| `created_at` | `TIMESTAMPTZ` | |
| `updated_at` | `TIMESTAMPTZ` | Auto-updated by trigger |

> **Why `username` is not UNIQUE:** Authentik reuses usernames after deletion, and the webhook-based cleanup isn't yet reliable. The `authentik_id` UUID is the true unique identifier.

### Photos & albums

Member-facing photos are always members-only — there is no public/private flag per row anymore. Public content lives entirely in a separate table (`homepage_photos`, below).

| Table | Purpose |
|-------|---------|
| `albums` | Eboard-created named albums (e.g. "Spring Retreat 2026"). `created_by` → `users`. |
| `photos` | `immich_asset_id`, `album_id` (nullable FK → `albums`; `NULL` = the general "Shared Album"), `title`, `caption`, `media_type` (`image`/`video`), `uploaded_by` → `users`. |

Any member in `active`/`chair`/`alumni`/`eboard`/`pledge` (the full list lives in `constants.js` as `SHARED_ALBUM_GROUPS`, reused across every member-facing feature below) can upload/view. The uploader can always delete their own photo; an eboard member can additionally delete any photo inside an album *they personally created* (moderation, not blanket access to every album).

### `homepage_photos` table

The **public** chapter gallery shown on the actual homepage — a completely separate system from the member photos above, different audience and permission model. No auth on reads. Writes are eboard-only. Deleting a row only unlists it from the gallery — it deliberately does not delete the underlying Immich asset, since a registered asset might be reused elsewhere.

| Column | Notes |
|--------|-------|
| `immich_asset_id`, `title`, `caption`, `media_type` | Same shape as `photos` |
| `display_order` | Controls gallery ordering (eboard can reorder) |
| `added_by` | FK → `users` |

### Documents (`document_folders` / `documents`)

An eboard-managed file library (bylaws, meeting minutes, course files, etc.) shown in the Files & Photos portal tab alongside photos. Deliberately **not** Immich — these are arbitrary file types, stored directly on ktp-api's own disk at `uploads/documents/` (bind-mounted in `docker-compose.yml` so uploads survive container rebuilds).

| Table | Purpose |
|-------|---------|
| `document_folders` | `name`, `parent_id` (self-referencing FK — folders nest to any depth; `NULL` = top level), `created_by` |
| `documents` | `folder_id` (nullable — `NULL` = top level), `filename` (original name shown to users), `storage_path` (actual disk path, never exposed to clients), `mime_type`, `file_size`, `uploaded_by` |

View: any shared-album-group member. Writes (folders and files alike): eboard only. Deleting a folder cascades its DB rows *and* walks a recursive query to delete every nested file from disk too — disk space on the API's LXC is limited, so orphaned files aren't left behind.

### Messaging (`announcements` / `direct_messages` / `group_chats` + friends)

Three distinct systems, not one generic "messages" table:

| Table | Purpose |
|-------|---------|
| `announcements` | Eboard-only, one-way broadcast. `audience` is nullable — `NULL` means everyone in `SHARED_ALBUM_GROUPS`, otherwise one specific group. Filtered server-side against the caller's Authentik groups. |
| `direct_messages` | Any member ↔ any member. No separate "conversation" row — a thread is just every row where `(sender_id, recipient_id)` matches that pair in either direction. `read_at` tracks per-message read state. |
| `group_chats` | Eboard-created, named, with an assigned member list (`group_chat_members`, a plain junction table) — unlike DMs and announcements, access here is gated by actual DB membership rows, not a broad Authentik group. Any group_chats members up in it can post; only eboard can create/delete a chat or manage its membership. |
| `group_chat_messages` | Messages within a `group_chats` thread. |
| `group_chat_reads` | Per-user "last read" marker per group chat — a single message can be read by many different members at different times, so (unlike `direct_messages`) read state can't live on the message row itself. |

### `events` table

Standalone — no user FK. Full CRUD. `title`/`start_date`/`end_date` are validated server-side before touching the DB (a raw NOT NULL crash from a missing field was the original motivation).

---

## Environment Variables

Set in `/opt/ktp-api/.env` on LXC 119:

```env
DATABASE_URL=postgresql://root:<pass>@10.0.0.54:5432/ugaktp_db
AUTHENTIK_ISSUER=https://auth.ugaktp.com/application/o/ktpapp/
AUTHENTIK_JWKS_URL=http://10.0.0.4:9000/application/o/ktpapp/jwks/
WEBHOOK_SECRET=<hex32>
IMMICH_URL=http://10.0.0.3:2283
IMMICH_API_KEY=<scoped to asset upload/read/delete only>
PORT=4000
NODE_ENV=production
```

> **Hairpin NAT:** The Docker container can't reach Authentik via the public domain. Use the internal IP (`http://10.0.0.4:9000`) for `AUTHENTIK_JWKS_URL`.

---

## Deployment

```bash
# On LXC 119
cd /opt/ktp-api
./refresh.sh
```

`refresh.sh` handles the whole deploy in one step: stops the container, pulls latest, rebuilds and restarts, then re-applies the schema (`node scripts/init-db.js --schema-only` — safe to re-run any time, every table uses `CREATE TABLE IF NOT EXISTS` so existing tables/data are untouched). It ends by printing a reminder that Postgres grants still need to be applied manually — that step has to happen on LXC 118, which the script (running on LXC 119) can't reach into.

### Granting `root` access to any newly-created tables

Whenever `db/schema.sql` gains new tables (which has happened a lot — albums, homepage_photos, documents, messages, announcements, group chats), this has bitten every single migration on this project, since whoever runs the script may be connected as a different Postgres role than `root` (what `DATABASE_URL` connects as):

```sql
GRANT ALL PRIVILEGES ON TABLE <new_table> TO root;
GRANT ALL PRIVILEGES ON SEQUENCE <new_table>_id_seq TO root;  -- only for tables with a SERIAL id; skip for junction tables with composite PKs (e.g. group_chat_members)
```

Skipping this step produces `permission denied for table <name>` (Postgres error 42501) the first time anyone hits the new feature.

To access PostgreSQL directly (on LXC 118):

```bash
su - postgres
psql -d ugaktp_db
```
