# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

UIGen is an AI-powered React component generator with live preview. Users describe components in natural language, Claude AI generates them via tool calls, and they render in a sandboxed iframe with live preview. Components live in a virtual file system (no disk writes).

## Commands

- `npm run setup` — Install deps, generate Prisma client, run migrations (first-time setup)
- `npm run dev` — Start dev server (Next.js + Turbopack on port 3000)
- `npm run build` — Production build
- `npm run lint` — ESLint
- `npm run test` — Run Vitest (all tests)
- `npx vitest run src/components/editor/__tests__/FileTree.test.tsx` — Run a single test file
- `npm run db:reset` — Reset SQLite database

## Architecture

### Stack
Next.js 15 (App Router) + React 19 + TypeScript + Tailwind CSS v4 + Prisma (SQLite) + Vercel AI SDK + Anthropic Claude Haiku 4.5

### Core Data Flow
1. User sends message via chat → `ChatContext` (wraps Vercel AI SDK's `useChat`)
2. `POST /api/chat/route.ts` streams response from Claude with tool calls
3. Tool calls (`str_replace_editor`, `file_manager`) mutate a server-side `VirtualFileSystem` instance
4. Client-side `FileSystemContext` replays the same tool calls to keep its `VirtualFileSystem` in sync
5. `PreviewFrame` renders files via Babel Standalone JSX transformation in a sandboxed iframe, using esm.sh CDN for npm imports

### Key Abstractions

**VirtualFileSystem** (`src/lib/file-system.ts`): In-memory Map-based file system. Supports create, read, update, delete, rename, str_replace, and insert operations. Serializes to/from JSON for database persistence. Used both server-side (in API route) and client-side (in context).

**Tool System** (`src/lib/tools/`): Two tools exposed to Claude:
- `str_replace_editor` — create, str_replace, insert, view file operations
- `file_manager` — rename, delete operations

**Contexts** (`src/lib/contexts/`):
- `FileSystemContext` — owns the client-side VirtualFileSystem, handles tool call replay, manages selected file state
- `ChatContext` — wraps `useChat` from Vercel AI SDK, serializes file system state into requests

**Provider** (`src/lib/provider.ts`): Returns Claude Haiku 4.5 if `ANTHROPIC_API_KEY` is set, otherwise returns `MockLanguageModel` that generates static demo components (counter, form, card).

### UI Layout
Three-panel resizable layout (`src/app/main-content.tsx`):
- Left panel: Chat interface
- Right panel (tabbed): Preview iframe | Code editor (FileTree + Monaco)

### Auth
JWT tokens (jose) in HTTP-only cookies with 7-day expiry. Server actions in `src/actions/index.ts` handle signUp/signIn/signOut. Anonymous users can generate components but cannot persist projects.

### Database
SQLite via Prisma. Two models: `User` and `Project`. Project stores `messages` (JSON array) and `data` (serialized VirtualFileSystem) as text fields. Schema at `prisma/schema.prisma`, generated client output to `src/generated/prisma/`.

### Preview Rendering
`PreviewFrame` (`src/components/preview/PreviewFrame.tsx`) builds an import map from file contents, transforms JSX with Babel Standalone, resolves bare imports to esm.sh CDN URLs, and renders in an iframe. Entry point detection looks for `/App.jsx`, `/App.tsx`, `/index.jsx`, etc.

## Conventions

- Import alias: `@/` maps to `./src/`
- UI components use shadcn/ui (new-york style) in `src/components/ui/`
- Tests live in `__tests__/` directories alongside components, using Vitest + Testing Library + jsdom
- All `node-compat.cjs` polyfills are required via `NODE_OPTIONS` in npm scripts
- API route max duration: 120 seconds; max AI steps: 40 (real) or 4 (mock)
- Use comments sparingly — only comment complex code
- Always reference `prisma/schema.prisma` when you need to understand database structure
