# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

UIGen is an AI-powered React component generator with live preview. Users describe components in natural language, and the application generates React code in a virtual file system with real-time preview.

**Key Feature**: The application can run without an API key - it falls back to a mock provider that generates static component templates.

## Development Commands

### Setup
```bash
npm run setup
```
This command installs dependencies, generates Prisma client, and runs database migrations. Run this first.

### Development Server
```bash
npm run dev
```
Starts Next.js development server with Turbopack at http://localhost:3000

### Testing
```bash
npm test              # Run all tests in watch mode
npm test -- --run     # Run tests once (CI mode)
```
Tests use Vitest with jsdom environment. Test files are located in `__tests__` directories alongside source files.

### Build & Production
```bash
npm run build         # Build for production
npm start             # Start production server
npm run lint          # Run ESLint
```

### Database
```bash
npm run db:reset      # Reset database (destructive!)
npx prisma studio     # Open Prisma Studio GUI
npx prisma migrate dev --name <migration-name>  # Create new migration
```

## Architecture

### Virtual File System

The core of UIGen is the `VirtualFileSystem` class in `src/lib/file-system.ts`. This is NOT a real file system - it's an in-memory tree structure that stores generated React components.

**Key points**:
- All file paths use Unix-style absolute paths starting with `/`
- Files exist only in memory; nothing is written to disk
- The VFS serializes to/from JSON for database persistence
- Path alias `@/` maps to root directory (`/`)

**Operations**: The VFS supports standard file operations (create, read, update, delete, rename) plus specialized methods for AI tool integration (`viewFile`, `createFileWithParents`, `replaceInFile`, `insertInFile`).

### AI Component Generation Flow

1. **Chat API** (`src/app/api/chat/route.ts`): Receives user messages and current file system state
2. **System Prompt** (`src/lib/prompts/generation.tsx`): Instructs the AI to create React components with:
   - `/App.jsx` as the required entry point (default export)
   - `@/` import alias for local files
   - Tailwind CSS for styling (never inline styles)
   - No HTML files (React components only)
3. **AI Tools**: Two tools are provided to the AI model:
   - `str_replace_editor`: View, create, edit files (uses `src/lib/tools/str-replace.ts`)
   - `file_manager`: Rename/delete files and directories (uses `src/lib/tools/file-manager.ts`)
4. **VFS Updates**: AI tool calls modify the VirtualFileSystem instance
5. **Preview Transformation** (`src/lib/transform/jsx-transformer.ts`):
   - Babel transforms JSX/TSX to browser-compatible JavaScript
   - Creates import map with blob URLs for each module
   - Resolves `@/` aliases to absolute paths
   - Loads third-party packages from esm.sh
   - Collects CSS imports into a single style block
   - Displays syntax errors in preview if transformation fails
6. **Persistence**: For authenticated users, the VFS state and chat history are saved to the database on completion

### Mock Provider Fallback

When `ANTHROPIC_API_KEY` is not set, the app uses `MockLanguageModel` (in `src/lib/provider.ts`) which:
- Generates predefined component templates (counter, form, or card based on user prompt keywords)
- Simulates multi-step tool usage (create component → enhance → create App.jsx → finish)
- Limits to 4 steps to prevent repetition (vs 40 for real API)

### Authentication & Projects

- **Anonymous Mode**: Users can work without signing up; project data stored in browser localStorage via `src/lib/anon-work-tracker.ts`
- **Authenticated Mode**: JWT-based sessions (see `src/lib/auth.ts`) with 7-day expiration; projects persist to SQLite database
- **Database Schema** (`prisma/schema.prisma`):
  - Users have many Projects
  - Projects store serialized VFS (`data` field) and chat history (`messages` field) as JSON strings
  - Prisma client generated to `src/generated/prisma` (not the default location)

### Component Structure

- **Layout**: Three-panel interface with file tree (left), editor/preview tabs (center), and chat (right)
- **Contexts**:
  - `FileSystemContext`: Manages VFS state and operations
  - `ChatContext`: Manages conversation state and AI streaming
- **Key Components**:
  - `FileTree`: Displays VFS as a tree
  - `CodeEditor`: Monaco editor for viewing/editing files
  - `PreviewFrame`: iframe that renders transformed React components
  - `ChatInterface`: Streaming chat UI with markdown rendering

## Important Patterns

### File Imports
All local imports use the `@/` alias which maps to `src/`:
```tsx
import { VirtualFileSystem } from '@/lib/file-system';  // NOT '../lib/file-system'
```

### Path Handling in VFS
The VFS normalizes all paths to absolute Unix-style:
```typescript
// Always starts with /
"/App.jsx"
"/components/Button.jsx"

// Never relative
"./components/Button.jsx"  // ❌ Don't use this format in VFS
```

### Entry Point Requirement
Every project MUST have `/App.jsx` (or `/App.tsx`) as the root component with a default export. The preview system loads this as the entry point.

### Preview Transformation
When working with the preview system, be aware that:
- TypeScript/JSX is transpiled to plain JavaScript at runtime in the browser
- Third-party imports are resolved from esm.sh CDN
- CSS files are concatenated into a single style block
- Syntax errors prevent preview rendering and display formatted error messages

### Database Client Location
The Prisma client is generated to a non-standard location: `src/generated/prisma`. Always import it from `@/lib/prisma`, not `@prisma/client` directly.
