# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# Initial setup (install deps, generate Prisma client, run migrations)
npm run setup

# Development server (Turbopack)
npm run dev

# Run all tests
npm test

# Run a single test file
npx vitest run src/lib/__tests__/file-system.test.ts

# Lint
npm run lint

# Reset database (destructive)
npm run db:reset
```

All dev/build scripts require the Node compatibility shim: `NODE_OPTIONS='--require ./node-compat.cjs'` (already embedded in npm scripts — don't strip it).

## Architecture

### Request / Data Flow

```
User types prompt
  → ChatContext (useAIChat from ai-sdk/react)
  → POST /api/chat  { messages, files: VirtualFileSystem.serialize(), projectId }
  → Claude streams text + tool calls (str_replace_editor / file_manager)
  → FileSystemContext.handleToolCall() applies each tool call to in-memory VirtualFileSystem
  → PreviewFrame re-renders (iframe with Babel-transformed JSX + esm.sh import maps)
  → onFinish: saves messages + serialized VFS to Prisma Project (JSON columns)
```

Anonymous users: state persisted to `sessionStorage` via `anon-work-tracker.ts`. On sign-in, `useAuth.handlePostSignIn()` migrates anonymous work into a new Prisma Project.

### Key Directories

| Path | What lives here |
|------|----------------|
| `src/app/api/chat/route.ts` | AI streaming endpoint — the core of code generation |
| `src/lib/file-system.ts` | `VirtualFileSystem` class (in-memory Map tree, serialize/deserialize) |
| `src/lib/contexts/` | `FileSystemContext` and `ChatContext` — shared state providers |
| `src/lib/tools/` | Zod schemas + executors for `str_replace_editor` and `file_manager` tools |
| `src/lib/prompts/` | System prompt sent to Claude for generation |
| `src/lib/transform/` | Babel standalone JSX transformer + esm.sh import-map builder |
| `src/lib/provider.ts` | Selects real Anthropic model vs. `MockLanguageModel` (no API key) |
| `src/lib/auth.ts` | JWT session management (jose, HS256, `auth-token` cookie, 7-day TTL) |
| `src/actions/index.ts` | Server actions: signIn, signUp, signOut, getUser, project CRUD |
| `src/components/preview/PreviewFrame.tsx` | Iframe-based live preview (generates HTML with import maps) |
| `src/components/editor/` | Monaco code editor + file tree |

### AI Tool Calling

Claude is given two tools that operate on the `VirtualFileSystem`:

- **`str_replace_editor`** — `create`, `str_replace`, `insert`, `view` file operations
- **`file_manager`** — `rename`, `delete` operations

Max steps: 40 (real API) / 4 (mock). The system prompt requires `/App.jsx` as the root entry point and `@/` as the import alias for the virtual FS.

### Authentication

JWT stored in `auth-token` cookie. `middleware.ts` guards `/api/projects/*` and `/api/filesystem*`. Page-level ownership check is done via server action in `[projectId]/page.tsx`. JWT secret defaults to `'development-secret-key'` if `JWT_SECRET` env var is not set.

### Database

SQLite via Prisma. Two models: `User` (email + bcrypt password) and `Project` (userId, name, `messages` JSON string, `data` JSON string for serialized VFS). Prisma client is a singleton in `src/lib/prisma.ts`, generated to `src/generated/prisma`.

### Preview Pipeline

`PreviewFrame` generates a full HTML document containing:
1. An esm.sh import map (React, ReactDOM, Tailwind CDN, any `@/` aliased files)
2. The Babel-standalone script to transform JSX in the browser
3. An inline `<script type="text/babel">` that imports and renders `/App.jsx`

No files are ever written to disk — everything is virtual.

### Environment Variables

| Variable | Required | Notes |
|----------|----------|-------|
| `ANTHROPIC_API_KEY` | No | Without it, `MockLanguageModel` returns static demo responses |
| `JWT_SECRET` | No | Defaults to `'development-secret-key'` |
