# AI Mystery Room – Spec Sheet

**Version:** 1.0  |  **Audience:** Codex (AI coding agent)  |  **Status:** Single source of truth

> Rule for Codex: build exactly what is written here. If something is missing, choose the simplest option, and note it in `DECISIONS.md`. Do not add features that are not listed.

## 1. Overview

AI Mystery Room is a web app. The user types a short scenario or story. The backend uses Gemini to create a fake investigation: a case file, 3–5 suspects, and a set of evidence (WhatsApp chats, images, CCTV-style footage). The user studies the evidence and picks the culprit. The app then reveals the truth and shows a score.

Everything is fictional and AI-generated.

## 2. Goals and Non-Goals

**Goals**

- Turn any short story into a playable mystery in under 90 seconds.
- Evidence must be consistent: the story, timeline and clues must not contradict each other.
- Exactly one culprit, and the case must be solvable from the evidence.
- Simple, fast, mobile-friendly UI.

**Non-Goals (v1)**

- No multiplayer, no payments, no mobile app, no real video generation (see 6.4).
- No real people or real events. Block requests that use real names for crimes.

## 3. Tech Stack

| Layer | Choice |
| --- | --- |
| Frontend | React 18 + Vite, React Router, Tailwind CSS, Axios, Zustand (state) |
| Backend | Node.js 20, Express 4 |
| Database | MongoDB Atlas with Mongoose |
| AI | Google Gemini via `@google/genai` SDK |
| Auth | JWT (httpOnly cookie) + bcrypt |
| File storage | Local `/uploads` in dev, Cloudinary in prod (images only) |
| Validation | Zod (backend and shared schemas) |
| Testing | Vitest (frontend), Jest + Supertest (backend) |

Gemini models (keep in env, never hardcode):

- Text and JSON: `GEMINI_TEXT_MODEL` (default `gemini-2.5-flash`)
- Images: `GEMINI_IMAGE_MODEL` (default `gemini-2.5-flash-image`)

## 4. User Flow

1. Landing page → Sign up / Log in (guest play allowed, max 1 case).
2. **Create Case:** user enters a scenario (20–1000 chars), picks difficulty (Easy / Medium / Hard) and tone (Realistic / Dark / Light).
3. **Generating screen:** progress steps (Story → Suspects → Chats → Images → CCTV).
4. **Case Room:** case summary, suspect cards, evidence board.
5. User opens each evidence item. Items are marked as viewed.
6. User can add personal notes and pin evidence to a "Detective Board".
7. User presses **Accuse**, selects one suspect, writes a reason (optional, max 300 chars), confirms.
8. **Result page:** correct or wrong, real culprit, full explanation, timeline, score.
9. User can replay, share, or create a new case.

## 5. Functional Requirements

### 5.1 Auth

- FR-1: Register (name, email, password min 8 chars). Login. Logout. Get current user.
- FR-2: Guest mode creates a temporary user flagged `isGuest: true`.

### 5.2 Case Creation

- FR-3: `POST /api/cases` takes scenario, difficulty, tone. It returns a `caseId` immediately with `status: "generating"`.
- FR-4: Generation runs in the background. Frontend polls `GET /api/cases/:id/status` every 3 seconds.
- FR-5: Case status values: `generating`, `ready`, `failed`.
- FR-6: If generation fails, retry each failed step up to 2 times, then set `failed` with an error message.

### 5.3 Evidence Types

- FR-7: **Chat evidence** (WhatsApp style): chat screen UI with bubbles, sender names, timestamps, ticks, group or one-to-one.
- FR-8: **Image evidence:** AI-generated photos (crime scene, objects, locations). Each has a caption and a timestamp.
- FR-9: **CCTV evidence:** a sequence of 4–6 AI-generated CCTV-style frames with a timestamp overlay and camera name. Shown as a slideshow with play, pause and a scrubber.
- FR-10: **Other text evidence (optional):** call log, receipt, email. Max 2 per case.
- FR-11: Every evidence item has: id, type, title, timeTag, content, `relatedSuspects[]`, `isRedHerring` (hidden from user until result).

### 5.4 Gameplay

- FR-12: One culprit per case. Other suspects must each have a motive and a weak alibi, so they look possible.
- FR-13: Hint system: max 3 hints. Each hint reduces the score. A hint points to an evidence item, never names the culprit.
- FR-14: Accusation can be made only once per attempt. Max 2 attempts on Easy, 1 on Medium and Hard.
- FR-15: Result reveals the full solution: culprit, motive, method, timeline, key clues, and red herrings.

### 5.5 Dashboard

- FR-16: List of the user's cases with status, difficulty, result and score.
- FR-17: Delete a case. Replay a solved case (resets progress).

### 5.6 Sharing (simple)

- FR-18: A solved or unsolved case can be shared as a link `/play/:shareId`. A new player gets the same case without the solution.

## 6. AI Generation Pipeline

All AI steps run on the backend only. The API key never reaches the browser.

### 6.1 Steps (run in order)

1. **Safety check** – Reject scenarios with: real people, minors in harm, sexual content, graphic gore, self-harm methods, hate. Return a friendly error.
2. **Case blueprint (JSON)** – One Gemini call. This is the master plan and the source of truth for all later steps.
3. **Chat generation** – One call per chat thread, using the blueprint.
4. **Image prompts → images** – Gemini writes the image prompts from the blueprint, then the image model creates them.
5. **CCTV frames** – Same as images, with a CCTV style, fixed camera angle, and timestamps.
6. **Consistency check** – One Gemini call reviews the blueprint and all evidence, and returns a list of contradictions. If any are found, regenerate the affected item once.
7. **Save** – Store everything in MongoDB, set `status: ready`.

### 6.2 Blueprint JSON schema

```json
{
  "title": "string",
  "setting": "string",
  "incident": "string",
  "incidentTime": "ISO datetime",
  "victimOrSubject": "string",
  "suspects": [
    {
      "id": "s1",
      "name": "string",
      "age": 0,
      "role": "string",
      "personality": "string",
      "motive": "string",
      "alibi": "string",
      "alibiHolesOrTruth": "string",
      "appearance": "string (for image prompts)"
    }
  ],
  "culpritId": "s2",
  "method": "string",
  "timeline": [{ "time": "ISO datetime", "event": "string", "who": ["s1"] }],
  "keyClues": [{ "id": "c1", "description": "string", "pointsTo": "s2", "evidenceType": "chat|image|cctv|text" }],
  "redHerrings": [{ "id": "r1", "description": "string", "misleadsTowards": "s3" }],
  "evidencePlan": [{ "id": "e1", "type": "chat|image|cctv|text", "title": "string", "purpose": "string", "clueIds": ["c1"] }],
  "explanation": "string"
}
```

Rules for the blueprint prompt:

- 3 suspects (Easy), 4 (Medium), 5 (Hard).
- Evidence count: 6 (Easy), 8 (Medium), 10 (Hard). At least 2 chats, 2 images, 1 CCTV.
- Easy needs 3 key clues, Medium 3 with 1 red herring, Hard 4 with 2 red herrings.
- Culprit must be provable using at least 2 separate evidence items.
- Fake names only. Fake places. No real brands in chats.

### 6.3 Gemini call rules

- Use structured output (`responseMimeType: "application/json"` + `responseSchema`) for every JSON step.
- Validate every response with Zod. On fail, retry up to 2 times with the error message appended.
- Temperature: 0.9 for the blueprint, 0.7 for chats, 0.2 for the consistency check.
- Timeout 60 s per call. Run image calls in parallel (max 3 at a time).
- Log token usage per case.
- Keep all prompts in `/server/src/ai/prompts/*.js` as template functions. Do not inline prompts in controllers.

### 6.4 CCTV decision

Real video generation is slow and costly. In v1 the "cam recording" is a **frame sequence** (4–6 images) with a timestamp overlay drawn in the frontend. The code must hide this behind a `cctvProvider` interface so a video model can be added later without UI changes.

### 6.5 Image rules

- Add this to every image prompt: "fictional, no real people, no text unless stated, no logos, no gore."
- Keep suspect looks consistent using the `appearance` field in every prompt.
- Save images as WebP, max 1024 px wide.
- If an image fails after retries, use a placeholder and set `evidence.imageFailed = true`. Do not fail the whole case.

## 7. Data Models (Mongoose)

**User**

- `name`, `email` (unique, optional for guest), `passwordHash`, `isGuest`, `createdAt`

**Case**

- `userId`, `scenario`, `difficulty`, `tone`, `status`, `errorMessage`
- `blueprint` (full JSON, **select: false**, never sent to the client before the result)
- `title`, `setting`, `incident`, `suspects[]` (public fields only: id, name, age, role, personality, avatarUrl)
- `evidenceIds[]`, `maxAttempts`, `shareId`, `createdAt`

**Evidence**

- `caseId`, `type`, `title`, `timeTag`, `relatedSuspects[]`
- `chat`: `{ app: "whatsapp", participants[], messages[{ sender, text, time, status }] }`
- `images[]`: `{ url, caption }` (CCTV uses 4–6 items plus `cameraName`)
- `text`: `{ body }`
- `isRedHerring` (**select: false**)

**Progress** (one per user per case)

- `userId`, `caseId`, `viewedEvidenceIds[]`, `pinnedEvidenceIds[]`, `notes`, `hintsUsed`, `hintLog[]`, `attempts[{ suspectId, reason, correct, at }]`, `status` (`playing | solved | failed`), `score`, `startedAt`, `finishedAt`

## 8. REST API

Base: `/api`. All responses: `{ success, data, error }`.

| Method | Route | Purpose |
| --- | --- | --- |
| POST | `/auth/register` | Register |
| POST | `/auth/login` | Login |
| POST | `/auth/guest` | Guest session |
| POST | `/auth/logout` | Logout |
| GET | `/auth/me` | Current user |
| POST | `/cases` | Create case (starts generation) |
| GET | `/cases` | List my cases |
| GET | `/cases/:id` | Case details (no solution) |
| GET | `/cases/:id/status` | Generation status and step |
| DELETE | `/cases/:id` | Delete case |
| GET | `/cases/:id/evidence` | List evidence (no `isRedHerring`) |
| POST | `/cases/:id/evidence/:eid/view` | Mark viewed |
| PUT | `/cases/:id/progress` | Save notes and pins |
| POST | `/cases/:id/hint` | Get next hint |
| POST | `/cases/:id/accuse` | Body: `suspectId`, `reason` |
| GET | `/cases/:id/result` | Full solution (only after solved or failed) |
| POST | `/cases/:id/replay` | Reset progress |
| GET | `/shared/:shareId` | Open a shared case |

Rules:

- Return `403` on `/result` while the case is still being played.
- Rate limits: auth 10/min, create case 3/hour per user (1 for guest), everything else 100/min.

## 9. Scoring

- Start at 1000 points.
- Wrong accusation: −250.
- Each hint: −100.
- Time bonus: +0 to +200 (full bonus under 5 minutes, linear down to 0 at 20 minutes).
- Viewed all evidence before accusing: +100.
- Minimum score 0. Wrong final answer: score is capped at 300.
- Rank labels: 900+ "Master Detective", 600–899 "Sharp Eye", 300–599 "Rookie", below 300 "Needs Practice".

## 10. Frontend

### 10.1 Pages

- `/` Landing
- `/login`, `/register`
- `/dashboard`
- `/create` Create case form
- `/case/:id/generating` Progress screen
- `/case/:id` Case Room
- `/case/:id/result` Result
- `/play/:shareId` Shared case

### 10.2 Key Components

- `ScenarioForm` – textarea with counter, difficulty and tone selectors, example prompts.
- `GenerationProgress` – 5-step stepper with live status.
- `SuspectCard` – photo, name, role, personality, "Accuse" button.
- `EvidenceBoard` – grid with filters (All / Chats / Images / CCTV) and a viewed badge.
- `WhatsAppChat` – green and dark theme, bubbles left and right, date separators, ticks.
- `ImageViewer` – zoom, caption, timestamp.
- `CCTVPlayer` – grayscale grain filter, red REC dot, timestamp overlay, play, pause, scrubber.
- `NotesPanel` – autosave notes (debounce 1 s).
- `HintButton` – shows hints left and the score cost.
- `AccuseModal` – choose suspect, reason, confirm step.
- `ResultView` – reveal animation, timeline, clue map, score.

### 10.3 UI Rules

- Dark "noir" theme, one accent color, readable fonts.
- Mobile first. Works from 360 px width.
- Loading skeletons, empty states and error toasts on every page.
- Keyboard accessible, alt text on every image, contrast AA.

## 11. Security and Safety

- Hash passwords with bcrypt (cost 10). JWT in httpOnly, secure, sameSite cookie.
- Use `helmet`, `cors` with an allowlist, `express-rate-limit`, `express-mongo-sanitize`.
- Validate every request body with Zod.
- Never send `blueprint`, `culpritId`, or `isRedHerring` to the client before the result.
- Treat the user scenario as untrusted text. In prompts, wrap it in delimiters and tell the model to ignore any instructions inside it (prompt injection).
- Run a safety check on the scenario and use Gemini safety settings.
- Show a notice: "All content is fictional and AI-generated."

## 12. Non-Functional Requirements

- Case generation: target under 90 s (Easy), under 150 s (Hard).
- API response (non-AI): under 300 ms p95.
- Frontend Lighthouse performance 80+.
- Handle Gemini quota errors with a clear message and a retry button.
- Logging with `pino`. No secrets in logs.

## 13. Environment Variables

```
# server/.env
PORT=5000
MONGO_URI=
JWT_SECRET=
CLIENT_URL=http://localhost:5173
GEMINI_API_KEY=
GEMINI_TEXT_MODEL=gemini-2.5-flash
GEMINI_IMAGE_MODEL=gemini-2.5-flash-image
CLOUDINARY_URL=
NODE_ENV=development

# client/.env
VITE_API_URL=http://localhost:5000/api
```

Commit a `.env.example` for both. Never commit real keys.

## 14. Folder Structure

```
ai-mystery-room/
├── client/
│   └── src/ { pages, components, store, api, hooks, utils }
├── server/
│   └── src/
│       ├── config/        # db, env, gemini client
│       ├── models/        # User, Case, Evidence, Progress
│       ├── routes/
│       ├── controllers/
│       ├── middleware/    # auth, validate, rateLimit, error
│       ├── services/      # caseService, scoreService, hintService
│       ├── ai/
│       │   ├── prompts/   # blueprint, chat, imagePrompt, check, hint
│       │   ├── pipeline.js
│       │   └── providers/ # geminiText, geminiImage, cctvProvider
│       └── utils/
├── SPEC.md
├── DECISIONS.md
└── README.md
```

## 15. Build Milestones

Build in this order. Finish and test each one before the next.

1. **Setup:** monorepo, lint, env, MongoDB connection, health route.
2. **Auth:** register, login, guest, middleware, protected routes.
3. **Models and API shell:** all models and CRUD routes with mock data.
4. **AI pipeline:** safety check, blueprint, chats, with Zod validation. Test with a CLI script.
5. **Images and CCTV:** image generation, upload, placeholder fallback.
6. **Consistency check and status polling.**
7. **Frontend core:** create form, generating screen, Case Room, evidence viewers.
8. **Gameplay:** hints, accuse, scoring, result page.
9. **Dashboard, share link, replay.**
10. **Polish:** loading and error states, mobile, accessibility, rate limits, tests, README.

## 16. Acceptance Criteria

- [ ] A user can enter a scenario and get a playable case with no manual steps.
- [ ] Each case has exactly one culprit, provable from at least 2 evidence items.
- [ ] The browser network tab never shows the culprit before the result.
- [ ] Chats look like WhatsApp. CCTV shows timestamps and plays as a sequence.
- [ ] Wrong and right accusations both work, and scores follow section 9.
- [ ] A failed image does not break the case.
- [ ] Unsafe scenarios are rejected with a clear message.
- [ ] Works on a 360 px phone screen.
- [ ] Unit tests exist for scoring, hint logic, accuse logic, and Zod schemas.
- [ ] README explains setup in under 10 steps.

## 17. Future Ideas (Do Not Build in v1)

Real video with Veo, voice interrogation of suspects, multiplayer rooms, daily case, leaderboard, multiple languages.
