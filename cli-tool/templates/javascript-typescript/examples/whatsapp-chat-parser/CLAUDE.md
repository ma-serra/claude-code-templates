# CLAUDE.md - WhatsApp Chat Parser Website

This file provides guidance to Claude Code when working with the whatsapp-chat-parser-website project — a privacy-first React/TypeScript web app that parses and visualizes exported WhatsApp chat logs entirely in the browser.

## Project Overview

This is a **client-side only** React application. No data is ever sent to a server. All file parsing and rendering happens locally in the user's browser. Preserving this privacy-first architecture is a hard requirement.

Key facts:
- Accepts `.txt` files (plain WhatsApp chat exports) or `.zip` archives (with media attachments)
- Displays images, videos, and audio inline when present in ZIP exports
- Built on the `whatsapp-chat-parser` npm package for the actual parsing logic
- Deployed on Netlify

## Development Commands

### Setup
```bash
npm install        # Install dependencies
```

### Development
```bash
npm start          # Start Vite dev server (http://localhost:5173)
```

### Production
```bash
npm run build      # Type-check + bundle for production
npm run preview    # Preview the production build locally
```

### Code Quality
```bash
npm run lint       # ESLint (Airbnb config)
npm run format     # Prettier formatting
```

### Docker
```bash
docker build -t whatsapp-chat-parser-website .
docker run -p 8080:80 whatsapp-chat-parser-website
```

## Technology Stack

| Layer | Technology |
|-------|-----------|
| Language | TypeScript (strict mode) |
| UI Framework | React (functional components + hooks) |
| State Management | Jotai (atomic state) |
| Styling | Styled Components |
| File Parsing | `whatsapp-chat-parser` npm package |
| ZIP Handling | JSZip |
| Build Tool | Vite |
| Linting | ESLint (Airbnb ruleset) |
| Formatting | Prettier |
| Deployment | Netlify |

## Architecture & Key Patterns

### State Management with Jotai
The app uses Jotai atoms for reactive state. Prefer atomic state over React context for shared state:

```typescript
import { atom, useAtom } from 'jotai';

// Define atoms at module level
export const messagesAtom = atom<Message[]>([]);
export const fileNameAtom = atom<string>('');
```

### File Processing Pipeline
All file handling happens client-side. The core flow is:

1. **User uploads file** → `<input type="file">` or drag-and-drop
2. **Detect file type** → `.txt` or `.zip`
3. **For ZIP files** → Use JSZip to extract contents; find the `.txt` chat log + media files
4. **Parse chat log** → Pass `.txt` content to `whatsapp-chat-parser`
5. **Map media** → Link parsed messages to their media files by filename
6. **Render** → Display messages with inline media

```typescript
import parseChat from 'whatsapp-chat-parser';
import JSZip from 'jszip';

// Parse a plain text chat
const messages = await parseChat(txtContent);

// Extract from ZIP
const zip = await JSZip.loadAsync(file);
const chatFile = zip.file(/.*\.txt$/)[0];
if (!chatFile) {
  throw new Error('No .txt chat file found in the ZIP archive.');
}
const chatContent = await chatFile.async('string');
```

### Privacy Invariant
Never add network requests for user data. The following must remain true:
- No analytics calls with message content
- No file uploads to any server
- No third-party SDKs that receive user data
- `Content-Security-Policy` must not allow external data exfiltration

### Styled Components Patterns
Use template literals with the `styled` API. Keep styles co-located with components:

```typescript
import styled from 'styled-components';

const MessageBubble = styled.div<{ $isOwn: boolean }>`
  background: ${({ $isOwn }) => $isOwn ? '#dcf8c6' : '#fff'};
  border-radius: 8px;
  padding: 8px 12px;
`;
```

### TypeScript Configuration
The project uses strict TypeScript. Always:
- Provide explicit return types on functions
- Use the types exported by `whatsapp-chat-parser` (`Message`, `Author`, etc.)
- Avoid `any` — use `unknown` and narrow with type guards
- Use `as const` for literal arrays/objects that shouldn't widen

## Project Structure

```
src/
├── atoms/          # Jotai atoms (global state)
├── components/     # React components
├── hooks/          # Custom hooks
├── types/          # TypeScript type definitions
├── utils/          # Pure utility functions (parsing helpers, formatters)
└── App.tsx         # Root component

public/             # Static assets
index.html          # Entry HTML (Vite)
vite.config.ts      # Build configuration
tsconfig.json       # TypeScript config (strict mode)
.eslintrc           # ESLint (Airbnb)
.prettierrc         # Prettier
dockerfile          # Docker build
```

## Media Handling

WhatsApp exports media with predictable filenames. The pattern for matching media to messages:
- ZIP archive contains the `.txt` chat log and media files in the same directory
- Message text references attachments as `<Media omitted>` or `filename.jpg (file attached)`
- Match by extracting the filename from the message text and looking it up in the ZIP

Supported media types to handle:
- Images: `.jpg`, `.jpeg`, `.png`, `.gif`, `.webp`
- Videos: `.mp4`, `.mov`, `.avi`
- Audio: `.opus`, `.ogg`, `.mp3`, `.m4a`
- Documents: PDF and others (show download link, not inline preview)

## Internationalization Considerations

WhatsApp chat formats differ by locale (date/time format, system message wording). The `whatsapp-chat-parser` package handles format detection. When adding UI text:
- Keep all user-facing strings accessible for future i18n
- Avoid hardcoding date/time formats — use the parsed `Date` objects
- RTL languages (Arabic, Hebrew) require CSS `direction: rtl` on message containers

## Performance Guidelines

- The parsed message list can be very large (thousands of messages). Use **virtualization** (`react-window` or `react-virtual`) for the message list.
- Media blobs should be created with `URL.createObjectURL()` and revoked when the component unmounts to avoid memory leaks.
- Large ZIP files: show a loading indicator and use async JSZip methods to avoid blocking the UI thread.

## Testing

The project does not currently have an automated test suite. When adding tests:
- Use **Vitest** (already aligned with Vite) for unit tests
- Use **React Testing Library** for component tests
- Mock file inputs with `File` and `Blob` constructors
- Test the parsing pipeline with fixture `.txt` files from the `whatsapp-chat-parser` package tests

## Common Tasks

### Adding a New Feature
1. Define any new state in `src/atoms/`
2. Create components in `src/components/` using Styled Components
3. Keep side effects (file I/O, media URL creation) in custom hooks in `src/hooks/`
4. Verify the privacy invariant is not violated
5. Run `npm run lint` and `npm run build` before committing

### Adding Support for a New Media Type
1. Add the file extension to the media type detection logic in the relevant utility
2. Render inline if the browser supports it natively; otherwise provide a download link
3. Ensure `URL.revokeObjectURL()` is called on cleanup

### Debugging Parsing Issues
The `whatsapp-chat-parser` package exports a `parseString` function that accepts raw chat text. To debug:
```typescript
import { parseString } from 'whatsapp-chat-parser';
const result = parseString(rawText);
console.log(result); // Array of Message objects
```

Different WhatsApp versions and locales produce different export formats. Check the parser's GitHub issues if a format isn't recognized.

## Deployment

The site deploys automatically to **Netlify** on merge to `main`. The build command is `npm run build` and the publish directory is `dist` (Vite default).

Environment variables: none required — the app has no backend and no API keys.

## Security & Privacy Checklist

Before merging any PR:
- [ ] No new network requests that transmit user-uploaded content
- [ ] No new third-party scripts that could exfiltrate data
- [ ] Media object URLs are revoked on component unmount
- [ ] No `dangerouslySetInnerHTML` with user content (XSS risk)
- [ ] Dependencies reviewed with `npm audit`
