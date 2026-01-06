# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

UIGen is an AI-powered React component generator with live preview. It uses Claude AI to generate React components dynamically, displays them in a sandboxed iframe, and operates entirely in a virtual file system (no files written to disk during development).

## Development Commands

### Essential Commands
- `npm run dev` - Start Next.js dev server with Turbopack
- `npm run build` - Build production bundle
- `npm run test` - Run tests with Vitest
- `npm run setup` - Complete first-time setup (install dependencies, generate Prisma client, run migrations)

### Database Commands
- `npx prisma generate` - Generate Prisma client to `src/generated/prisma`
- `npx prisma migrate dev` - Create and apply database migrations
- `npm run db:reset` - Reset database (destructive)

### Testing
- `npm test` - Run all tests
- Tests use Vitest with jsdom environment
- Test files located in `__tests__` directories adjacent to source files

## Architecture

### Virtual File System (VFS)

The core of the application is a client-side virtual file system that stores generated React components entirely in memory:

- **VirtualFileSystem class** (`src/lib/file-system.ts`): Implements file/directory operations (create, read, update, delete, rename) with path normalization
- **FileSystemContext** (`src/lib/contexts/file-system-context.tsx`): React context wrapping VFS with UI state management
- **Serialization**: VFS serializes to JSON for database storage and deserializes on load

### AI Integration Architecture

1. **Chat Flow**: User messages → Next.js API route → Claude AI via Vercel AI SDK
2. **Tool Execution**: Claude uses two tools to manipulate the VFS:
   - `str_replace_editor` (`src/lib/tools/str-replace.ts`): Create, view, edit files via string replacement
   - `file_manager` (`src/lib/tools/file-manager.ts`): Rename and delete files/directories
3. **Tool Handling**: Tools execute in API route, client receives tool calls and mirrors changes locally via `handleToolCall` in FileSystemContext
4. **System Prompt**: Generation prompt in `src/lib/prompts/generation.tsx` instructs AI to create React components with specific conventions (App.jsx entrypoint, @/ import alias, Tailwind styling)

### Live Preview System

1. **JSX Transformation** (`src/lib/transform/jsx-transformer.ts`):
   - Uses `@babel/standalone` to transpile JSX/TSX to vanilla JS in the browser
   - Creates ES Module import map with blob URLs for each transformed file
   - Resolves `@/` import alias to root directory
   - Handles third-party imports via esm.sh CDN
   - Collects CSS imports and injects as inline styles
   - Tracks syntax errors per file

2. **Preview Rendering** (`src/components/preview/PreviewFrame.tsx`):
   - Renders sandboxed iframe with generated HTML
   - Detects entry point (prefers /App.jsx, falls back to other patterns)
   - Shows syntax errors in formatted UI before attempting render
   - Uses import maps (native browser feature) to resolve module imports
   - Error boundary catches runtime errors

3. **Entry Point Convention**: Every project must have `/App.jsx` (or similar) as root component with default export

### Data Persistence

- **Database**: SQLite with Prisma ORM
- **Schema** (`prisma/schema.prisma`):
  - `User`: Authentication (email, bcrypt password)
  - `Project`: Belongs to user (optional), stores serialized messages and VFS data as JSON strings
- **Prisma Client**: Generated to `src/generated/prisma` (non-standard location)
- **Session Management**: JWT-based auth using `jose` library (`src/lib/auth.ts`)
- **Anonymous Mode**: Users can work without account; unsaved work tracked in localStorage

### Application Structure

- **Next.js 15** with App Router
- **Route Structure**:
  - `/` - Landing page with anonymous project creation
  - `/[projectId]` - Load existing project with VFS and messages restored
  - `/api/chat/route.ts` - Streaming AI chat endpoint with tool execution
- **Main UI** (`src/app/main-content.tsx`): Split view with chat interface, code editor (Monaco), and live preview
- **Context Providers**: FileSystemProvider wraps ChatProvider; both required for app functionality

## Key Patterns and Conventions

### Import Aliases
- All non-library imports use `@/` alias (e.g., `import Component from '@/components/Component'`)
- Alias maps to `src/` directory via TypeScript paths config
- VFS transformer resolves `@/` to root `/` in virtual filesystem

### File Path Normalization
- VFS always uses absolute paths starting with `/`
- Path normalization handles multiple slashes, trailing slashes, ensures consistency
- When AI creates files, they're always at virtual root level (e.g., `/App.jsx`, `/components/Button.jsx`)

### AI Tool Execution Pattern
1. AI calls tool in API route (server-side)
2. Tool executes on server-side VFS instance
3. Tool call returned to client via streaming
4. Client mirrors operation on client-side VFS via `handleToolCall`
5. UI refreshes via `refreshTrigger` state increment

### React Component Requirements
- Must export default component
- No HTML files (App.jsx is entrypoint)
- Use Tailwind CSS for styling (loaded via CDN in preview)
- Can import from esm.sh for third-party packages

## Testing Notes

- Test files use `__tests__` directories
- Vitest configured with React plugin and jsdom
- VFS has comprehensive unit tests (`src/lib/__tests__/file-system.test.ts`)
- Use `@testing-library/react` for component tests

## Environment Variables

- `ANTHROPIC_API_KEY` (optional): If not set, uses mock provider with static responses instead of real AI
- Database URL hardcoded to `file:./dev.db` in schema

## Important Implementation Details

1. **VFS Serialization**: Uses custom serialize/deserialize to handle Map data structure and parent-child relationships
2. **Monaco Editor**: Configured to work with VFS via custom file system provider
3. **Blob URLs**: All transformed modules converted to blob URLs for browser import map
4. **Prompt Caching**: System prompt uses Anthropic prompt caching with `cacheControl: { type: "ephemeral" }`
5. **Max Steps**: API route allows up to 40 tool execution steps (4 for mock provider)
6. **No TypeScript at Runtime**: All TS/TSX transpiled to JS via Babel before preview execution
