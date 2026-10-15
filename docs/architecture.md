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

- Presentation tier decision: Keep UI state and rendering in React components, with requests routed through the shared API client.
- Logic tier decision: Implement API validation and capability behavior in Express routes/services; keep persistence details in the data tier.
- Data tier decision: Add required Mongoose schema validation/defaults and preserve the existing idempotent seed behavior.
- Activity tracking approach: Validate request fields and well-formed, existing user/team IDs before creating records; compute points and `occurredAt` in the logic tier.
- Team management approach: Reject a missing name with `400`, and create/list teams with their submitted name, mascot, and members.
- Leaderboard approach: Aggregate activity points by team, order highest first, and assign 1-based ranks.
- Workout suggestions approach: Find the user by email, use their goal and recent activities to select a matching suggestion with an explanatory reason, and return a 4xx for unknown users.

Smallest safe implementation order: data schema contracts; activity/team API behavior; leaderboard and workout suggestions; presentation data and loading/empty/error states. Run the focused contract suite after each tier, then run all contract suites and both builds.

## Test plan

Baseline results recorded 2026-10-15. Implement in the order above and rerun these checks after changes.

- Data-tier command: `npm run test:data --prefix octofit-tracker/backend` - 4 failed, 1 passed. Required-field validation is missing for user, team, activity, and workout models; seed idempotency passes.
- API command: `npm run test:api --prefix octofit-tracker/backend` - 4 failed. Activity/team invalid requests return `501` instead of `400`; leaderboard is empty despite activity records; Mona's first suggestion is strength-focused rather than endurance/run-focused.
- Frontend command: `npm test --prefix octofit-tracker/frontend` - 15 failed. The views do not render API data or request user-specific workout suggestions as expected, and all four views fail loading, empty-state, and accessible API-error state contracts.
- Backend build command: `npm run build --prefix octofit-tracker/backend` - not run; build status unknown.
- Frontend build command: `npm run build --prefix octofit-tracker/frontend` - not run; build status unknown.
