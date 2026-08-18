## Setup

### Setup Steps

1. Verified the installed prerequisites: Python 3.11.9, Node.js, and Git.
2. Cloned Open WebUI into the separate `home-exercice` workspace.
3. Created the frontend environment file from `.env.example`.
4. Installed frontend dependencies with `npm install --force` because this computer has Node.js 24 while the project expects an older Node version.
5. Created `backend/venv` with Python 3.11 and installed `backend/requirements.txt`.
6. Configured an OpenRouter OpenAI-compatible connection for testing and selected the `openrouter/free` model.
7. Verified the frontend at `http://localhost:5173` or the next available Vite port.
8. Verified the backend API at `http://localhost:8080/docs`.

### Troubleshooting Encountered

- **Node version mismatch:** Node.js 24 is newer than the project's expected range. `npm install --force` was used to install the dependencies.
- **Frontend build memory:** The build initially ran out of memory. Use `NODE_OPTIONS=--max-old-space-size=8192 npm run build` when necessary.
- **Git Bash backend script failure:** Running `sh dev.sh` caused shell glob arguments to be passed to Uvicorn. The supported Windows batch file was used instead.
- **Windows secret-key generation failure:** `start_windows.bat` printed `The system cannot find the file specified` while generating `.webui_secret_key`. Defining `WEBUI_SECRET_KEY` manually before starting the batch file fixed it.
- **Chrome CORS workaround:** Normal Chrome worked with the configured CORS origin. An insecure Chrome profile is only a fallback for an actual CORS error and should not be used for normal browsing.

### Final Commands To Run

Use two separate terminals.

Frontend in Git Bash:

```bash
npm run dev
```

Backend in a native Windows Command Prompt:

```bat
cd /d C:\...\backend
call venv\Scripts\activate.bat
set "WEBUI_SECRET_KEY=xxxxx"
set "CORS_ALLOW_ORIGIN=http://localhost:5173;http://localhost:8080"
start_windows.bat
```

## Exercise

### 1. Expose The System Prompt

The normal message input remains the user's question. We added a separate System Prompt editor that appears after a file is uploaded. It lets the user define instructions about how the model should analyze the file. The value is sent in `params.system` in the chat completion request.

Files touched:

- `src/lib/components/chat/MessageInput.svelte` - added the System Prompt textarea to the file-upload composer.
- `src/lib/components/chat/Chat.svelte` - stores the prompt, resets/restores it with chat state, and copies the textarea value into `params.system` immediately before submission.


### 2. Optimize Interface Settings

The Interface settings previously called the persistence function whenever a control changed. We changed the Interface tab to collect changes in a local draft and send one filtered batch only when Save is clicked. Closing the dialog without saving discards the draft. Other Settings tabs were left unchanged.

Files touched:

- `src/lib/components/chat/Settings/Interface.svelte` - collects local changes and persists them on form submission.
- `src/lib/components/chat/SettingsModal.svelte` - passes the real persistence function to the Interface draft layer.


### 3. Add Chat Management

We added a Workspace Chats screen for managing conversations. It lists chats with their title, model, and updated date; supports sorting by date, model, and title; opens a chat when its title is clicked; and supports inline title editing with persistence. Folder-specific behavior, deletion, export, and advanced search were intentionally excluded from this exercise.

Files touched:

- `src/lib/components/workspace/Chats.svelte` - list UI, model loading, sorting, opening chats, and title editing.
- `src/routes/(app)/workspace/chats/+page.svelte` - new route page for `/workspace/chats`.
- `src/routes/(app)/workspace/+layout.svelte` - Chats tab, admin route protection, and chat count beside the tab.
- `src/lib/apis/chats/index.ts` - exposes `sort_by` and `sort_dir` query parameters for chat listing.