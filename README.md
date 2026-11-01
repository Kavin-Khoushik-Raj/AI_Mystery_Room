# AI Mystery Room

Responsive React mystery game with an Express/MongoDB API foundation. The current browser experience runs in local demo mode and stores cases in browser storage. The UI does **not yet call the API**, so deploying it will publish the demo game; it will not turn on account-backed AI case generation. The API endpoints are available separately at `/api` for continued integration.

## Deploy to Render

This repository includes `render.yaml`. Render builds the React client and runs Express as one web service, so the site and API share an HTTPS origin.

### 1. Prepare the project

1. Make sure `server/.env` contains your local credentials and is never committed. `.gitignore` excludes it.
2. Create a private or public GitHub repository.
3. From this project folder, run:

   ```powershell
   git init
   git add .
   git status
   ```

   Check that `server/.env` and all `node_modules` directories are absent from the staged files. Then commit and push:

   ```powershell
   git commit -m "Prepare AI Mystery Room deployment"
   git branch -M main
   git remote add origin https://github.com/YOUR-NAME/YOUR-REPOSITORY.git
   git push -u origin main
   ```

### 2. Create MongoDB Atlas access

1. Create an Atlas project and cluster, then create a database user with a strong password.
2. In **Network Access**, allow connections from your Render service. For tighter access, use Render's outbound IP ranges for the service's region (or dedicated outbound IPs where available). Avoid `0.0.0.0/0` for a production database; if you temporarily use it for a demo, keep a least-privilege database user and rotate the password afterward.
3. Copy the Atlas driver connection string. Replace the password placeholder and add the database name, for example `...mongodb.net/ai-mystery-room?retryWrites=true&w=majority`.

### 3. Deploy the Render Blueprint

1. Sign in to Render and choose **New → Blueprint**.
2. Connect the GitHub repository and select the `main` branch. Render reads `render.yaml` and creates the web service.
3. When prompted, provide `MONGO_URI` and `GEMINI_API_KEY`. Render generates `JWT_SECRET` for you. Keep secrets in Render's environment settings; never put production secrets in GitHub or `client/.env`.
4. Deploy. The configured build command is `npm ci && npm run build`; the start command is `npm start`.
5. Open the service URL. Check `/api/health` to confirm the API is up and MongoDB reports `connected`.

### 4. Verify and update

- The Render URL should open the React app. Refreshing `/dashboard`, `/create`, or a case route should return the app rather than a 404.
- `/api/health` should return `{ "success": true, "data": { "status": "ok", "database": "connected" }, "error": null }`.
- Push later changes to `main`; Render auto-deploys them.
- Add a custom domain from the Render service settings when ready. The single-origin setup needs no production `CLIENT_URL` or `VITE_API_URL` override.

### Deployment limitations in this version

- Gameplay currently uses deterministic browser demo data and local storage. Browser data is specific to that browser/device and can be cleared by the user.
- The React app is not yet connected to server auth, case creation, or gameplay endpoints. Supplying a Gemini key does not change the current UI's demo generation behavior.
- The API uses a placeholder evidence builder; full Gemini evidence generation and server-backed progress/scoring still need completion before this is a public production game.
- Render's free web services spin down after 15 minutes idle, so the next visit can have a cold start. The free plan is intended for demos rather than production traffic.

## Local development

1. Run `npm install` from the repository root.
2. Copy `server/.env.example` to `server/.env`, then set `MONGO_URI` and a `JWT_SECRET` of at least 32 characters. `GEMINI_API_KEY` is needed for the API's blueprint-generation endpoint.
3. Run `npm run dev` for the Vite client and Express API.

See `spec.md` for the full product requirements and `DECISIONS.md` for implementation choices.
