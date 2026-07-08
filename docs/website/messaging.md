---
sidebar_position: 3
---

# Messaging

The **Messages** portal tab has three parts, each solving a genuinely different communication pattern. See [API: Announcements](../api/endpoints.md#announcements), [Messages](../api/endpoints.md#messages-direct-messages), and [Group Chats](../api/endpoints.md#group-chats) for the backend routes.

All three use **short-interval polling** (roughly every 5-10 seconds while a relevant view is open) rather than real-time delivery (WebSockets) — this matches every other feature in the app (plain REST + refetch) and needed no new infrastructure. Genuinely instant delivery was considered and explicitly deferred; polling is imperceptibly slow for a chapter this size.

---

## Announcements

One-way, eboard-only broadcasts — no replies. Eboard picks an audience per post: everyone, or one specific group (active/chair/alumni/eboard/pledge). Everyone else just reads them, filtered server-side to what their own group is allowed to see.

---

## Direct Messages

Any member can message any other member — no membership list or approval needed, mirroring the same "everyone's in one shared space together" model used for photos and documents. A conversation is just the two people's message history; there's no separate "start a conversation" step beyond picking someone and sending a message.

Each conversation shows an unread badge and the last message preview. Opening a conversation marks it read.

---

## Group Chats

The one messaging mode that's **not** open to everyone by default: eboard creates a named group chat (e.g. "Rush Committee") and assigns specific members to it. Only assigned members can see or post in it — access is checked against an actual membership list in the database, not a broad Authentik group like everything else in the app.

- **Eboard only**: create a chat, delete a chat, add/remove members (editable any time after creation, not just at creation).
- **Any assigned member**: post messages — this is a real multi-party thread, not a broadcast, so message bubbles show the sender's name (unlike DMs, where it's just "you" vs. "them").

---

## Starting a conversation from the Directory

The Member Directory's profile view has a **Message** button that jumps straight to a DM thread with that person (`/member/messages?with=<id>`, or the equivalent path for whichever portal you're in) — see [Profiles & Directory](./profiles-and-directory.md).
