# cloudassistant

This project is a minimal personal assistant website (local-first). It provides a simple chat interface plus built-in management for to‑dos, notes and reminders — all stored in the browser `localStorage`.

Quick start
1. Install dependencies:

```bash
cd /workspaces/cloudassistant
npm install
```

2. Run the server:

```bash
npm start
# open http://localhost:3000 in your browser
```

What you get
- A single-page UI with: `Chat`, `To-dos`, `Notes`, and `Reminders` views.
- Local storage persistence for to-dos and notes.
- Browser notifications for scheduled reminders (you will be asked for permission).

Next steps / Improvements
- Add user accounts and server-side persistence (database).
- Integrate an LLM (OpenAI / other) to provide smarter chat replies (this project is intentionally local-first).
- Add OAuth, syncing across devices, and calendar integrations.

Cloudflare Workers deployment (Google Drive + Gemini integration)

This project now includes a Cloudflare Worker that:
- Handles Google OAuth2 for Drive readonly access (`/auth/google/start` and `/auth/google/callback`).
- Stores tokens into a KV namespace binding named `TOKENS`.
- Provides a Drive file list endpoint at `/api/drive/list?email=you@example.com`.
- Proxies chat requests to an external Gemini chatbot at `/api/chat/gemini` and can enrich prompts with Drive metadata.

Files added for Workers
- `wrangler.toml` — Worker config and KV binding placeholder.
- `worker/index.js` — Worker script implementing OAuth, Drive integration, and Gemini proxy.

Setup steps (summary)
1. Install `wrangler` (or use the included `npm run deploy:worker` which expects a global `wrangler` binary):

```bash
npm install
npm run deploy:worker # this runs `wrangler publish` so you must have wrangler installed and authenticated
```

2. Create a Cloudflare Workers KV namespace and add its ID to `wrangler.toml` under `kv_namespaces` for the `TOKENS` binding (or use `wrangler kv:namespace create TOKENS` — then copy the resulting id into `wrangler.toml`).

3. Create Google OAuth2 credentials (OAuth client ID) in Google Cloud Console:
	- Authorized redirect URI should be: `https://<your-worker-domain>/auth/google/callback`
	- Enable the Drive API for the project and request the scope: `https://www.googleapis.com/auth/drive.readonly`

4. Set Worker secrets (use `wrangler secret put` or set in Cloudflare dashboard):

```bash
wrangler secret put GOOGLE_CLIENT_ID
wrangler secret put GOOGLE_CLIENT_SECRET
wrangler secret put GEMINI_API_KEY
# Optionally set GEMINI_API_URL if it differs from default
```

5. Publish the Worker with `wrangler publish` or `npm run deploy:worker`.

Notes and security
- Tokens are stored in KV bound to `TOKENS` under keys `google:email@example.com`.
- For production, restrict access to the Worker endpoints (Cloudflare Access, authenticated front-end, or signed requests).
- The Gemini proxy requires a valid `GEMINI_API_KEY` and `GEMINI_API_URL` (replace placeholders with the real API endpoint).

If you'd like, I can:
- Add a simple login flow in the frontend to open `/auth/google/start` and finish auth.
- Implement secure session handling so the frontend can call `/api/drive/list` without exposing KV keys.
- Add GitHub Actions to publish the Worker automatically on push.

What I implemented for beginners (done)
- A Sign in with Google button in the UI that opens the Worker OAuth flow in a popup and automatically notifies the page when complete.
- The Worker now returns a small HTML page on the OAuth callback which posts a message to the opener and closes the popup (so sign-in is easy).
- Automatic token refresh is attempted when calling the Drive list endpoint; refreshed access tokens are stored back into KV.
- A GitHub Actions workflow file at `.github/workflows/deploy-worker.yml` which runs `wrangler publish` on pushes to `main` (requires `CF_API_TOKEN` secret in your GitHub repo).

Beginner-friendly deploy checklist
1. Create a Cloudflare account and install `wrangler` locally for testing (optional):

```bash
npm install
npm run deploy:worker # publishes the Worker using the wrangler binary from devDependencies
# or: npx wrangler publish
```

2. Create a Workers KV namespace:

```bash
# create namespace locally via wrangler
npx wrangler kv:namespace create TOKENS --preview
# or create in Cloudflare dashboard and copy the id into `wrangler.toml` under kv_namespaces
```

3. Set Worker secrets (use `wrangler secret put`):

```bash
npx wrangler secret put GOOGLE_CLIENT_ID
npx wrangler secret put GOOGLE_CLIENT_SECRET
npx wrangler secret put GEMINI_API_KEY
# (optional) npx wrangler secret put GEMINI_API_URL
```

4. Configure Google OAuth credentials:
	- In Google Cloud Console create OAuth 2.0 Client ID credentials.
	- Set Authorized redirect URI to `https://<your-worker-host>/auth/google/callback` (replace with your Worker domain).
	- Enable the Drive API and request the `https://www.googleapis.com/auth/drive.readonly` scope.

5. Add `CF_API_TOKEN` in GitHub repository Secrets (with permissions to publish Workers and manage KV if you want the Action to create bindings).

6. Deploy by pushing to `main` (Action will run) or run `npm run deploy:worker` locally.

Note about Wrangler commands:

Recent versions of the `wrangler` CLI use the `publish` subcommand rather than `deploy`. If you see an error such as `Unknown argument: deploy` in your build logs, replace `wrangler deploy` with `wrangler publish` or use the npm script provided in this repo:

```bash
# publish the Worker using the wrangler CLI
npx wrangler publish
# or use the npm alias in this repo
npm run publish
```

If your CI or another tool is running `npx wrangler deploy`, update it to `npx wrangler publish`.

Automatically configuring the frontend to use your Worker


After you publish the Worker, you'll have a workers.dev URL (for example `https://cloudassistant-worker.janedoe.workers.dev`). To make the frontend call that worker without rebuilding, create a `public/worker-config.json` file containing your deployed URL.

For convenience we've added a tiny script to create that (local-only) file:

```bash
# from the repo root
npm run set-worker-base -- https://cloudassistant-worker.janedoe.workers.dev
```

Important: the repo intentionally ignores `public/worker-config.json` (see `.gitignore`) so it is safe to create locally without committing it to GitHub. Instead, we've committed `public/worker-config.example.json` as a template you can copy or use with the script.

This approach ensures no Google Drive content, tokens, or other secrets are stored in the repository. Always set credentials/secrets using `wrangler secret put` or GitHub repository secrets — never commit them.


Running locally for development
- The frontend is static in `public/` and can be served via the included Express server (`npm start`) for quick testing. OAuth flows require a publicly reachable redirect, so testing the full Google OAuth flow is easiest after publishing the Worker to Cloudflare (or using a tunnel that maps to your Worker URL).



google and msft assistant
