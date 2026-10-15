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

- Presentation tier decision: Keep React components responsible for rendering and user interaction. Load API data through the shared client using relative `/api` paths; let Vite proxy requests to the private Express port. Each capability shows loading, success, empty, and accessible error states.
- Logic tier decision: Keep endpoint wiring and HTTP status/response formatting in Express routes. Put validation, persistence orchestration, scoring, aggregation, and recommendation selection in the TypeScript service layer. Return deliberate 4xx responses for invalid input, malformed or missing references, and unknown users; do not expose Mongoose cast errors as 500 responses.
- Data tier decision: Define required fields, defaults, and ObjectId references in Mongoose schemas. Keep seed data representative across all four capabilities, use stable keys/upserts so repeated runs are idempotent, and never delete or overwrite unrelated records.
- Activity tracking approach: Validate the request shape and both IDs before writing. Confirm that the referenced user and team exist, then save the activity with those references, its type and minutes, computed points at one point per minute, and an occurrence timestamp. List persisted activities through the API.
- Team management approach: Require a non-empty name and mascot, default members to an empty array, and persist submitted members as user references. List teams with their names, mascots, and member IDs intact.
- Leaderboard approach: Aggregate activity points by team, include teams with no activity at zero points, sort by total points descending, and assign 1-based ranks in the returned response.
- Workout suggestions approach: Find the user by email; return a deliberate 4xx if there is no match. Prefer workouts matching the user's fitness goal or recent activity pattern, and include a short reason for the first recommendation.

## Test plan

Implement and validate one tier at a time. Complete the data models and seed behavior first, then the API services and routes, then the presentation components and shared client. Do not start the next tier until the current tier's focused contract tests pass. Preserve the documented contracts; do not weaken tests to accommodate an incomplete implementation.

- Data tier: `npm run test:data --prefix octofit-tracker/backend` checks required model fields, references/defaults, and repeatable seed behavior that preserves unrelated data. Failures point to schemas or seed logic.
- Logic/API tier: `npm run test:api --prefix octofit-tracker/backend` checks activity and team validation/create/list behavior, leaderboard totals and ordering, and user-specific workout recommendations. Failures point to service behavior, route responses, or their persistence integration.
- Backend typecheck: `npm run build --prefix octofit-tracker/backend` runs the TypeScript compiler without emitting files; failures indicate backend type or module-contract issues.
- Presentation tier: `npm test --prefix octofit-tracker/frontend` checks API-backed rendering, the requested workout user, and loading, empty, and accessible error states for each route. Failures point to component behavior or shared API-client usage.
- Frontend typecheck and production build: `npm run build --prefix octofit-tracker/frontend` runs TypeScript checking and Vite bundling; failures indicate frontend type, import, or build integration issues.
- Final regression check: `npm test --prefix octofit-tracker/backend` runs both backend contract suites together after the targeted checks pass.
