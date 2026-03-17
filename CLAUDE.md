# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

UIGen is an AI-powered React component generator with live preview. Users describe components in natural language, and the AI generates React code with Tailwind CSS styling that renders in real-time.

## Commands

```bash
# Setup (install dependencies & initialize database)
npm run setup

# Development server (uses Turbopack)
npm run dev

# Production build
npm run build

# Linting
npm run lint

# Run tests
npm run test

# Run single test file
npx vitest run path/to/test.test.ts

# Reset database
npm run db:reset
```

## Architecture

### Core Concept: Virtual File System

This app uses an in-memory Virtual File System (`src/lib/file-system.ts`) - files are never written to disk. The `VirtualFileSystem` class manages a tree structure of `FileNode` objects with path-based operations (create, read, update, delete, rename).

### AI Code Generation Flow

1. User sends a message via the chat interface
2. `ChatContext` (`src/lib/contexts/chat-context.tsx`) sends the message + current file state to `/api/chat`
3. The API route (`src/app/api/chat/route.ts`) uses the Vercel AI SDK with Claude (via `src/lib/provider.ts`)
4. The AI has access to two tools:
   - `str_replace_editor` (`src/lib/tools/str-replace.ts`) - Create files, replace/edit code
   - `file_manager` (`src/lib/tools/file-manager.ts`) - Rename/delete files
5. Tool calls update the VirtualFileSystem
6. Changes trigger preview refresh

### Preview System

The `PreviewFrame` component (`src/components/preview/PreviewFrame.tsx`) renders generated code in a sandboxed iframe:
- Uses Babel (`@babel/standalone`) to transpile JSX/TSX
- Creates import maps with esm.sh CDN for dependencies
- Supports `@/` path aliases for local imports
- Provides React 19 and Tailwind CSS via CDN

### Key Contexts

- `FileSystemContext` - Manages virtual files, handles AI tool calls to update files
- `ChatContext` - Manages chat messages, integrates with Vercel AI SDK's `useChat` hook

### Authentication

- JWT-based sessions stored in HTTP-only cookies (`src/lib/auth.ts`)
- Prisma with SQLite for persistence
- Anonymous users can use the app; registered users can save projects

### Database Schema

```prisma
model User {
  id        String
  email     String   @unique
  password  String
  projects  Project[]
}

model Project {
  id        String
  name      String
  userId    String?
  messages  String   @default("[]")  // JSON array of chat messages
  data      String   @default("{}")  // JSON serialized VirtualFileSystem
  user      User?    @relation(...)
}
```

### Model Configuration

The app uses Claude Haiku 4.5 by default (configured in `src/lib/provider.ts`). When no `ANTHROPIC_API_KEY` environment variable is set, it falls back to a mock provider that generates static example components.

## Testing

Tests use Vitest with jsdom environment. Test files are co-located with source files in `__tests__` directories. Run specific tests with:
```bash
npx vitest run src/lib/__tests__/file-system.test.ts
```

## Important Patterns

- Use `@/` import alias for local files (e.g., `import { foo } from '@/lib/foo'`)
- The AI prompt (`src/lib/prompts/generation.tsx`) instructs the AI to always start with `/App.jsx` as the entry point
- Files are structured with `/App.jsx` as root, and `/components/` for additional files
- All AI tool responses are handled client-side in `FileSystemContext.handleToolCall`