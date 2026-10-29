# AGENTS.md - openclaw-webchat

## Project Overview
This is a simple Node.js WebChat server for OpenClaw. It serves a frontend and relays WebSocket messages between a browser and an OpenClaw Gateway plugin. The original UI is in Chinese.

**Key files:**
- `server/server.js` — main Node.js server (Express + WebSocket).
- `server/public/` — all frontend assets (HTML, CSS, JS). This is where the UI and Chinese text live.
- `plugin/` — OpenClaw channel plugin (not our focus).

## Goals for this project
1.  **Localize UI to Russian**: Replace all hardcoded Chinese strings in `server/public/` with Russian text. Keep the logic intact.
2.  **Add Markdown rendering**: The chat interface should render assistant responses as formatted Markdown (bold, lists, code blocks, etc.) instead of plain text.

## Commands
- Install dependencies: `cd server && npm install`
- Run server (development): `cd server && node server.js`
- The server runs on `http://localhost:3100` by default.

## Conventions & Constraints
- Do not modify the WebSocket protocol or the plugin logic unless strictly necessary.
- For frontend changes, prefer editing existing files in `server/public/` rather than introducing new build tools.
- When adding a Markdown library (e.g., `marked` or `markdown-it`), add it to `server/package.json` and serve it as a static asset if possible.

## Workflow
- First, analyze `server/public/` to find where Chinese text is defined (HTML files, JS strings).
- Propose a step-by-step plan before making changes. Use **Plan Mode**.
- After changes, verify by running the server and checking the UI.
