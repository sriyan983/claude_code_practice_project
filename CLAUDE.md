# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

UIGen is an AI-powered React component generator with live preview capabilities. It's a Next.js 15 application that allows users to describe React components through chat and see them generated and rendered in real-time using a virtual file system.

## Development Commands

### Setup and Installation
```bash
npm run setup  # Installs dependencies, generates Prisma client, runs migrations
```

### Development
```bash
npm run dev             # Start development server with Turbopack
npm run dev:daemon      # Start dev server in background, logs to logs.txt
```

### Testing and Quality
```bash
npm test                        # Run all Vitest tests
npm test -- src/lib/__tests__/file-system.test.ts  # Run a single test file
npm run lint                    # ESLint checking
npm run build                   # Production build
```

### Database Management
```bash
npx prisma generate    # Generate Prisma client
npx prisma migrate dev # Run database migrations
npm run db:reset       # Reset database (force)
```

## Architecture Overview

### Core Data Flow
1. User sends a message in `ChatInterface` → `ChatContext` calls `/api/chat` with the serialized file system and messages
2. `POST /api/chat` reconstructs a `VirtualFileSystem` from the serialized nodes, calls `streamText` with two tools, and streams back the response
3. Tool calls arrive at the client via `onToolCall` in `useAIChat`, which dispatches to `handleToolCall` in `FileSystemContext`
4. `FileSystemContext` applies mutations to the in-memory `VirtualFileSystem` and increments `refreshTrigger`
5. `PreviewFrame` watches `refreshTrigger`, transforms all files via Babel + blob URLs + an import map, then sets `iframe.srcdoc`

### Virtual File System (`src/lib/file-system.ts`)
`VirtualFileSystem` is an in-memory tree of `FileNode` objects (files and directories). Key methods:
- `serialize()` → `Record<string, FileNode>` (strips `Map` children for JSON)
- `deserializeFromNodes(data)` → reconstructs from serialized form
- `createFileWithParents`, `replaceInFile`, `insertInFile` — used by AI tool handlers
- The singleton `export const fileSystem` at the bottom is **not** used at runtime; each request and context creates its own instance.

### AI Tool System
Two tools are registered per request in `src/app/api/chat/route.ts`:
- **`str_replace_editor`** (`src/lib/tools/str-replace.ts`): `view`, `create`, `str_replace`, `insert` commands. `undo_edit` returns an error.
- **`file_manager`** (`src/lib/tools/file-manager.ts`): `rename`, `delete` commands.

Tools execute server-side (mutating the request-scoped `VirtualFileSystem`) to return results to the model, and also execute client-side via `onToolCall` to keep the browser's file system in sync. Both sides must stay consistent.

### Preview Pipeline (`src/lib/transform/jsx-transformer.ts`)
`createImportMap(files)` does all the heavy lifting:
1. Transforms every `.js/.jsx/.ts/.tsx` file with Babel Standalone (JSX + optional TypeScript)
2. Creates blob URLs for each transformed file
3. Builds an ES import map: React/ReactDOM point to `esm.sh`, third-party packages resolve to `https://esm.sh/<package>`, local files resolve to their blob URLs with multiple alias forms (`@/`, without extension, etc.)
4. Missing local imports get placeholder modules instead of failing
5. CSS files are inlined as `<style>` blocks

`createPreviewHTML` wraps everything in a sandboxed iframe document with Tailwind CDN, the import map, an `ErrorBoundary`, and a `loadApp()` call.

### Context Architecture
Both contexts are provided in `src/lib/provider.ts`:
- **`FileSystemProvider`**: Holds the single `VirtualFileSystem` instance for the session. Exposes CRUD operations and `handleToolCall`. Uses `refreshTrigger` (an incrementing integer) as the signal to re-render.
- **`ChatProvider`**: Wraps `useAIChat` from `@ai-sdk/react`. Passes the serialized file system and `projectId` in every request body. Routes tool calls to `FileSystemContext.handleToolCall`.

### Authentication & Sessions
- JWT in an `httpOnly` cookie (`auth-token`), signed with `JWT_SECRET`, valid 7 days (`src/lib/auth.ts`)
- `getSession()` is server-only (imports `server-only`); `verifySession(request)` is used in middleware
- Middleware only protects `/api/projects` and `/api/filesystem`; the chat route `/api/chat` is intentionally unprotected (project saving inside `onFinish` checks auth itself)

### Anonymous Work Tracking (`src/lib/anon-work-tracker.ts`)
When an unauthenticated user has chat messages or files, the state is persisted in `sessionStorage` under `uigen_has_anon_work` and `uigen_anon_data`. This allows prompting users to sign up to save their work. Cleared on explicit logout or after saving to a project.

### Project Routing
- `/` — if authenticated, redirects to the most recent project or creates a new one; otherwise renders `MainContent` for anonymous use
- `/[projectId]` — loads a specific project's messages and file system from the database, passes them as `initialMessages` and `initialData` to the providers

### Mock Provider (`src/lib/provider.ts`)
When `ANTHROPIC_API_KEY` is absent, `getLanguageModel()` returns `MockLanguageModel`, which generates deterministic component files (Counter, ContactForm, Card) based on keywords in the user's prompt. The mock uses `toolMessageCount` to sequence steps across the multi-step loop. The real model uses `claude-sonnet-4-0` with `maxSteps: 40`; the mock uses `maxSteps: 4`.

## Instructions for Claude
- Do NOT save project-related instructions or context to the auto-memory system (`~/.claude/projects/.../memory/`). Always update this CLAUDE.md file directly instead.
- Database schema is defined in `prisma/schema.prisma` — reference it for all database structure questions.
- Database migrations are in `prisma/migrations/` — contains the initial table definitions and subsequent schema alterations in chronological order.

## Development Notes
- Vitest config is in `vitest.config.mts`
- Uses Tailwind CSS v4 with custom configuration
- Monaco Editor for code editing with syntax highlighting
- React 19 with concurrent features
- TypeScript throughout with strict configuration
- Add comments sparingly — only for complex logic; every comment must end with `- by Narayan`
