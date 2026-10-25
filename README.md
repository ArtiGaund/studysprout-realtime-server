# StudySprout Realtime Server ⚡

> The real-time collaboration backbone for [StudySprout](https://studysprouts.in/): Socket.io + Yjs powered live editing, presence, and generation locking.

[![Node.js](https://img.shields.io/badge/Node.js-Express-339933?logo=node.js)](https://nodejs.org/)
[![Socket.io](https://img.shields.io/badge/Socket.io-realtime-010101?logo=socket.io)](https://socket.io/)
[![Yjs](https://img.shields.io/badge/Yjs-CRDT-yellow)](https://github.com/yjs/yjs)

**Live app this powers:** [studysprouts.in](https://studysprouts.in/)
**Main repo:** [`studysprout`](https://github.com/ArtiGaund/studysprout) — the Next.js app, API routes, and background workers this server works with
**Rate limiter:** [`rate-limiter`](https://github.com/ArtiGaund/rate-limiter) — standalone Java service that rate-limits Gemini calls (this server isn't involved in that check)

---

## What This Service Does

A standalone Express + Socket.io server, deployed independently from the main StudySprout app, responsible for everything that needs a persistent connection:

1. **Real-time presence**: who's in a workspace, and who's in a specific file
2. **Collaborative document editing**: Yjs CRDT sync over WebSockets, so multiple users edit one file at once with no conflicts
3. **AI generation locking**: stops two users from triggering flashcard generation for the same resource at once
4. **Title-editing presence**: stops two users from renaming the same folder, file, or flashcard set at the same time
5. **Cross-service event relay**: receives HTTP calls from the Next.js app and the background worker and re-broadcasts them to connected browsers as live socket events

It's a separate deployment because the main app runs on Vercel (serverless), which can't hold WebSocket connections open. This server stays alive continuously on Render.

| | Main app (`studysprout`, Vercel) | This server (Render) |
|---|---|---|
| Execution model | Request/response, serverless | Long-running process |
| Can hold open WebSocket connections | No | Yes |

---

## Architecture

```
┌─────────────────────┐                    ┌────────────────────────┐
│  Browser (client)   │◄──── WebSocket ──► │  This server           │
│  Yjs doc + Socket.io│                    │  Socket.io + Yjs mirror│
└─────────────────────┘                    └──────────┬─────────────┘
                                                      │
                       ┌──────────────────────────────┼─────────────────────┐
                       │                              │                     │
                  HTTP POST /emit/*              enqueue job          verify-membership
                  (from app + worker)         (file-sync-queue)       call to main app
                       │                              │
                       ▼                              ▼
          ┌─────────────────────┐          ┌───────────────────┐
          │ Next.js API routes  │          │   Shared Redis    │
          │ + background worker │          │   (BullMQ)        │
          └─────────────────────┘          └─────────┬─────────┘
                                                     │ consumed by
                                                     ▼
                                          ┌──────────────────────┐
                                          │  Background workers  │
                                          │  (main repo, Render) │
                                          │  write to MongoDB    │
                                          └──────────────────────┘
```

**Communication paths**

1. **Browser ↔ this server (WebSocket)**: live Yjs updates, presence, awareness (cursors), and generation-lock requests
2. **This server → Redis (BullMQ)**: Yjs edits are debounced for 2 seconds, then enqueued as `persist-file` jobs on the shared `file-sync-queue`. The main repo's worker consumes them and writes the final state to MongoDB
3. **Main app and worker → this server (HTTP)**: both call this server's `/emit/*` endpoints whenever something should be pushed live to browsers (a file finishing PDF processing, a flashcard set completing, a workspace invitation arriving)
4. **This server → main app (HTTP)**: calls the app's internal verify endpoint with a shared secret (`x-internal-secret`) to check workspace access

---

## Core Mechanisms

### 1. Socket Authentication

Each connection is authenticated by decoding the same NextAuth JWT the main app issues:

```typescript
const decoded = await decode({ token, secret: config.NEXTAUTH_SECRET! });
socket.data.user = decoded;
```

`NEXTAUTH_SECRET` here must be byte-identical to the main app's value. If it differs, sockets still connect (connections without a valid token are allowed through as unauthenticated) but never join their personal notification room, so targeted events like workspace invitations only show up after a refresh. This is the most common cause of "sockets work but this one feature doesn't."

### 2. Presence Tracking (two layers)

- **Workspace presence**: `workspaceId → userId → Set<socketId>`. Tracks who's active in a workspace and supports multiple tabs per user without flicker
- **File presence**: `fileId → userId → { username, avatarUrl, sockets }`. Tracks who's viewing or editing a specific file, broadcast to everyone else in that file's room

### 3. Yjs Collaborative Editing

The server keeps an in-memory mirror `Y.Doc` per active file. When a client sends a raw Yjs update:

```typescript
socket.on("file:update-raw", ({ fileId, update }) => {
  socket.to(`file:${fileId}`).emit("file:update-raw", update);  // live relay
  Y.applyUpdate(doc, new Uint8Array(update));                    // update server mirror
  // debounce 2s, then enqueue a persist-file job to Redis
});
```

Updates are relayed instantly and persisted on a separate debounce, so typing never hits the database on every keystroke while the live experience stays immediate. Awareness updates (cursor position, user color) go out with `socket.volatile`, best effort with no delivery guarantee, since an occasional dropped cursor packet doesn't matter.

### 4. AI Generation Locks (hierarchical)

```typescript
socket.on("request_gen_start", ({ resourceId, parentId, workspaceId }) => {
  // 1. Is this exact resource already locked?
  // 2. Is the parent folder generating? (blocks child-file generation)
  // 3. Is any child file generating? (blocks folder-level generation)
  // → grant or deny the lock
});
```

Locks auto-release after 5 minutes as a safety net if a client disconnects mid-generation ("ghost lock protection"). Every lock change is broadcast to the whole workspace so progress bars stay in sync for everyone.

This lock prevents *concurrent* generation of the same resource. It's separate from the rate limiter, which limits *repeated* requests over time.

### 5. Title-Editing Presence

A three-event flow (`presence:remote-editing-start` / `-typing` / `-stop`) broadcasts when someone begins editing a title, so other clients can show "someone's renaming this" instead of two people racing to submit conflicting names.

### 6. HTTP Emit Endpoints (`/emit/*`)

Internal POST endpoints the app and worker use to push events to connected browsers: workspace tree updates, member changes, file updates, generation progress, flashcard set lifecycle (created, updated, deleted, regenerated, outdated), card regeneration, file stats, activity entries, and workspace invitations and responses. These let a background worker, which has no WebSocket of its own, still trigger an instant UI update. There is also an endpoint that applies a captured Inbox block into the live Yjs document, which is how Clipper merges stay consistent with live edits.

---

## Socket Events

| Area | Events |
|---|---|
| Workspace | `workspace:join`, `workspace:leave`, `user:rejoin` |
| Generation locks | `request_gen_start`, `report_progress`, `request_gen_end`, `get_workspace_locks` |
| Title editing | `presence:remote-editing-start`, `presence:remote-editing-typing`, `presence:remote-editing-stop` |
| Files | `file:join`, `file:update-raw`, `file:awareness-update`, `file:leave` |

`scripts/socket-audit.sh` scans the repo for HTTP routes, emit calls, and listeners, and was used to find duplicate emissions.

---

## Limitations (by design, for now)

- **Single instance.** Presence maps, generation locks, and the Yjs mirror documents live in process memory. Running more than one instance would need the Socket.io Redis adapter and shared lock state
- **Locks don't survive a restart.** They clear when the process restarts; the 5-minute auto-release covers the other failure case (a client that disconnects mid-generation)
- **Hardening in progress:** a shared-secret check on `/emit/*`, and workspace-membership verification when a socket joins a workspace or file room

---

## Tech Stack

- Express (HTTP server and `/emit/*` endpoints)
- Socket.io (WebSocket layer)
- Yjs (CRDT document sync)
- BullMQ + ioredis (producer for `file-sync-queue`)
- `next-auth/jwt` (shared JWT decoding with the main app)
- TypeScript, run with `tsx` in development and compiled with `tsc` for production

---

## Environment Variables

```bash
REDIS_URL=...                       # same Redis the main repo's worker consumes
NEXTAUTH_SECRET=...                 # must exactly match the main app's value
NEXT_PUBLIC_APP_URL=https://studysprouts.in   # allowed origin for the HTTP CORS config
MAIN_API_URL=...                    # base URL of the main app (membership checks)
INTERNAL_SECRET=...                 # shared secret with the main app's internal API
NEXTAUTH_URL=...                    # declared in config
MONGO_URL=...                       # declared in config
```

---

## Getting Started

```bash
git clone https://github.com/ArtiGaund/studysprout-realtime-server.git
cd studysprout-realtime-server
npm install
npm run dev
```

Runs on `http://localhost:4000`. You also need Redis running, plus the [main `studysprout` repo](https://github.com/ArtiGaund/studysprout) with its `NEXT_PUBLIC_REALTIME_URL` pointed at this server, for the full app to work.

## Deployment

Deployed on Render as a Node web service from this repo (`npm run build`, then `npm start`). Set the environment variables above in Render's dashboard.

---

## What I Learned Building This

- Designing socket authentication that shares a JWT secret across two independently deployed services, and the silent failure mode when the secret drifts out of sync
- Building real concurrency primitives for multi-user collaboration: hierarchical locking with crash-safety timeouts, not just event relay
- The debounce-then-queue pattern for persisting high-frequency live edits without hammering the database
- Debugging a Yjs echo-back bug where a client resending its own hydrated document state as a live delta corrupted merged state for other clients, traced to one redundant emit in the editor's hydration effect
- Coordinating one Redis connection format (`REDIS_URL`) across services that originally used inconsistent configs

---

## Related

- **Main application**: [`studysprout`](https://github.com/ArtiGaund/studysprout)
- **Rate limiter**: [`rate-limiter`](https://github.com/ArtiGaund/rate-limiter)
- **Live demo**: [studysprouts.in](https://studysprouts.in/)

## License
This project is not open source. The code is publicly viewable for portfolio purposes, but all rights are reserved. Please do not copy, reuse, or redistribute without permission.

## Contact
**Arti Gaund** — [LinkedIn](https://linkedin.com/in/artigaund) · [Email](mailto:artigaund2210@gmail.com) · [GitHub](https://github.com/ArtiGaund)