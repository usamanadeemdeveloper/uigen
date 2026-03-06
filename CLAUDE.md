# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# First-time setup
npm run setup          # installs deps, generates Prisma client, runs migrations

# Development
npm run dev            # starts Next.js dev server with Turbopack at localhost:3000

# Build & lint
npm run build
npm run lint

# Tests
npm test               # runs all tests with Vitest
npm test -- path/to/test.ts   # run a single test file

# Database
npm run db:reset       # reset and re-run all migrations (destructive)
npx prisma migrate dev # run new migrations after schema changes
npx prisma generate    # regenerate Prisma client after schema changes
```

## Environment

Copy `.env` and set:
- `ANTHROPIC_API_KEY` — if absent, a `MockLanguageModel` is used instead (returns static components)
- `JWT_SECRET` — defaults to `"development-secret-key"` if not set

## Architecture

### High-level flow

1. User opens the app → anonymous or authenticated session
2. Chat messages are sent to `POST /api/chat` with the serialized virtual file system
3. The API uses Vercel AI SDK `streamText` with two tools: `str_replace_editor` and `file_manager`
4. The AI calls those tools to create/edit files in the virtual FS on the server
5. Tool calls stream back to the client, where `FileSystemContext.handleToolCall` applies them to the client-side `VirtualFileSystem`
6. `PreviewFrame` watches `refreshTrigger` and re-renders the iframe whenever files change
7. For authenticated users, the project (messages + file system snapshot) is persisted to SQLite via Prisma on each `onFinish`

### Virtual File System (`src/lib/file-system.ts`)

`VirtualFileSystem` is an in-memory tree of `FileNode` objects (files and directories). It lives on both the server (reconstructed per-request from the serialized `files` payload) and the client (held in `FileSystemProvider`). Key methods: `serialize()` / `deserializeFromNodes()` for round-tripping through JSON.

### Contexts

- `FileSystemProvider` (`src/lib/contexts/file-system-context.tsx`) — owns the client VFS instance, exposes CRUD helpers and `handleToolCall` which interprets `str_replace_editor` and `file_manager` tool call payloads
- `ChatProvider` (`src/lib/contexts/chat-context.tsx`) — wraps Vercel AI SDK `useChat`, sends `files` (serialized VFS) and optional `projectId` on every request, wires `onToolCall` → `handleToolCall`

### Preview (`src/components/preview/PreviewFrame.tsx`)

Renders an iframe whose `srcdoc` is rebuilt on every `refreshTrigger`. `jsx-transformer.ts` uses `@babel/standalone` to transpile JSX/TSX in-browser, builds an ES module import map pointing each file path to a blob URL, and resolves third-party packages through `esm.sh`. The generated app must have `/App.jsx` as its entry point (or `App.tsx`, `index.jsx`, etc.).

### AI tooling (`src/lib/tools/`)

- `str_replace_editor` — supports `view`, `create`, `str_replace`, and `insert` commands on the VFS
- `file_manager` — supports `rename` and `delete`

### AI provider (`src/lib/provider.ts`)

`getLanguageModel()` returns `anthropic("claude-haiku-4-5")` when `ANTHROPIC_API_KEY` is set, otherwise a `MockLanguageModel` that emits static tool calls to demonstrate the flow.

### Auth (`src/lib/auth.ts`)

JWT-based sessions stored in an `httpOnly` cookie (`auth-token`). Password hashing via `bcrypt`. All auth logic is `server-only`. Anonymous users can use the app; their work is tracked in `src/lib/anon-work-tracker.ts` so it can be migrated on sign-up.

### Data model (`prisma/schema.prisma`)

Two models: `User` (email + bcrypt password) and `Project` (name, optional userId, `messages` JSON string, `data` JSON string for the serialized VFS). Prisma client is generated to `src/generated/prisma`.

### AI prompt (`src/lib/prompts/generation.tsx`)

Instructs the model to:
- Always create `/App.jsx` as the entry point
- Style with Tailwind (no hardcoded styles)
- Use `@/` alias for all local file imports (e.g. `@/components/Button`)
- Never create HTML files
