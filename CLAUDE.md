# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# Development
npm run dev          # Start dev server with Turbopack
npm run build        # Production build
npm run lint         # ESLint

# Testing
npm test             # Run all tests (Vitest)
npx vitest run src/lib/__tests__/foo.test.ts  # Run a single test file

# Database
npm run setup        # Install deps + generate Prisma client + migrate
npm run db:reset     # Reset database (destructive)
npx prisma migrate dev   # Apply migrations
npx prisma generate      # Regenerate client after schema changes
```

## Environment

Copy `.env.example` to `.env` and set `ANTHROPIC_API_KEY`. If not set, the app uses a mock provider that returns placeholder component code.

## Architecture

**UIGen** is an AI-powered React component generator. Users describe components in natural language; Claude generates them live with a virtual file system and browser preview.

### AI Integration (`src/app/api/chat/route.ts`)

The streaming chat endpoint uses the Vercel AI SDK (`useChat` hook on the client, `streamText` on the server). Claude is given two tools:
- `str_replace_editor` (`src/lib/tools/str-replace.ts`) — create/edit/view files via string replacement
- `file_manager` (`src/lib/tools/file-manager.ts`) — rename/delete files

The system prompt lives in `src/lib/prompts/generation.tsx`.

### Virtual File System

`src/lib/file-system.ts` defines a `VirtualFileSystem` class — all project files live in memory (no disk writes). The file system is serialized to JSON and persisted in the Prisma `Project.files` column. `src/lib/contexts/file-system-context.tsx` exposes it via React context.

### State Management

- **Chat state**: `src/lib/contexts/chat-context.tsx` wraps the Vercel AI SDK `useChat` hook
- **File system state**: `src/lib/contexts/file-system-context.tsx`
- **Anonymous users**: Work is serialized to `localStorage` via `src/lib/anon-work-tracker.ts` so it survives page reloads and can be saved on sign-up

### Authentication

JWT tokens (via `jose`) stored in httpOnly cookies. `src/middleware.ts` protects `/[projectId]` routes. Server actions in `src/actions/index.ts` handle sign-up/sign-in/sign-out using `bcrypt` for password hashing. Session utilities are in `src/lib/auth.ts`.

### Database

Prisma + SQLite (`prisma/dev.db`). Two models:
- `User` — email + hashed password
- `Project` — `data` (virtual file system) and `messages` stored as JSON strings, belongs to `User` (optional, nullable for anonymous projects)

The full schema is the source of truth: `prisma/schema.prisma`.

### UI Layout

Split-pane layout (`react-resizable-panels`):
- **Left**: Chat interface (`src/components/chat/`)
- **Right**: Tabbed panel — Preview (iframe sandbox, Babel in-browser transpilation) or Code view (Monaco Editor)

Components are organized by feature: `src/components/auth/`, `src/components/chat/`, `src/components/editor/`, `src/components/preview/`. Shared UI primitives from shadcn/ui live in `src/components/ui/`.

### Path Aliases

`@/*` maps to `src/*`.
