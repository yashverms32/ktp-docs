---
sidebar_position: 2
---

# API Endpoints

Base URL (production): `https://api2.ugaktp.com`  
Base URL (internal): `http://10.0.0.53:4000`

All protected routes require `Authorization: Bearer <access_token>`.

---

## Users

### `POST /users/sync`

**Auth required.** Called automatically on every login by the website (`auth.ts`). Creates or updates the user's row in the database and returns their `profile_complete` status.

**Request body:**
```json
{
  "authentik_id": "uuid",
  "username": "jsmith",
  "member_group": "active"
}
```

**Response:**
```json
{
  "authentik_id": "uuid",
  "profile_complete": false
}
```

The `member_group` is determined by group priority: `eboard > chair > active > pledge > alumni`.

---

### `GET /users/me`

**Auth required.** Returns the full profile row for the authenticated user.

**Response:**
```json
{
  "authentik_id": "uuid",
  "username": "jsmith",
  "first_name": "John",
  "last_name": "Smith",
  "preferred_name": "John",
  "dob": "2003-01-15",
  "major": "Computer Science",
  "graduation_date": "Spring 2026",
  "phone": "4045550000",
  "email": "jsmith@uga.edu",
  "linkedin_url": "https://linkedin.com/in/jsmith",
  "pledge_class": "Alpha",
  "member_group": "active",
  "profile_complete": true,
  "created_at": "...",
  "updated_at": "..."
}
```

---

### `PUT /users/me/profile`

**Auth required.** Saves profile fields and sets `profile_complete = true`. Accepts `multipart/form-data`.

**Form fields:**
- `first_name`, `last_name`, `preferred_name`
- `dob` — date of birth
- `major`
- `graduation_date` — string like "Spring 2026" (the form combines semester + year before sending)
- `phone`
- `email`
- `linkedin_url`
- `pledge_class`

**Response:**
```json
{ "profile_complete": true }
```

After a successful response, the website calls `update({ profile_complete: true })` to update the NextAuth session without a round-trip.

---

### `PUT /users/me/profile-picture`

**Auth required.** Multipart upload, field name `file`. Images only (JPEG/PNG/WebP), 10MB limit — a separate, stricter multer config than the general photo uploader. Uploads to Immich and saves the returned asset id onto the user's row.

---

### `GET /users/:id/profile-picture/media`

**Auth required.** Streams a member's profile picture from Immich (forwards `Range` headers). Generalized to **any** user id, not just "me" — needed so the directory, admin panel, and message threads can show other members' pictures too. 404s if the member has no picture set (the frontend's `AvatarImage` falls back to initials on a 404, no special-casing needed).

---

### `DELETE /users/me`

**Auth required.** Self-service account deletion. **Anonymizes, does not hard-delete** — clears every PII field (name, email, phone, major, etc.), sets `profile_complete = false`, and stamps `deleted_at`, but leaves the row (and their messages/photos/committee history) in place so other members' conversations and albums aren't broken. Excluded from `/members` and `/admin/users` afterward. Best-effort deletes their Immich profile-picture asset first.

Never touches Authentik — revoking real chapter/SSO access is a separate, eboard-owned action (deleting them in Authentik directly, which the webhook below still hard-deletes for). The website signs the user out immediately after this succeeds; any client calling this should do the same.

### `GET /users/blocked`

**Auth required.** The caller's own block list — people they've blocked, not people who've blocked them.

### `POST /users/:id/block`

**Auth required.** Blocks a member. Prevents new direct messages starting in either direction and hides the blocked user's messages from the caller's own DM and group chat views. One-directional and idempotent.

### `DELETE /users/:id/block`

**Auth required.** Unblocks.

---

## Members

### `GET /members`

**Auth required.** Returns a list of members. Supports optional `?group=` query param to filter by `member_group`.

**Query params:**
- `group` — one of `active`, `pledge`, `eboard`, `chair`, `alumni`

**Response:**
```json
[
  {
    "authentik_id": "uuid",
    "first_name": "John",
    "last_name": "Smith",
    "member_group": "active",
    "graduation_date": "Spring 2026",
    ...
  }
]
```

---

### `GET /members/:id`

**Auth required.** Returns a single member by `authentik_id`.

---

## Events

Reads (`GET /events`, `GET /events/:id`) use `optionalAuth` — a valid token personalizes the results (targeted events included), but a missing/invalid token falls back to public-only results rather than rejecting (needed so the mobile app's calendar, which fetches anonymously, keeps working). Writes require `requireAuth` + membership in `SHARED_ALBUM_GROUPS`.

### `GET /events`

Returns events visible to the caller: public ones (no `audience`/`committee_id` set) always; plus, if authenticated, ones where `audience` overlaps the caller's Authentik groups, or the caller is a member of the event's `committee_id`. Eboard gets every event unfiltered. Chair and eboard members count as `active` for this overlap even though Authentik keeps the three groups mutually exclusive — targeting `audience: ["active"]` still reaches them (Announcements and Polls, targeted the same way as Events, inherit this too).

### `POST /events`

**Request body:**
```json
{
  "title": "General Meeting",
  "description": "Weekly chapter meeting",
  "date": "2026-02-01T18:00:00Z",
  "location": "Boyd GSRC",
  "audience": ["active", "chair"],
  "committee_id": null,
  "calendly_url": null
}
```

Permission depends on who's asking:
- **Eboard**: can set any `audience` array and/or any `committee_id`.
- **Committee chair**: can only set `committee_id` to a committee they chair — no `audience` (implicitly scoped to that committee's members).
- **Anyone else**: 403.

`title`/`date`(start/end)/`location` fields are validated before touching the DB.

### `PUT /events/:id`

Same permission logic as `POST /events`.

### `DELETE /events/:id`

Eboard can delete any event; a non-eboard creator can delete their own.

---

## Committees

DB-only membership (no Authentik group per committee) — same shape as Group Chats below. Every committee gets a linked group chat created alongside it, and joining/leaving a committee automatically joins/leaves that chat too.

### `GET /committees`

**Auth required.** All committees, with member count and whether the caller is a member/chair of each.

### `POST /committees`

**Eboard only.** `{ "name": "..." }`

### `DELETE /committees/:id`

**Eboard only.** Also deletes the linked group chat.

### `POST /committees/:id/join`

Self-service — adds the caller as `role: "member"` (no-op if already a member).

### `DELETE /committees/:id/leave`

Self-service — removes the caller (and from the linked group chat).

### `GET /committees/:id/members`

Any current member can view.

### `PUT /committees/:id/members/:userId/role`

**Eboard only.** `{ "role": "chair" | "member" }` — auto-adds the target user as a committee member first if they aren't already one.

---

## Polls

Targeted the same way as Events/Announcements (`audience` array or `committee_id`, mutually exclusive). Voting is self-service; only eboard sees who voted for what — not anonymous to eboard.

### `GET /polls`

Visible polls for the caller (or all, if eboard), each with its options, aggregate vote counts per option, the caller's own selection (`my_option_ids`), and `total_votes`. `is_closed` reflects either a manual close or `expires_at` having passed — computed fresh on every read, not a stored flag, so it's always accurate even if nothing ever flips it in the DB.

### `POST /polls`

**Eboard only.**
```json
{
  "question": "Best meeting time?",
  "description": "...",
  "options": ["Monday 6pm", "Wednesday 7pm"],
  "multi_select": false,
  "audience": ["active", "chair"],
  "committee_id": null,
  "expires_at": "2026-08-01T23:00:00Z"
}
```
`options` needs at least 2. `expires_at` is optional — voting closes automatically at that time with no cron job involved (see `GET /polls` above). Omit for a poll that only closes when eboard manually closes it.

### `DELETE /polls/:id`

**Eboard only.**

### `PUT /polls/:id/close`

**Eboard only.** One-way manual close, independent of `expires_at`.

### `POST /polls/:id/vote`

Any shared-album-group member. `{ "option_ids": [3] }` — a single id unless `multi_select`. Replaces the caller's entire prior selection for this poll, so calling it again is also how you change your vote while it's still open.

### `GET /polls/:id/stats`

**Eboard only.** Same per-option counts as the list view, but with a `voters` array (name + id) per option.

---

## Photos & Albums

Members-only — the general shared album and eboard-created named albums alike. Public photos live in the separate **Homepage Photos** section below, not here. All routes require auth + membership in one of `active`/`chair`/`alumni`/`eboard`/`pledge`.

### `GET /photos?album_id=<id>`

Returns photos in the given album. Omit `album_id` for the general "Shared Album."

### `POST /photos`

Multipart upload, field `file` (images or video, 250MB limit). Optional form fields: `album_id` (puts it in a named album instead of the general shared album), `title`, `caption`.

### `GET /photos/:id/media`

Streams the photo/video from Immich (forwards `Range` headers for video seeking).

### `DELETE /photos/:id`

The uploader can always delete their own photo. **Eboard can delete any photo, in any album** — including the general Shared Album — as a real moderation power. (Previously scoped to only albums an eboard member personally created; broadened so eboard can act on any reported photo, not just their own albums.)

---

### `GET /albums`

Lists eboard-created named albums (e.g. "Spring Retreat 2026"). Any shared-album-group member can view.

### `POST /albums`

**Eboard only.** `{ "name": "...", "description": "..." }`

---

## Homepage Photos

The **public** chapter gallery — no auth on reads, anyone can view. Writes are eboard-only. Entirely separate from the member `photos` system above (different audience, different permission model).

### `GET /homepage-photos`

Public. Returns the gallery, ordered by `display_order`.

### `POST /homepage-photos`

**Eboard only.** Multipart upload, field `file`, plus `title`/`caption`.

### `POST /homepage-photos/register`

**Eboard only.** For when someone (e.g. SWE committee) already uploaded straight into the Immich UI — registers an existing asset by id instead of uploading a new one. `{ "immich_asset_id": "...", "media_type": "image", "title": "...", "caption": "..." }`. Validates the asset actually exists in Immich before saving (a typo'd id here previously caused repeated errors on view).

### `GET /homepage-photos/:id/media`

Public. Streams the photo/video.

### `PUT /homepage-photos/reorder`

**Eboard only.** `{ "ids": [3, 1, 2] }` — sets `display_order` to match array position.

### `DELETE /homepage-photos/:id`

**Eboard only.** Only unlists the photo from the gallery — does **not** delete the underlying Immich asset, since a registered asset might be reused elsewhere.

---

## Documents

An eboard-managed file library (bylaws, meeting minutes, course files) shown in the Files & Photos portal tab. Not Immich — arbitrary file types stored on ktp-api's own disk. View: any shared-album-group member. Writes: eboard only.

### `GET /documents/folders?parent_id=<id>`

Lists subfolders of `parent_id` (omit for the top level).

### `POST /documents/folders`

**Eboard only.** `{ "name": "...", "parent_id": null }` — folders nest to any depth.

### `DELETE /documents/folders/:id`

**Eboard only.** Cascades subfolders and document rows in the DB, and also walks a recursive query to delete every nested file from disk (disk space on this server is limited — DB cascade alone would leave orphaned files).

### `GET /documents?folder_id=<id>`

Lists documents in `folder_id` (omit for the top level).

### `POST /documents`

**Eboard only.** Multipart upload, field `file` (any file type, 50MB limit), form field `folder_id` (omit for the top level).

### `GET /documents/:id/download`

Streams the file with `Content-Disposition: attachment` and the original filename.

### `GET /documents/:id/preview`

Same file, but `Content-Disposition: inline` — used for the in-portal preview (images render inline, PDFs in an `<iframe>`) instead of forcing a download.

### `POST /documents/link`

**Eboard only.** An external hyperlink (Google Docs/Slides/Sheets, or any URL) shown alongside real files in the same folder tree — no file on disk. `{ "folder_id": null, "filename": "Meeting Notes", "url": "https://docs.google.com/..." }`. Every document row now carries `kind: "file" | "link"` — link rows have `url` set and no `mime_type`/`file_size`/`storage_path`.

### `DELETE /documents/:id`

**Eboard only.** Removes the DB row and, for `kind: "file"` rows, unlinks the file from disk (link rows have nothing on disk to clean up).

---

## Announcements

Eboard-only broadcast — one-way, no replies. Any shared-album-group member can view (filtered by audience). Same targeting shape as Events: `audience` is an array, or scope to one `committee_id` instead.

### `GET /announcements`

Returns announcements visible to the caller: `audience` empty/null (everyone), `audience` overlaps the caller's Authentik groups, or the caller belongs to the announcement's `committee_id`. Eboard sees every announcement unfiltered.

### `POST /announcements`

**Eboard only.** `{ "title": "...", "body": "...", "audience": ["active", "chair"], "committee_id": null }` — omit/empty `audience` (and no `committee_id`) to send to everyone.

### `PUT /announcements/:id`

**Eboard only.**

### `DELETE /announcements/:id`

**Eboard only.**

---

## Messages (Direct Messages)

Any member can message any other member — no membership list, unlike Group Chats below.

**Safety features, both here and in Group Chats:** message bodies are checked against a basic content filter (`services/contentFilter.js`, wraps the `bad-words` package — **pinned to v3**, v4 ships broken CJS packaging that can't be `require()`'d from this project) and rejected with `400` if flagged. Sending is rate-limited to 20 messages/minute/user (`middleware/rateLimit.js`, in-memory, no Redis) — a `429` means slow down, not a real error. Blocking (see Users above) also applies: `POST /messages` 403s if either party has blocked the other, and a blocked user's messages are filtered out of `GET /messages/conversations` and `GET /messages/conversations/:userId` for whoever did the blocking.

### `GET /messages/conversations`

Returns one entry per person the caller has exchanged messages with: their basic info, the last message, and an unread count — sorted by most recent activity.

### `GET /messages/conversations/:userId`

Full thread with that specific user, chronological.

### `POST /messages`

`{ "recipient_id": "uuid", "body": "..." }`

### `PUT /messages/conversations/:userId/read`

Marks every message from `:userId` to the caller as read.

---

## Group Chats

Eboard-created, named, with an assigned member list — access is gated by actual DB membership (`group_chat_members`), not a broad Authentik group, so most routes below check membership inline and return **403** if the caller isn't in that specific chat. Same content filter + rate limit as Direct Messages above. Blocking doesn't stop someone from posting in a shared group chat (they're a legitimate member) — it filters their messages out of `GET /group-chats/:id/messages` for whoever blocked them, same idea as the DM thread filtering.

### `GET /group-chats`

Chats the caller is a member of, with last message + unread count.

### `POST /group-chats`

**Eboard only.** `{ "name": "...", "member_ids": ["uuid", "uuid"] }` — the creator is automatically added too, so eboard isn't locked out of a chat they just made.

### `DELETE /group-chats/:id`

**Eboard only.** Cascades members, messages, and read markers.

### `GET /group-chats/:id/messages`

Full thread — **403** if the caller isn't a member of this chat.

### `POST /group-chats/:id/messages`

`{ "body": "..." }` — **403** if not a member.

### `PUT /group-chats/:id/read`

Marks the chat read for the caller.

### `GET /group-chats/:id/members`

Any current member can view the participant list.

### `POST /group-chats/:id/members`

**Eboard only.** `{ "user_id": "uuid" }`

### `DELETE /group-chats/:id/members/:userId`

**Eboard only.**

---

## Reports & Moderation

App Store safety requirement (reporting/blocking/moderation for an app with user-generated content). Submitting a report is self-service; reviewing the queue is eboard-only — there's no separate "moderator" Authentik group, this reuses eboard the same way every other admin-gated feature does.

### `POST /reports`

Any shared-album-group member. `{ "content_type": "user" | "message" | "group_message" | "photo", "content_id": "...", "reported_user_id": "uuid", "reason": "...", "explanation": "..." }`. `content_id` is required unless `content_type` is `"user"`. `reported_user_id` is trusted from the client (the UI reporting a message/photo/profile already knows who authored it) rather than re-derived server-side — content spans three different tables, not worth a per-type lookup just to double-check what the caller already knows.

### `GET /reports`

**Eboard only.** Optional `?status=open|resolved|dismissed`. Returns reporter/reported-user names already joined in.

### `PUT /reports/:id/status`

**Eboard only.** `{ "status": "resolved" | "dismissed" | "open", "moderator_response": "..." }` — stamps `resolved_by`/`resolved_at` whenever status moves away from `open`.

---

## Admin

All routes below require `requireAuth` + `requireGroup("eboard")`.

### `GET /admin/users`

Returns all users in the database.

### `PUT /admin/users/:authentikId/group`

`{ "group": "eboard" | "chair" | "active" | "alumni" | "pledge" }` — moves the target user to this group in Authentik itself (removing them from whichever other role group they're currently in there), then mirrors the change into `users.member_group` immediately. See [API Overview: Eboard — changing a member's group](./overview.md#eboard-changing-a-members-group) for the full mechanism and required Authentik setup.

---

## Webhooks

### `POST /webhooks/authentik`

Called by Authentik's notification transport when a user is deleted — **fixed and confirmed working in production.**

**Headers:**
```
X-Authentik-Token: <WEBHOOK_SECRET>
```

**Request body (Authentik format):**
```json
{
  "action": "model_deleted",
  "model": {
    "model_name": "user",
    "pk": 42
  }
}
```

Deleting a user in Authentik now correctly removes their row from `users` — no manual DB cleanup needed. The root causes of the long-standing "nothing happens" issue were entirely on the Authentik side, not this endpoint:
1. The Notification Rule needs a Group **or** "Send notification to event user" set — Authentik silently no-ops a Rule with neither.
2. The bound Event Matcher Policy's **App** field compares against the Event's own `event.app`, not `event.context.model.app` — leave App blank; Action=`Model Deleted` + Model=`User (authentik_core)` alone is correct and sufficient.

If this breaks again, `docker logs worker` on the Authentik host (not this API's logs) shows structured `policy_execution` entries with a per-criterion pass/fail — the fastest way to see which criterion is silently failing.

---

## Error Responses

| Status | Meaning |
|--------|---------|
| `400` | Bad request — missing or invalid fields |
| `401` | Missing or invalid Bearer token |
| `403` | Authenticated but not authorized (wrong group) |
| `404` | Resource not found |
| `500` | Internal server error |

All error responses use the format:
```json
{ "message": "Error description" }
```