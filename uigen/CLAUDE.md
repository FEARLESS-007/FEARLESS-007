# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# Initial setup (install deps + Prisma generate + migrate)
npm run setup

# Development server (Turbopack)
npm run dev

# Build
npm run build

# Lint
npm run lint

# Run all tests
npm test

# Run a single test file
npx vitest run src/components/chat/__tests__/ChatInterface.test.tsx

# Reset database
npm run db:reset

# After schema changes
npx prisma migrate dev
npx prisma generate
```

## Architecture

UIGen is a Next.js 15 App Router app where users chat with Claude AI to generate React components, which are previewed live in the browser without writing files to disk.

### Core Data Flow

1. **Chat** (`/api/chat`) — User sends a message → the server calls Claude (claude-haiku-4-5 via `@ai-sdk/anthropic`) with two tools: `str_replace_editor` and `file_manager`. Claude uses these tools to create/edit files in the virtual file system.
2. **Virtual File System** (`src/lib/file-system.ts`) — An in-memory tree (`VirtualFileSystem` class) that holds all generated files. It serializes to plain JSON for transport and storage.
3. **Live Preview** (`src/components/preview/PreviewFrame.tsx`) — On every VFS change, JSX/TSX files are compiled client-side via `@babel/standalone`, turned into Blob URLs, assembled into an ES module import map, and injected into a sandboxed `<iframe>` via `srcdoc`. Third-party npm imports resolve to `esm.sh`.

### State Management via React Contexts

- `FileSystemContext` (`src/lib/contexts/file-system-context.tsx`) — Wraps `VirtualFileSystem`, exposes CRUD operations, tracks `refreshTrigger` (a counter) to signal re-renders. Also processes tool call callbacks from the AI stream to apply file mutations.
- `ChatContext` (`src/lib/contexts/chat-context.tsx`) — Wraps Vercel AI SDK's `useChat`, serializes the current VFS state into the request body on every message send, and forwards `onToolCall` events to `FileSystemContext.handleToolCall`.

### Key Files

| Path | Purpose |
|---|---|
| `src/app/api/chat/route.ts` | Streaming chat endpoint; reconstructs VFS from request, calls Claude, persists to DB on finish |
| `src/lib/file-system.ts` | `VirtualFileSystem` class — all file operations |
| `src/lib/transform/jsx-transformer.ts` | Babel transform, import map creation, preview HTML generation |
| `src/lib/provider.ts` | `getLanguageModel()` — returns real Claude or `MockLanguageModel` when no API key |
| `src/lib/auth.ts` | JWT sessions via `jose`, stored in httpOnly cookies |
| `src/lib/tools/str-replace.ts` | AI tool: view/create/str_replace/insert file operations |
| `src/lib/tools/file-manager.ts` | AI tool: rename/delete operations |
| `src/lib/prompts/generation.tsx` | System prompt for component generation |
| `src/app/main-content.tsx` | Root layout: resizable chat + preview/code panels |

### Database

Prisma with SQLite (`prisma/dev.db`). Two models:
- `User` — email/password (bcrypt), owns projects
- `Project` — stores `messages` (JSON array) and `data` (serialized VFS JSON); `userId` is optional to allow anonymous projects

Prisma client output is at `src/generated/prisma` (not the default location).

### Auth

Custom JWT auth (no NextAuth). `src/lib/auth.ts` handles session creation/verification. `src/middleware.ts` runs on every request. The `use-auth` hook (`src/hooks/use-auth.ts`) manages client-side auth state. Anonymous users can work without signing in; `src/lib/anon-work-tracker.ts` stores their work in localStorage so it can be claimed on sign-up.

### Mock Provider

When `ANTHROPIC_API_KEY` is not set, `getLanguageModel()` returns `MockLanguageModel` which streams pre-canned component code (counter, form, or card) to allow local development without an API key.

### Testing

Vitest + jsdom + React Testing Library. Tests live in `__tests__` folders co-located with source. The vitest config uses `vite-tsconfig-paths` so `@/` path aliases work in tests.
