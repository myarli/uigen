# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm run setup       # Install deps + generate Prisma client + run migrations (first-time setup)
npm run dev         # Start dev server (Turbopack) at http://localhost:3000
npm run build       # Production build
npm run lint        # ESLint
npm test            # Run all Vitest tests
npm run db:reset    # Reset SQLite database (destructive)
```

**Run a single test file:**
```bash
npx vitest src/lib/__tests__/file-system.test.ts
```

**Note on `npm run dev` on Windows:** The `package.json` script uses Unix env var syntax (`NODE_OPTIONS='...'`) which fails in CMD/PowerShell. Run it via Git Bash — the script already injects `--require ./node-compat.cjs`, which removes `globalThis.localStorage`/`sessionStorage` that Node 25 exposes by default and breaks Next.js SSR guards.

## Environment

- `ANTHROPIC_API_KEY` in `.env` — optional. Without it, a `MockLanguageModel` returns static component stubs (Counter/Form/Card). With it, uses `claude-haiku-4-5`.

## Architecture

UIGen is a full-stack Next.js 15 (App Router) app where users chat with Claude to generate React components that render in a live preview iframe.

### Core Data Flow

```
User prompt → ChatInterface
  → useChat (Vercel AI SDK) → POST /api/chat/route.ts
    → streamText() with Claude + 2 tools (str_replace_editor, file_manager)
    → Tool calls modify an in-memory VirtualFileSystem
    → Streamed back to client
  → ChatProvider (chat-context.tsx) processes tool calls
  → FileSystemContext (file-system-context.tsx) updates VirtualFileSystem state
  → PreviewFrame re-renders: Babel transforms JSX → JS, iframe loads via esm.sh CDN
  → onFinish: project saved to SQLite (authenticated users only)
```

### Virtual File System

`src/lib/file-system.ts` — an in-memory Map-based tree. No files are ever written to disk. The VFS is serialized to JSON and stored in the `Project.data` column in SQLite. The AI always creates `/App.jsx` as the root entry point and uses `@/` import aliases for cross-file imports.

### AI Tools (defined in `/api/chat/route.ts`)

- `str_replace_editor` — create, view, str_replace, or insert into files in the VFS
- `file_manager` — rename or delete files in the VFS

### Authentication

JWT-based (`jose` library), stored as an HTTP-only `auth-token` cookie. Session expires in 7 days. Middleware (`src/middleware.ts`) protects `/api/projects/*` and `/api/filesystem/*`. Anonymous users can use the app; their work is tracked in localStorage and migrated to a new project on sign-up (via `useAuth` hook + `anon-work-tracker.ts`).

### Database

SQLite via Prisma. Schema has two models:
- `User` — email + bcrypt password
- `Project` — belongs to User; `messages` (JSON string of chat history) and `data` (JSON string of serialized VFS)

Prisma client is generated to `src/generated/prisma` (not the default location).

### Preview Rendering

`PreviewFrame` (`src/components/preview/PreviewFrame.tsx`) renders an iframe. The JSX transformer (`src/lib/transform/jsx-transformer.ts`) uses Babel standalone to compile JSX → JS in-browser. External imports (React, Tailwind, etc.) are resolved via `esm.sh` CDN using an import map injected into the iframe.

### Path Alias

`@/*` maps to `src/*` (configured in `tsconfig.json` and `vitest.config.mts`).
