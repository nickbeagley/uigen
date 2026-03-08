# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm run setup        # First-time setup: install deps, generate Prisma client, run migrations
npm run dev          # Start dev server (http://localhost:3000)
npm run build        # Production build
npm run lint         # ESLint
npm run test         # Run all tests with Vitest
npm run db:reset     # Reset SQLite database
```

Run a single test file:
```bash
npx vitest src/lib/transform/__tests__/jsx-transformer.test.ts
```

## Architecture

UIGen is a Next.js 15 app where users describe React components in natural language and Claude generates them with live preview.

### Request Flow

1. User sends a message in `ChatInterface` → `chat-context.tsx` calls `/api/chat`
2. `/api/chat/route.ts` streams responses from Claude with two tools:
   - `str_replace_editor` — view/create/edit files in the virtual file system
   - `file_manager` — rename/delete files
3. Tool calls update `FileSystemContext` (in-memory `VirtualFileSystem`)
4. `PreviewFrame` watches the file system, runs `createPreviewHTML()` → renders in a sandboxed iframe

### Virtual File System

`src/lib/file-system.ts` — pure in-memory implementation. No files are written to disk. Serializes to JSON for persistence in the DB (`Project.data` column).

### JSX Transform & Preview

`src/lib/transform/jsx-transformer.ts` handles the browser-side build pipeline:
- Babel standalone transforms JSX/TSX to plain JS
- `createImportMap()` maps bare imports: local files → Blob URLs, npm packages → `esm.sh` CDN
- `createPreviewHTML()` produces a full HTML document with Tailwind CDN and an error boundary

### AI Provider

`src/lib/provider.ts` — uses `claude-haiku-4-5` by default. Falls back to `MockLanguageModel` (returns static demo components) when `ANTHROPIC_API_KEY` is absent.

### Persistence

Prisma + SQLite (`prisma/dev.db`). Authenticated users get projects saved; `Project.messages` stores chat history and `Project.data` stores the serialized virtual file system. Anonymous work is tracked via `anon-work-tracker.ts` and can be claimed on sign-in.

### Auth

JWT sessions via JOSE + cookies. Password hashing with bcrypt. Middleware in `src/middleware.ts` protects routes.

## Key Conventions

- Path alias `@/*` → `src/*`
- UI components from shadcn/ui live in `src/components/ui/`; add new ones with `npx shadcn@latest add <component>`
- Tailwind v4 (no `tailwind.config.js` — configured via CSS)
- Tests use Vitest + React Testing Library with JSDOM environment
