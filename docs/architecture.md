# OctoFit Tracker architecture

OctoFit Tracker is a three-tier application for a school fitness program. The starter code includes API contracts and intentionally incomplete implementation points so you can use GitHub Copilot agent mode in a test-driven loop.

![Three-tier OctoFit architecture](../.github/images/octofit-architecture.svg)

## Tier boundaries

- **Presentation tier**: React 19, Vite, React Router, Bootstrap, and a shared API client in `octofit-tracker/frontend/src`.
- **Logic tier**: Express and TypeScript routes/services in `octofit-tracker/backend/src` expose `/api/*` behavior on port `8000`.
- **Data tier**: Mongoose models and seed data in `octofit-tracker/backend/src/models` and `src/scripts/seed.ts` persist to MongoDB on port `27017`.

## Ports and environment rules

- MongoDB: `27017`
- Express API: `8000`
- Vite frontend: `5173`
- The browser calls relative `/api` paths. Vite proxies them to `http://localhost:8000`, so the unauthenticated API port stays private in Codespaces. `VITE_API_BASE_URL` can override the base URL.

## API contract

| Capability | Method and path | Expected behavior |
| --- | --- | --- |
| Activity tracking | `GET /api/activities`, `POST /api/activities` | List and create valid activity records. POST accepts `userId`, `teamId`, `type`, and `minutes`; `userId` and `teamId` must be well-formed IDs for an existing user and team; otherwise respond with a deliberate 4xx (never a 500 from a cast error). A created activity stores the submitted `user`, `team`, `type`, and `minutes` plus computed `points` and `occurredAt`. |
| Teams | `GET /api/teams`, `POST /api/teams` | List and create teams with members. Created and listed teams keep the submitted `name` and `mascot`. Reject a team without a name with `400`. |
| Leaderboard | `GET /api/leaderboard` | Return teams ordered by the sum of their activity points, highest first. Each entry has `teamName`, `totalPoints`, and a 1-based `rank`. |
| Workout suggestions | `GET /api/workouts/suggestions/:userEmail` | Recommend workouts from recent activity patterns for the matching seeded user's email address. The first suggestion's `focus` matches that user's goal or recent activity, with a `reason` explaining why. An email with no matching user returns a deliberate 4xx (for example, `404`). |

## Implementation decisions

Record your plan here before editing code.

- Presentation tier decision: Keep routing, rendering, and loading/empty/error states in React components. Components use the shared API client for relative `/api` paths; they do not call `fetch` directly or duplicate backend business rules.
- Logic tier decision: Keep HTTP parsing, validation, and response status handling in the Express API/service boundary. Implement behavior in `octofitService.ts`, preserve the existing route contract, and return deliberate 4xx responses for invalid references and unknown users rather than cast-error 500s.
- Data tier decision: Put required fields, references, and defaults in the Mongoose models. Keep demo seeding upsert-based and idempotent, without clearing or overwriting unrelated records.
- Activity tracking approach: Validate the submitted user/team IDs and confirm both records exist before writing. Require a supported activity type and positive minutes; store the references, type, and minutes with points computed from minutes and a server-generated `occurredAt`.
- Team management approach: Reject a missing or blank name with `400`, preserve the submitted name and mascot, and store the supplied member references. Listing returns the teams and their members.
- Leaderboard approach: Aggregate activity points by team, sort by total descending, and add 1-based ranks; return `teamName`, `totalPoints`, and `rank` for each team.
- Workout suggestions approach: Find the user by the requested email and return a deliberate 4xx if absent. Rank available workouts using the user's goal or recent activity, with the leading suggestion's focus aligned to that signal and a human-readable reason.

Smallest safe implementation order: (1) complete and verify model constraints while preserving the already-passing idempotent seed contract; (2) implement the API service behavior and validation, checking data and API contracts after each slice; (3) connect the four presentation components through the shared client and add accessible loading, empty, and error states; (4) run both builds as final type/bundle gates. This follows the data-to-logic-to-presentation dependency and avoids changing the existing router or port/proxy boundaries.

## Test plan

Record the commands you run and what each failure means.

- Data-tier command: `npm run test:data --prefix octofit-tracker/backend` — a failure points to Mongoose required fields/references/defaults or seed completeness, idempotency, and preservation of existing records. Not run separately yet; the combined backend run had 4 data-contract failures and the seed contract passed.
- API command: `npm run test:api --prefix octofit-tracker/backend` — a failure points to route/service behavior, input/reference validation and status codes, persistence/serialization, leaderboard aggregation/ranking, or personalized workout matching. Not run separately yet; the combined backend run had 4 API-contract failures: activity creation returned `501`, team creation returned `501`, leaderboard was empty, and Mona received the wrong first workout.
- Frontend command: `npm test --prefix octofit-tracker/frontend` — a failure points to API-client integration, rendered data, or accessible loading/empty/error states. Current run: all 15 tests fail; the components still show “Implementation pending” and do not issue the expected requests or states.
- Backend build command: `npm run build --prefix octofit-tracker/backend` — a failure indicates TypeScript errors or incompatible service/model/route contracts. Not run yet.
- Frontend build command: `npm run build --prefix octofit-tracker/frontend` — a failure indicates TypeScript or Vite production-build errors in the presentation tier or its API integration. Not run yet.
