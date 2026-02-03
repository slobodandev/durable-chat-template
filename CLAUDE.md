# CLAUDE.md

## Project Overview

Durable Chat Template — a real-time chat application built on **Cloudflare Workers**, **Durable Objects**, and **PartyKit**. Users create or join chat rooms via unique URL-based room IDs and communicate over WebSockets. Chat history is persisted using Durable Object SQL Storage.

## Tech Stack

- **Runtime:** Cloudflare Workers + Durable Objects
- **Server framework:** PartyServer (WebSocket management on Durable Objects)
- **Client framework:** React 18, React Router 7, PartySocket (WebSocket client)
- **Language:** TypeScript (strict mode)
- **Bundler:** esbuild (client), Wrangler (server)
- **CSS:** Skeleton CSS framework + custom styles
- **IDs:** nanoid

## Directory Structure

```
src/
  shared.ts              # Shared types (ChatMessage, Message) and constants
  client/
    index.tsx            # React app: routing, WebSocket connection, chat UI
    tsconfig.json        # Client TS config (jsx: react, lib: DOM)
  server/
    index.ts             # Chat Durable Object class + Worker fetch handler
    tsconfig.json        # Server TS config
    worker-configuration.d.ts  # Auto-generated Wrangler types (do not edit)
public/
  index.html             # HTML shell
  styles.css             # Custom chat styles
  css/                   # Skeleton CSS framework files
  dist/                  # Build output for client bundle (gitignored)
```

## Commands

```bash
npm install              # Install dependencies
npm run dev              # Start local dev server (wrangler dev, localhost:8787)
npm run check            # Type-check client + server, then dry-run deploy
npm run deploy           # Deploy to Cloudflare Workers
npm run cf-typegen       # Regenerate worker-configuration.d.ts types
```

## Architecture

### Message Protocol

Defined in `src/shared.ts` as a discriminated union type `Message`:
- `"add"` — new message from a user
- `"update"` — update to an existing message
- `"all"` — full message history sync (sent on connection)

### Server (`src/server/index.ts`)

The `Chat` class extends `Server<Env>` from PartyServer with hibernate enabled:
- `onStart()` — creates SQL table if needed, loads all messages into memory
- `onConnect()` — sends full message history to new connections
- `onMessage()` — broadcasts raw message to all clients, persists via `saveMessage()`
- `saveMessage()` — upserts to both in-memory array and SQL storage

The default export routes requests through `routePartykitRequest()`, falling back to `env.ASSETS.fetch()` for static files.

### Client (`src/client/index.tsx`)

Single `App` component using `usePartySocket` hook:
- Connects to the `"chat"` party with the current room ID from URL params
- Handles incoming messages: add (deduplicated by ID), update, all (full replace)
- Optimistic UI: messages are added locally before server confirmation
- User identity: randomly assigned from a predefined names list on mount
- Routing: `/` redirects to a random room, `/:room` is the chat view

### Data Flow

1. Client submits message -> adds to local state (optimistic) -> sends via WebSocket
2. Server receives -> broadcasts to all connections -> persists to SQL
3. All clients receive broadcast -> reconcile with local state (deduplicate by ID)

## Key Conventions

- **TypeScript strict mode** across all code; separate tsconfig files for client and server
- **Shared types** live in `src/shared.ts` — imported by both client and server
- **Discriminated unions** for the WebSocket message protocol (switch on `message.type`)
- **PascalCase** for classes/components, **camelCase** for functions/variables
- **`satisfies`** keyword used for type-safe message construction
- **No test framework** is configured; validation is done via `npm run check` (type-check + dry-run deploy)
- **`worker-configuration.d.ts`** is auto-generated — regenerate with `npm run cf-typegen`, do not edit manually

## Wrangler Configuration (`wrangler.json`)

- Entry point: `src/server/index.ts`
- Assets served from `./public` with SPA fallback (`not_found_handling: "single-page-application"`)
- Build step: esbuild bundles client to `public/dist/`
- Durable Object binding: `Chat` class
- SQL migration tag `v1` creates the `Chat` SQLite class
- Observability and source map upload enabled

## Common Pitfalls

- The SQL queries in `saveMessage()` use string interpolation — `message.content` is JSON-stringified but `user`, `role`, and `id` are interpolated directly. Keep this in mind when modifying the message schema.
- The client `onMessage` handler uses a closure over `messages` state for `findIndex`, which may reference stale state. The `setMessages` calls correctly use functional updaters.
- `public/dist/` is a build artifact created by esbuild during `wrangler dev` or `wrangler deploy` — it should not be committed.
