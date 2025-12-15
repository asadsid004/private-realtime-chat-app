# Real-time Private Chat App

A private, self-destructing chat room built with **Next.js (App Router)**, **React 19**, **Elysia** (API), **Upstash Redis** (storage + TTL), and **Upstash Realtime** (pub/sub).

The app creates a short-lived room, restricts it to **max 2 participants**, persists messages in Redis, and broadcasts realtime events so both participants stay in sync.

---

## Table of contents

- [Overview](#overview)
- [Features](#features)
- [Tech stack](#tech-stack)
- [Project structure](#project-structure)
- [How it works (end-to-end)](#how-it-works-end-to-end)
- [API (HTTP)](#api-http)
- [Realtime events](#realtime-events)
- [Data model (Redis keys)](#data-model-redis-keys)
- [Environment variables](#environment-variables)
- [Local development](#local-development)
- [Production notes](#production-notes)
- [Troubleshooting](#troubleshooting)
- [Security notes](#security-notes)

---

## Overview

This project is a minimal “secure room” chat:

- A user lands on `/` and generates a local anonymous username.
- Clicking **Create Secure Room** calls an API endpoint that creates a room ID and stores room metadata in Redis with a TTL.
- Navigating to `/room/[roomId]` triggers a server-side access gate that:
  - Creates/reads a cookie auth token
  - Ensures **no more than 2 users** can join
  - Blocks/redirects if the room is missing/expired
- Inside the room:
  - The UI polls the room TTL (countdown)
  - Fetches the message history from Redis
  - Subscribes to realtime events:
    - `chat.message` → refetch messages
    - `chat.destroy` → redirect back to lobby

---

## Features

- **Private rooms**: share the room URL with one other person.
- **Max 2 participants**: enforced at request-time.
- **Self-destructing rooms**:
  - Rooms have a TTL (currently **10 minutes**) stored in Redis.
  - Countdown is shown in the room UI.
- **Realtime updates** via Upstash Realtime.
- **Message history** stored in Redis lists.
- **Manual “Destroy now”**: deletes room + metadata + history, and notifies both clients.

---

## Tech stack

- **Framework**: Next.js 16 (App Router)
- **UI**: React 19, Tailwind CSS v4
- **Data fetching**: @tanstack/react-query
- **API layer**: Elysia (mounted under Next.js route handler)
- **Storage**: Upstash Redis
- **Realtime**: Upstash Realtime
- **Types & validation**: TypeScript, Zod
- **IDs**: nanoid

---

## Project structure

Key files/folders:

- `src/app/page.tsx`
  - Lobby UI (create room, show errors)
- `src/app/room/[roomId]/page.tsx`
  - Room UI (messages, TTL countdown, destroy button, realtime subscription)
- `src/app/api/[[...slugs]]/route.ts`
  - Elysia app mounted at `/api/*` (rooms + messages)
- `src/app/api/[[...slugs]]/auth.ts`
  - Elysia middleware: validates `roomId` and `x-auth-token` cookie
- `src/app/api/realtime/route.ts`
  - Upstash Realtime handler endpoint (`/api/realtime`)
- `src/lib/redis.ts`
  - Redis client: `Redis.fromEnv()`
- `src/lib/realtime.ts`
  - Realtime schema + `new Realtime({ schema, redis })`
- `src/lib/realtime-client.ts`
  - Client hook: `createRealtime().useRealtime`
- `src/lib/client.ts`
  - Eden treaty client for calling `/api/*` from the browser
- `src/proxy.ts`
  - Request gate for `/room/*` (room existence, capacity, auth cookie)
- `src/components/providers.tsx`
  - React Query + Upstash Realtime providers

---

## How it works (end-to-end)

### 1) Creating a room

UI (`/`) calls:

- `POST /api/rooms/create`

Server:

- Generates `roomId` via `nanoid()`
- Stores room metadata at `meta:{roomId}`
- Sets expiry for `meta:{roomId}` to `ROOM_TTL_SECONDS` (currently 10 minutes)

### 2) Joining a room (access control)

When the browser requests `/room/{roomId}`, the server-side proxy logic (`src/proxy.ts`) applies:

- If the room meta key is missing → redirect to `/?error=room-not-found`
- If there’s an existing `x-auth-token` cookie that’s already in `meta:{roomId}.connected` → allow
- If `connected.length >= 2` → redirect to `/?error=room-full`
- Otherwise:
  - Creates a new `x-auth-token` (random)
  - Adds it to `meta:{roomId}.connected`
  - Proceeds to the room

This token is later used by the API auth middleware to restrict access to TTL/messages/destroy endpoints.

### 3) Sending messages

UI calls:

- `POST /api/messages?roomId={roomId}`

Server:

- Validates the token is connected for this room
- Pushes the message into `messages:{roomId}` list
- Emits realtime event `chat.message`
- Syncs expirations of message/history keys to match the room remaining TTL

### 4) Realtime sync

Clients subscribe to a channel named by the `roomId`.

- On `chat.message`: UI refetches messages.
- On `chat.destroy`: UI navigates to `/?destroyed=true`.

### 5) Room TTL countdown

Room UI:

- Queries `GET /api/rooms/ttl?roomId={roomId}`
- Starts a local countdown timer based on returned TTL
- When it reaches `0` → redirects back to `/?destroyed=true`

Note: the “authoritative” TTL lives in Redis. The client countdown is a UI convenience.

---

## API (HTTP)

Base prefix: `/api`

### Rooms

#### Create room

- **Method**: `POST`
- **Path**: `/api/rooms/create`
- **Auth**: none
- **Response**:

```json
{ "roomId": "..." }
```

#### Get room TTL

- **Method**: `GET`
- **Path**: `/api/rooms/ttl`
- **Query**:
  - `roomId` (string)
- **Auth**: requires `x-auth-token` cookie that is connected to the room
- **Response**:

```json
{ "ttl": 123 }
```

#### Destroy room

- **Method**: `DELETE`
- **Path**: `/api/rooms`
- **Query**:
  - `roomId` (string)
- **Auth**: requires `x-auth-token` cookie
- **Behavior**:
  - Emits realtime `chat.destroy`
  - Deletes room-related Redis keys

### Messages

#### Send message

- **Method**: `POST`
- **Path**: `/api/messages`
- **Query**:
  - `roomId` (string)
- **Body**:

```json
{ "sender": "anonymous-cat-abc12", "text": "hello" }
```

- **Auth**: requires `x-auth-token` cookie

#### List messages

- **Method**: `GET`
- **Path**: `/api/messages`
- **Query**:
  - `roomId` (string)
- **Auth**: requires `x-auth-token` cookie
- **Response**:

```json
{
  "messages": [
    {
      "id": "...",
      "sender": "...",
      "text": "...",
      "timestamp": 1730000000000,
      "roomId": "...",
      "token": "... (only returned to the sender)"
    }
  ]
}
```

---

## Realtime events

- **Channel**: `roomId`
- **Events**:
  - `chat.message`
  - `chat.destroy`

Schema is defined in `src/lib/realtime.ts` using Zod and inferred into `RealtimeEvents`.

### `chat.message`

Payload:

```ts
{
  id: string;
  sender: string;
  text: string;
  timestamp: number;
  roomId: string;
  token?: string;
}
```

### `chat.destroy`

Payload:

```ts
{
  isDestroyed: true;
}
```

---

## Data model (Redis keys)

The Redis keyspace is organized by room:

- `meta:{roomId}` (hash)

  - `connected`: array of tokens (max 2)
  - `createdAt`: timestamp (ms)
  - **TTL**: set to `ROOM_TTL_SECONDS`

- `messages:{roomId}` (list)

  - list entries are message objects
  - **TTL**: aligned to room remaining TTL after each message

- `{roomId}`
  - used as an additional key that gets expired alongside messages (housekeeping)

When the room is destroyed, the API deletes:

- `{roomId}`
- `meta:{roomId}`
- `messages:{roomId}`

---

## Environment variables

This project uses **Upstash Redis** via `Redis.fromEnv()`. You must set the standard Upstash Redis env vars.

Create a local `.env` file (not committed).

Required:

- `UPSTASH_REDIS_REST_URL`
- `UPSTASH_REDIS_REST_TOKEN`

Recommended / implied:

- `NODE_ENV`
  - Used to decide whether auth cookies are set with `secure: true`.

If you deploy, also ensure your Upstash Realtime configuration is set up according to Upstash docs (the `/api/realtime` route is already present).

---

## Local development

### Prerequisites

- Node.js (compatible with Next.js 16)
- Bun (recommended, since the repo includes `bun.lock`)
- An Upstash Redis database

### Install

```bash
bun install
```

### Configure env

Create `.env`:

```bash
UPSTASH_REDIS_REST_URL=...
UPSTASH_REDIS_REST_TOKEN=...
```

### Run

```bash
bun dev
```

App:

- `http://localhost:3000`

---

## Production notes

- **Cookies**: `x-auth-token` is `httpOnly`, `sameSite: strict`, and `secure` only in production.
- **Workspace root warning**: If Next.js warns about multiple lockfiles, consider removing extra lockfiles or set the appropriate Next.js root config.
- **Eden client base URL**: `src/lib/client.ts` currently points to `localhost:3000`. For production deployments, you typically want to:
  - Use a relative URL, or
  - Use an env-based base URL.

---

## Troubleshooting

### "ROOM NOT FOUND"

- The room likely expired (TTL reached 0), or the roomId is invalid.
- Confirm Redis connectivity and env vars.

### "ROOM FULL"

- Two tokens are already connected.
- Clear cookies for the site if you’re testing with multiple tabs/profiles.

### Realtime not updating

- Ensure `/api/realtime` is reachable.
- Verify Upstash Realtime is configured correctly.
- Check browser console for subscription errors.

### TTL countdown looks wrong

- The UI countdown is derived from the latest TTL fetch.
- Refresh to re-sync with Redis TTL.

---

## Security notes

- This is a lightweight “privacy” model: it’s designed to limit participation via a cookie token and a max capacity gate.
- Messages are not end-to-end encrypted.
- Anyone with the room URL _and_ an accepted token could access the room until it expires.

---

## Scripts

- `bun dev` → start dev server
- `bun run build` → production build
- `bun start` → start production server
- `bun run lint` → lint
