# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# Initial setup (install deps + generate Prisma client + run migrations)
npm run setup

# Development server (Turbopack)
npm run dev

# Build for production
npm run build

# Run tests (watch mode)
npm run test

# Run a single test file
npx vitest run src/lib/__tests__/file-system.test.ts

# Lint
npm run lint

# Reset database
npm run db:reset
```

Set `ANTHROPIC_API_KEY` in `.env` to enable AI generation; the app falls back to a mock provider without it.

## Architecture

UIGen is a Next.js 15 (App Router) AI-powered React component generator. Users describe components in natural language; Claude generates and edits files via tool calls that update an in-memory virtual file system, which is then compiled and rendered in a sandboxed iframe.

### Core Data Flow

1. User sends a message → **ChatContext** (`src/lib/contexts/chat-context.tsx`) streams it to `/api/chat`
2. `/api/chat` (`src/app/api/chat/route.ts`) calls Claude with the current file system serialized as context and two tools: `str_replace_editor` and `file_manager`
3. Tool calls update the **virtual file system** (`src/lib/file-system.ts`) — an in-memory tree with no disk I/O
4. **FileSystemContext** (`src/lib/contexts/file-system-context.tsx`) propagates changes to the editor and preview
5. **JSX Transformer** (`src/lib/transform/jsx-transformer.ts`) uses Babel to compile JSX/TypeScript in the browser
6. **PreviewFrame** (`src/components/preview/PreviewFrame.tsx`) renders the compiled output in a sandboxed iframe

### Key Directories

- `src/app/` — Next.js App Router pages and the `/api/chat` route
- `src/components/` — UI split into `chat/`, `editor/`, `preview/`, `auth/`, and `ui/` (shadcn)
- `src/lib/` — Core logic: virtual FS, AI provider, auth, contexts, tools, prompts, transformer
- `src/actions/` — Next.js server actions for auth and project CRUD (Prisma → SQLite)
- `prisma/` — Schema with `User` and `Project` models; project state (messages + FS) is stored as JSON strings

### AI Integration

- Provider: `src/lib/provider.ts` — returns a real Anthropic client or `MockLanguageModel` based on env
- System prompt: `src/lib/prompts/generation.tsx`
- Tools: `src/lib/tools/str-replace.ts` (create/view/replace/insert files) and `src/lib/tools/file-manager.ts` (list/delete)
- Model: `claude-haiku-4-5` with prompt caching (`ephemeral` cache control)

### Authentication

Custom JWT sessions via `jose` + `bcrypt`. `src/lib/auth.ts` manages tokens; `src/actions/index.ts` exposes server actions (`signup`, `signin`, `signout`, `getUser`). Middleware at `src/middleware.ts` handles route protection.

### Testing

Vitest + jsdom + React Testing Library. Tests live in `__tests__` directories alongside source files.
