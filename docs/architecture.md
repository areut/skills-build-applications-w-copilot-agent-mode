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

- Presentation tier decision: Keep route composition in `App.tsx` and each capability in its own component. Components use the shared API client with relative `/api` paths and expose accessible loading, error, empty, and populated states.
- Logic tier decision: Keep HTTP handling in Express routes and input validation, persistence orchestration, and response shaping in the service layer. Return deliberate 4xx responses for invalid input or missing records; do not leak Mongoose cast errors as 500 responses.
- Data tier decision: Define required fields and references in Mongoose schemas. Users require `name`, `email`, and `fitnessGoal`; teams require `name` and `mascot` and default `members` to an empty array; activities require user/team references, type, minutes, points, and `occurredAt`; workouts require title, focus, minutes, and difficulty. Seed representative linked records idempotently without deleting or overwriting unrelated records.
- Activity tracking approach: Validate required values, ObjectId syntax, and that the referenced user and team exist before writing. Store the user/team references, type, minutes, computed points at one point per minute, and occurrence time. Return a 4xx for invalid or unknown references and list persisted activities through the same API contract.
- Team management approach: Reject a missing name with `400`, preserve submitted name and mascot, and support member references with an empty array as the default. Create and list teams through the service layer.
- Leaderboard approach: Sum persisted activity points by team, sort highest total first, and return `teamName`, `totalPoints`, and a 1-based rank.
- Workout suggestions approach: Find the user by email; prefer workouts matching the user's fitness goal and use recent activity as the fallback signal. Include a short reason tied to the selected goal or activity; return a deliberate 4xx when no user matches.

## Test plan

Record the commands you run and what each failure means.

- Data-tier command: `npm run test:data --prefix octofit-tracker/backend` — checks schema validation and repeatable, non-destructive seeding. Baseline: 4 failed, 1 passed; failures indicate missing required model fields/validation, while the seed contract passes.
- API command: `npm run test:api --prefix octofit-tracker/backend` — checks activity and team create/list behavior, invalid-input responses, leaderboard totals/ranks, and user-specific workout suggestions. Baseline: 4 failed; current service stubs return `501`, an empty leaderboard, or suggestions without a matching reason.
- Frontend command: `npm test --prefix octofit-tracker/frontend` — checks API-backed rendering and accessible loading, empty, and error states for all four capabilities. Baseline: 15 failed; the screens are still implementation placeholders and do not satisfy these interactions.
- Backend build command: `npm run build --prefix octofit-tracker/backend` — checks TypeScript types and backend compilation; a failure indicates a type or compile issue. Not run in the recorded baseline.
- Frontend build command: `npm run build --prefix octofit-tracker/frontend` — checks frontend TypeScript and Vite production compilation; a failure indicates a type, module, or bundling issue. Not run in the recorded baseline.
