This is a comprehensive plan to refactor the LibreChat application into a backend-less, static web app that connects directly to the Gemini API.

### 1. Google Cloud Project Setup

1.  **Create a Google Cloud Project:**
    *   Go to the [Google Cloud Console](https://console.cloud.google.com/).
    *   Create a new project.
2.  **Enable the Gemini API:**
    *   In your new project, navigate to "APIs & Services" > "Library".
    *   Search for "Gemini API" and enable it.
3.  **Configure OAuth 2.0 Consent Screen:**
    *   Go to "APIs & Services" > "OAuth consent screen".
    *   Choose "External" user type.
    *   Fill in the required application details.
    *   Add the necessary scopes for the Gemini API. You will likely need `https://www.googleapis.com/auth/userinfo.email` and `https://www.googleapis.com/auth/userinfo.profile` for user authentication, and potentially `https://www.googleapis.com/auth/cloud-platform` for API access, though you should verify the exact scopes needed for Gemini.
4.  **Create OAuth 2.0 Client ID:**
    *   Go to "APIs & Services" > "Credentials".
    *   Click "Create Credentials" > "OAuth client ID".
    *   Select "Web application" as the application type.
    *   Add your authorized JavaScript origins (e.g., `http://localhost:3000`, `https://your-hostname.com`).
    *   Add your authorized redirect URIs. For a purely client-side app, this will be the URL of your application where Google will redirect after authentication.
    *   Take note of the **Client ID**. You will need it in your frontend application.

    ### Project secrets (SPA-only, prescriptive)

    This project is strictly SPA-only. No backend will be added. Do not design or implement server-side flows.

    In-repo allowed values:
    * OAuth Client ID (public) — put in a build-time env var (example: `VITE_GOOGLE_CLIENT_ID`) or commit it. This is not a secret.
    * Project ID, authorized origins, redirect URIs — safe to include.

    Items that must NOT appear in the repo or client code:
    * OAuth Client Secret
    * Service account private keys / JSON files
    * Server-side API keys or project-level keys that grant access without user tokens
    * Long-lived refresh tokens stored server-side

    Handling rule: if any sensitive items exist in the repository, remove them immediately; this project will not use or store confidential server credentials.

### 2. Frontend Refactoring

#### Authentication

*   **Remove existing authentication:** All code related to username/password login, registration, and user management in the client should be removed. This includes components, hooks, and store slices.
*   **Implement Google OAuth 2.0:**
    *   Use a library like `@react-oauth/google` to simplify the integration.
    *   In your main application component (likely `App.jsx` or a similar entry point), wrap your application with the `GoogleOAuthProvider`, providing your Client ID.
    *   Create a login component that uses the `useGoogleLogin` hook or the `GoogleLogin` button component to initiate the OAuth flow.
    *   Store the access token received from Google in memory or a secure part of your application's state. Do not store the client secret in the repo or in client code. The library will handle token management. See "Project secrets" above for the short guidance on what may be committed and what must not.

#### API Interaction

*   **Remove `client/src/data-provider/`:** Delete proxy/server-call logic in `client/src/data-provider/`. Replace it with `client/src/services/gemini.js` which makes direct, authenticated calls to Gemini from the browser using the user's OAuth token.
*   **Create a Gemini API service:**
    *   Create a new file, e.g., `client/src/services/gemini.js`.
    *   This service will be responsible for making authenticated requests to the Gemini API endpoint (`https://generativelanguage.googleapis.com`).
    *   Use a library like `axios` or the native `fetch` API to make the requests.
    *   Each request to the Gemini API must include the user's access token in the `Authorization` header: `Authorization: Bearer <access_token>`. Do not embed any private project API keys or service account credentials in the frontend. This project uses user tokens in-browser only; there is no server-side credential storage.
    *   Implement functions for the different Gemini API actions you need, such as `generateContent`.

#### Data Storage

*   **Remove Redux-persist middleware for remote synchronization:** Remove any middleware that syncs application state to remote servers; persist chat data only to local IndexedDB.
*   **Implement IndexedDB storage:**
    *   Use a library like `dexie.js` to simplify working with IndexedDB.
    *   Define a schema for your local database, which will include a table for `chats` (or `conversations`) and `messages`.
    *   Refactor the Redux store actions and reducers (or whatever state management solution is in use) to read from and write to IndexedDB instead of making API calls to the backend for data persistence.
    *   When the application loads, fetch the chat history from IndexedDB and populate the state.
    *   When a new chat is created or a message is sent/received, update both the application state and the corresponding tables in IndexedDB.

### 3. Build Process Simplification

*   **Modify `vite.config.ts`:**
    *   The `client/vite.config.ts` file will be your primary build configuration.
    *   Ensure the `build.outDir` is set to a suitable directory (e.g., `dist` or `build`).
    *   Remove any dev proxy settings that forward API requests to remote servers; configure the app to call the Gemini endpoints directly from the browser using user OAuth tokens.
*   **Update `package.json` scripts:**
    *   The `client/package.json` should be the main `package.json` for the project. You can move it to the root or keep it as is, but the root `package.json` will be mostly redundant.
    *   Remove any scripts that build or start server processes; keep only frontend scripts (`dev`, `build`, `preview`) for the static SPA.
    *   The main scripts will be:
        *   `dev`: `vite` (to run the dev server)
        *   `build`: `vite build` (to create a static production build)
        *   `preview`: `vite preview` (to preview the production build locally)

### 4. Code Deletion and Simplification

The following files and directories can be **completely deleted**:

*   `api/`
*   `config/`
*   `e2e/`
*   `helm/`
*   `packages/` (after verifying they don't contain critical frontend logic you want to keep)
*   `redis-config/`
*   `docker-compose.yml`
*   `docker-compose.override.yml.example`
*   `Dockerfile`
*   `Dockerfile.multi`
*   `bun.lockb`
*   `librechat.example.yaml`
*   `rag.yml`
*   The root `package.json` and `package-lock.json` (or merge necessary parts into `client/package.json` and move it to the root).

The following parts of the **frontend code** should be **refactored or removed**:

*   `client/src/data-provider/`: Delete.
*   `client/src/store/`:
    *   Remove slices related to user authentication, registration, and backend-specific state.
    *   Refactor conversation/chat slices to use IndexedDB.
*   `client/src/hooks/`: Review and remove any hooks that interact with the now-deleted backend API.
*   `client/src/components/`:
    *   Remove components for server-side account management (username/password registration, server-stored profiles). Keep only the OAuth-based login flow and client-side profile UI stored in IndexedDB.
    *   Refactor components that fetch data to use the new Gemini API service and read from the local IndexedDB-powered store.

### 5. Testing Strategy

*   **Delete server-dependent tests:** Remove any tests that require a server. Delete the `api/` and `e2e/` test directories; keep and adapt frontend unit tests to mock the Gemini API and IndexedDB.
*   **Keep and adapt frontend tests:**
    *   Unit tests for components and utility functions in the `client/src/` directory should be kept.
    *   You will need to update these tests to mock the Gemini API service and IndexedDB instead of the old data provider.
    *   Consider using libraries like `testing-library/react` to test your components' behavior in a more user-centric way.
*   **Add new tests:**
    *   Write new unit tests for the Gemini API service to ensure it constructs requests correctly.
    *   Write tests for your IndexedDB data access layer to ensure data is stored and retrieved correctly.

### Initial implementation steps (practical runbook)

The following ordered steps turn the plan above into an initial implementation you can follow immediately. Treat these as the "first commits" and small, testable milestones.

1. Verify Gemini browser compatibility (CORS & streaming)
    - Create a tiny static HTML + JS page that performs a bearer-auth fetch to a Gemini endpoint (use a temporary OAuth access token obtained via OAuth playground or your Google account).
    - Confirm whether responses can be fetched from the browser and whether the API supports streaming (ReadableStream) or requires polling.
    - If CORS blocks direct calls, stop and re-evaluate: a server proxy would be required (but this project policy forbids it).

2. Register and configure Google Cloud (documented earlier)
    - Create a project, enable Gemini API, configure OAuth consent screen, and create a Web OAuth Client ID.
    - Add your local/dev origin (e.g., `http://localhost:5173`) and your production hostname to Authorized JavaScript origins.
    - Save the Client ID to your build environment as `VITE_GOOGLE_CLIENT_ID` (no client secret in repo).

3. Add a minimal PKCE-based OAuth integration in the app
    - Install and wire `@react-oauth/google` (or implement manual PKCE) and wrap `App` with `GoogleOAuthProvider` using `VITE_GOOGLE_CLIENT_ID`.
    - Implement a compact login UI that requests the scopes you chose. Persist only the access token in memory or ephemeral state; prefer in-memory and Redux state rather than localStorage unless necessary.
    - Add a simple token-refresh strategy: detect 401s from Gemini and trigger a silent re-auth (or re-initiate auth). Document whether refresh tokens are available for your OAuth client; do not commit any refresh-token storage in repo.

4. Create `client/src/services/gemini.js`
    - Implement a small wrapper that accepts an access token and issues authenticated requests to `https://generativelanguage.googleapis.com`.
    - Provide two modes: non-streaming (single request/response) and streaming (readable stream) if the API supports it.
    - Keep the surface minimal: `createChat`, `sendMessage`, `streamResponse`.

5. Implement IndexedDB storage (Dexie)
    - Add `dexie` and create a compact DB schema: `conversations` and `messages` tables with sensible indices.
    - Implement a thin data-access layer `client/src/services/storage.js` that exports `loadConversations()`, `saveMessage()`, `createConversation()`.
    - Unit-test this layer with in-memory IndexedDB mocks.

6. Refactor the Redux store and components incrementally
    - Replace backend data-provider calls with calls to `services/gemini.js` for generation and `services/storage.js` for persistence.
    - Remove auth slices and server-dependent slices; add a small `auth` slice that stores the current user's basic profile and access token (ephemeral).
    - Migrate only one chat flow first (happy path): user signs in, opens a conversation, sends a prompt, receives a streamed response, and the whole conversation is saved in IndexedDB.

7. Update build and repository layout
    - Remove server code and vendor directories as planned (or move them to an archive branch). Keep the `client/` app as the primary source and update root scripts to call `client` scripts or move `client/package.json` to root.
    - Remove dev proxy configuration from `vite.config.*` and ensure build outputs a static site (`dist`).

8. Tests and CI
    - Update frontend tests to mock `services/gemini.js` and `services/storage.js` instead of the old data-provider.
    - Add unit tests for the OAuth login flow (mocking the provider) and for the Gemini service request construction.

9. Security review and final checks
    - Scan the repo to ensure no secrets exist (client secrets, service account JSON, project API keys). Remove or rotate any found secrets.
    - Confirm that all network calls to Gemini use the user's bearer token only.

10. Small production smoke test
    - Build the static site and host it on a staging host with the OAuth origin configured. Test a full end-to-end sign-in and chat generation from a real browser.

Checklist mapping (requirements -> initial status):

* SPA-only, no backend: Planned (documented) — implement steps 4,5,6,7 above.
* Google OAuth (PKCE) for users: Planned — implement step 3 and 2 above.
* Use user access tokens for Gemini calls: Planned — implemented in step 4.
* Persist only to IndexedDB in browser: Planned — implement step 5 and 6.
* Remove backend, Docker, server tests: Planned — implement step 7 and 8.
* Verify Gemini CORS/streaming from browser: Blocker (must verify) — perform step 1 immediately.

Edge cases & notes
* Access token lifetime: SPAs may not receive long-lived refresh tokens. If you cannot obtain refresh tokens in your client OAuth config, implement a silent reauth / prompt flow and document UX implications.
* Rate-limits & billing: Since every user uses the project Gemini quota via their user token, confirm how billing/scopes are applied. Monitor and document limits during staging.
* Privacy: Chats are stored locally — document data export and deletion options in the UI.

By following these initial steps you will convert the high-level plan into small, verifiable commits and demos. Start with step 1 (CORS/stream spike) and step 2 (Google Cloud registration) in parallel.

By following this plan, you will be able to strip out the backend, simplify the codebase and build process, and create a modern, static web application that leverages the power of the Gemini API directly from the client-side.