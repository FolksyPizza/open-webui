# Open WebUI — Architecture map for Claude sessions

Read this first. It's a pointer file, not a spec — open the referenced files for detail.

## Stack
- Frontend: SvelteKit (Svelte 4), Tailwind CSS, paneforge for resizable panels, CodeMirror, Pyodide worker for in-browser Python.
- Backend: FastAPI + SQLAlchemy, Alembic migrations, Socket.IO for streaming, SSE-style chat events.
- Data: SQLite default; chat JSON lives on the `chats.chat` column.

## Chat streaming pipeline
- Backend stream emit: `backend/open_webui/socket/main.py`, `backend/open_webui/routers/chats.py`.
- Event shape: `{ type, data }`. Event types include `chat:message:delta`, `chat:message`, `chat:message:files`, `chat:message:embeds`, `chat:message:follow_ups`.
- Frontend dispatch: `src/lib/components/chat/Chat.svelte` ~lines 490–510. Add new event branches here.
- Per-token streaming animation: `src/lib/components/chat/Messages/Markdown/MarkdownInlineTokens/TextToken.svelte` + `.fade-in-token` in `src/app.css`.

## Artifacts (current + target)
- Renderer: `src/lib/components/chat/Artifacts.svelte` — iframe HTML/CSS/JS with CSP, SVG via `SvgPanZoom`, version pager.
- Extraction (post-hoc, from message content): `getCodeBlockContents()` in `src/lib/utils/index.ts` ~line 1871.
- Stores: `showArtifacts`, `artifactCode`, `artifactContents` in `src/lib/stores/index.ts` ~line 100.
- Target (in plan): inline artifact card for html/svg/mermaid/code/react in the message column; right-side `FilePreviewPane` for pdf/docx/image/markdown-report file previews and model-emitted file links; per-artifact identity with versions via a `canvas` backend tool emitting `chat:artifact:create|update`; `/artifacts` library route.

## Right-side pane shell
- `src/lib/components/chat/ChatControls.svelte` owns the paneforge `<Pane>` on large screens, Drawer on mobile.
- Currently tabs Controls / Files / Overview. New modes (Canvas, FilePreview, future Thinking sidebar) mount in the same slot, mutually exclusive.

## Settings (current + target)
- Current: modal at `src/lib/components/chat/SettingsModal.svelte` with per-tab components under `src/lib/components/chat/Settings/`.
- Target (in plan): dedicated route `src/routes/(app)/settings/+page.svelte` reusing the same tab components. Modal remains for quick access. New **Capabilities** tab toggles per-user `memory`, `web_search`, `code_execution_and_files`, `image_generation`, `calendar`, `canvas` + tool loading mode (`auto` / `always_loaded` / `on_demand`).
- Admin panel route untouched at `/admin`.

## Projects (was Folders)
- Backend model: `backend/open_webui/models/folders.py`. Router: `backend/open_webui/routers/folders.py`.
- Keep the table name, semantically rename to Project. Additive Alembic migration adds `system_prompt`, `instructions`, and a new `project_files` table.
- Project files extracted at upload via `backend/open_webui/retrieval/` and made available to every chat in the project as an implicit RAG source.
- Route target: `src/routes/(app)/projects/[id]/+page.svelte`. Sidebar (`src/lib/components/layout/Sidebar.svelte`) renames Folders → Projects.

## Tool / function registry
- Backend: `backend/open_webui/models/functions.py`, `backend/open_webui/routers/functions.py`, schema assembly in `backend/open_webui/utils/tools.py`.
- Frontend API client: `src/lib/apis/functions/index.ts`.
- New built-in tools (planned): `canvas` (create/update), `memory`, `calendar`. Registration pattern lives in `utils/tools.py`.

## Capabilities-driven tool routing (planned)
- New `user.capabilities` JSON column (Alembic). Per-chat overrides on the chat object.
- Chat-turn assembly merges user.capabilities ∪ chat.capability_overrides → tool defs included for the turn. Removes per-message toggles; chat `+` menu becomes a per-chat override layer only.

## Files / retrieval
- Loaders: `backend/open_webui/retrieval/loaders/` — many type-specific paths today; planned unified loader using `unstructured` as default (already in `backend/requirements.txt`).
- Pyodide worker: `src/lib/pyodide/pyodideKernel.ts`, `src/lib/workers/pyodide.worker.ts`. Generated files surface as message file objects.
- Model-emitted file links → render as inline file chips (planned) → open right-side `FilePreviewPane`.

## Visual / UX constraint
**Uniform with existing UI.** New surfaces reuse the existing component primitives: tab styling from `SettingsModal.svelte`, sidebar list rows from `layout/Sidebar.svelte`, buttons / inputs / modal shells from `lib/components/common/`. Same Tailwind utility classes, same border-radius, same Lucide icons, same paneforge layout for right-pane work. Do not introduce a new visual vocabulary.

## Migrations
- Every planned migration is additive. Old chats / folders / users keep working without backfill. Migrations live in `backend/open_webui/migrations/versions/`.

## Run locally (dev)
- Frontend: `npm run dev` → http://localhost:5173.
- Backend: `/home/william/.venvs-open-webui/bin/python -m uvicorn open_webui.main:app --port 8080 --host 0.0.0.0 --forwarded-allow-ips '*'` from `backend/`. CORS already allows 5173.
- The Python venv is intentionally outside the repo (inotify watch limits with torch headers under `.venv`).

## Plan
The active multi-phase plan lives at `~/.claude/plans/we-want-to-have-fizzy-puppy.md`. Phases: 0 (this file), 1 Settings+Capabilities, 2 Tool routing, 3 Projects, 4 Artifacts split, 5 Files, 6 Tool prompts/loading. Roadmap: Memory, CoT sidebar, Safety classifier, Deep research, Calendar tool, Audit/Rate-limit/Analytics/Scheduled tasks.
