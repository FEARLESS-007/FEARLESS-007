# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Location

All application code is in the `uigen/` subdirectory. Run all commands from there.

## Commands

```bash
# Initial setup (install deps + generate Prisma client + run migrations)
npm run setup

# Development server (Turbopack)
npm run dev

# Build for production
npm run build

# Lint
npm run lint

# Run all tests
npm test

# Run a single test file
npx vitest run src/components/chat/__tests__/ChatInterface.test.tsx

# Reset database
npm run db:reset

# Regenerate Prisma client after schema changes
npx prisma generate

# Run new migrations after schema changes
npx prisma migrate dev
```

## Architecture

### Overview

UIGen is an AI-powered React component generator. Users describe a component in chat; Claude generates files into a **virtual file system** (never written to disk); a live preview renders the result in an iframe.

### Data Flow

1. User sends a chat message from `ChatProvider` (`src/lib/contexts/chat-context.tsx`)
2. The current virtual file system is serialized and sent with the message to `POST /api/chat` (`src/app/api/chat/route.ts`)
3. The API route deserializes the file system, calls Claude (or `MockLanguageModel` if no API key) via Vercel AI SDK with two tools: `str_replace_editor` and `file_manager`
4. Tool calls stream back to the client and are handled by `FileSystemContext` (`src/lib/contexts/file-system-context.tsx`), which updates the in-memory `VirtualFileSystem`
5. `PreviewFrame` (`src/components/preview/PreviewFrame.tsx`) watches `refreshTrigger` and re-renders the iframe whenever files change
6. For authenticated users with a `projectId`, the final messages and file state are persisted to SQLite via Prisma

### Virtual File System

`VirtualFileSystem` (`src/lib/file-system.ts`) is an in-memory tree of `FileNode` objects. Files are stored in a flat `Map<string, FileNode>` keyed by path. It implements editor-style commands (`viewFile`, `replaceInFile`, `insertInFile`, `createFileWithParents`) that the AI tools delegate to.

Two AI tools wrap it:
- `str_replace_editor` (`src/lib/tools/str-replace.ts`): `create`, `str_replace`, `insert`, `view` commands
- `file_manager` (`src/lib/tools/file-manager.ts`): `rename`, `delete` commands

### Preview Pipeline

`createImportMap` in `src/lib/transform/jsx-transformer.ts` transforms all `.js/.jsx/.ts/.tsx` files using **Babel standalone**, creates blob URLs for each, and builds an ES module import map. Third-party packages (non-relative imports) resolve to `https://esm.sh/<package>`. The import map and an HTML shell are injected into the sandboxed iframe as `srcdoc`. Tailwind CSS is loaded via CDN in the preview iframe.

### AI Provider

`getLanguageModel()` in `src/lib/provider.ts` returns `anthropic("claude-haiku-4-5")` when `ANTHROPIC_API_KEY` is set, otherwise a `MockLanguageModel` that returns static component code. The mock avoids API calls during development without a key.

### Auth

JWT-based auth stored in an httpOnly cookie (`auth-token`). `src/lib/auth.ts` handles session creation/verification using `jose`. Anonymous users can work without signing in; their work is tracked in `src/lib/anon-work-tracker.ts` and can be linked to an account on sign-up.

### Database

Prisma with SQLite (`prisma/dev.db`). The `Project` model stores `messages` (JSON string) and `data` (serialized virtual file system as JSON). Schema is in `prisma/schema.prisma`.

### Testing

Vitest with jsdom and `@testing-library/react`. Tests live in `__tests__/` directories co-located with the code they test.

## Environment Variables

`.env` in `uigen/`:
- `ANTHROPIC_API_KEY` — required for real AI generation; omit to use the mock provider
- `JWT_SECRET` — defaults to `"development-secret-key"` if not set
