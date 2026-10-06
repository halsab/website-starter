# Website Starter Foundation Specification

This document is the complete source of truth for initializing a neutral website project in an empty repository. A coding agent needs only this file and one user-provided value, `PROJECT_NAME`. The starter repository contains specifications and instructions only; generated application files belong in the target repository.

## Input and deterministic names

The only required input is `PROJECT_NAME`, a non-empty human-readable name. Normalize it once to a project slug as follows:

1. Trim leading and trailing whitespace.
2. Convert to lowercase ASCII. Transliterate accented Latin characters to their base letters; discard other characters that are not ASCII letters or digits.
3. Replace each run of characters other than `a-z` and `0-9` with one hyphen, then trim hyphens from both ends.
4. If the result is empty, stop and request a usable `PROJECT_NAME`; do not invent another name.

Derive every technical identifier from that slug:

| Identifier | Rule | Example for `North Star` |
| --- | --- | --- |
| Project slug | normalized slug | `north-star` |
| Root package name | project slug | `north-star` |
| npm scope | `@` + project slug | `@north-star` |
| Backend package | npm scope + `/backend` | `@north-star/backend` |
| Website package | npm scope + `/website` | `@north-star/website` |
| Contracts package | npm scope + `/contracts` | `@north-star/contracts` |
| Development database | replace slug hyphens with underscores | `north_star` |
| Test database | development database + `_test` | `north_star_test` |
| User-facing project title | split slug on hyphens and capitalize the first character of each segment, joined with spaces | `North Star` |

If normalization would produce a package name or database identifier rejected by the chosen tools, stop and explain the constraint; do not silently substitute an unrelated identifier. Use the same derived values consistently in package manifests, environment examples, Compose configuration, documentation, and website copy.

## Repository shape

Create this minimum structure (generated files shown; no speculative directories):

```text
.
├── .bun-version
├── .gitignore
├── AGENTS.md
├── README.md
├── bun.lock
├── docker-compose.yml
├── package.json
├── backend/
│   ├── .env.example
│   ├── package.json
│   ├── prisma/
│   │   ├── migrations/
│   │   │   └── <timestamp>_init/migration.sql
│   │   └── schema.prisma
│   ├── scripts/prepareIntegration.ts
│   ├── src/
│   │   ├── app.ts
│   │   ├── db.ts
│   │   ├── env.ts
│   │   └── index.ts
│   ├── test/
│   │   ├── integration/
│   │   ├── unit/
│   │   ├── testDatabase.ts
│   │   └── testDatabaseUrl.ts
│   └── tsconfig.json
├── docs/
│   ├── ARCHITECTURE.md
│   └── TESTING.md
├── packages/
│   └── contracts/
│       ├── package.json
│       ├── src/index.ts
│       └── tsconfig.json
└── website/
    ├── .env.example
    ├── astro.config.mjs
    ├── package.json
    ├── src/
    │   ├── locales/
    │   │   ├── en.ts
    │   │   └── messages.ts
    │   └── pages/index.astro
    └── tsconfig.json
```

The first Prisma migration may be empty apart from Prisma's migration bookkeeping: the baseline starts without product models. Include only the files needed for this foundation. Do not add generated product features.

## Runtime and workspace baseline

- Use Bun `1.3.14`; record it in `.bun-version` and pin the package manager as `"packageManager": "bun@1.3.14"` in the root manifest. Commit the Bun lockfile and use Bun for install, scripts, and tests.
- Use a private root package named by the slug and Bun workspaces `backend`, `website`, and `packages/*`.
- Use strict TypeScript with no emitted JavaScript for backend and contracts. Astro uses its strict TypeScript configuration.
- Keep `.gitignore` focused on dependencies, generated output, local environment files, logs, and OS/editor detritus. Ignore `.env` and `.env.*` while allowing checked-in `.env.example` files.
- Do not add a license unless requested.

### Package responsibilities and dependencies

- **Root:** orchestration scripts only; no application runtime dependencies.
- **Backend (`@<slug>/backend`):** Bun server, Hono HTTP app, Zod environment validation, Prisma Client persistence. Runtime dependencies: `hono`, `zod`, `@prisma/client`. Development dependencies: `prisma`, `typescript`, `@types/bun`.
- **Contracts (`@<slug>/contracts`):** shared TypeScript API contracts and baseline locale identifier types. It exports source through its package entry point and is consumed through a Bun workspace dependency. Development dependencies: `typescript`, `@types/bun`.
- **Website (`@<slug>/website`):** static/localizable Astro website. Runtime dependencies: `astro` and the contracts workspace. Development dependencies: `@astrojs/check`, `typescript`.

Install current compatible releases at initialization and record exact resolutions in `bun.lock`; do not copy dependency versions from an unrelated project except for the pinned Bun version. Add a dependency only if one of the specified roles requires it.

### Required script classes

Implement scripts with these names and behaviors:

- Root `dev`: run backend and website development servers in parallel; `dev:backend` and `dev:website` delegate to each package.
- Root `build`: build the Astro website.
- Root `typecheck`: typecheck contracts, backend, then website.
- Root `test:unit`: run backend unit tests and contracts tests; `test:integration`: run backend integration tests; root `test` runs unit then integration suites.
- Root `check`: validate the Prisma schema, generate Prisma Client, then run root typecheck, all tests, and website build, stopping at the first failure.
- Backend: `dev`, `start`, `typecheck`, `test:unit`, `test:integration:prepare`, `test:integration`, `test`, `prisma:generate`, `prisma:validate`, `prisma:migrate`, and `prisma:deploy`.
- Contracts: `typecheck` and `test`.
- Website: `dev`, `build`, and `typecheck` (Astro check).

Document and maintain the canonical full verification command as `bun run check`. Narrow commands may be used during development; the full check must pass before the generated project's initial commit.

## Backend behavior

- Create an injectable Hono app so route behavior can be tested without binding a network port. Keep startup and Prisma client construction outside the app module.
- `GET /health` is a liveness endpoint. It returns HTTP 200 with `{ "status": "ok" }` and does not query the database.
- `GET /health/ready` calls a database readiness function that executes a trivial query. Return HTTP 200 with `{ "status": "ready" }` on success. On database failure return HTTP 503 with `{ "error": { "code": "DATABASE_UNAVAILABLE", "message": "Service is not ready" } }`.
- Add a generic error handler that logs the internal error and returns HTTP 500 with `{ "error": { "code": "INTERNAL_ERROR", "message": "Internal server error" } }`; do not expose exception details.
- Parse environment configuration before starting the server. `DATABASE_URL` is required and must be a PostgreSQL URL. `PORT` is an integer from 1 through 65535 and defaults to 3000.
- Start Bun's HTTP server on `PORT`; close the server and disconnect Prisma on SIGINT/SIGTERM.
- No other endpoint or business behavior belongs in the baseline.

## Environment files

`backend/.env.example` contains:

```dotenv
DATABASE_URL=postgresql://<db-user>:<db-password>@localhost:5432/<development_database>?schema=public
TEST_DATABASE_URL=postgresql://<db-user>:<db-password>@localhost:5432/<test_database>?schema=public
PORT=3000
```

Use the project slug as the local PostgreSQL user and password unless a tool constraint requires otherwise. `website/.env.example` should state that no environment variables are currently required. Never commit actual `.env` files or secrets.

## Prisma and PostgreSQL

- Use Prisma with PostgreSQL. Start `schema.prisma` with only the Prisma Client generator and PostgreSQL datasource using `DATABASE_URL`; add no product models.
- Provide `prisma:generate`, `prisma:validate`, `prisma:migrate` (`prisma migrate dev`), and `prisma:deploy` (`prisma migrate deploy`). Check in migrations created for intentional schema changes.
- Use Docker Compose with `postgres:17-alpine`, a named persistent volume, health check, and environment values derived from the project identifiers. Bind the host port only to `127.0.0.1`; allow port override with `POSTGRES_PORT` defaulting to `5432`.
- The local PostgreSQL instance contains separate development and test databases. Explain that the test database must be created once for a new volume and how to check before creating it, so setup does not fail on an existing database.
- `DATABASE_URL` is only for development/runtime. Integration tests must use a Prisma client constructed with `TEST_DATABASE_URL`.
- Before integration tests, run `prisma migrate deploy` with `DATABASE_URL` overridden to the validated test URL. Migration failure must fail the preparation step and prevent tests from running.

### Fail-closed test database rules

The integration-test preparation and test Prisma client must share validation that:

1. Requires `TEST_DATABASE_URL` and rejects missing or malformed values.
2. Requires a `postgres:` or `postgresql:` protocol.
3. Requires a database path whose decoded name ends in `_test`.
4. Rejects a URL that targets the same host, effective port (default 5432), and database name as `DATABASE_URL`.
5. Returns/uses the validated URL as the test connection and never silently falls back to `DATABASE_URL`.

The integration preparation script must preserve the rest of the process environment, override only the migration subprocess's `DATABASE_URL` with the test URL, forward its output, and propagate a nonzero exit code. Unit tests must exercise each rejection and the distinct-database success case.

## Testing policy

- Use Bun's native test runner; separate backend tests under `test/unit` and `test/integration`. Contracts tests live with the package source or under its test directory.
- Unit tests cover pure rules, environment parsing, and app behavior without requiring PostgreSQL. Integration tests exercise HTTP/database behavior against real local PostgreSQL through `TEST_DATABASE_URL`.
- Prefer testing real internal behavior. Use narrow fakes only where a failure is otherwise unreliable; fake external services if any are introduced later.
- Use pragmatic TDD when behavior can be specified: write the narrowest failing test, verify the expected failure, implement the smallest correct change, then rerun the focused test.
- Test behavior rather than implementation details. Do not target coverage percentages or add end-to-end infrastructure by default.
- Never run integration setup, migrations, or tests against the development database. Missing or unsafe test configuration must stop execution.

## Shared contracts ownership

Any browser/backend request or response contract used on both sides belongs first in `packages/contracts`, including runtime validation schemas and shared TypeScript types. The backend owns HTTP transport and validation against those contracts; the website consumes the same contracts. Keep contracts package typechecking in the root verification workflow. Do not add API contracts until a feature needs them; locale identifiers are the only baseline shared type.

## Localization and website baseline

- Build the website with Astro and keep it product-neutral and minimal.
- Put English baseline messages in `website/src/locales/en.ts` and define the message shape in `messages.ts`.
- Define the locale identifier and default locale in `packages/contracts`; the default locale is `en`.
- Use the derived project title in the document title and simple placeholder heading/copy. Keep copy in the locale dictionary rather than embedding it in the page.
- Structure locale dictionaries so another locale can be added without changing page components. Do not add a translation framework unless a real requirement calls for it.

## Documentation

Create concise English documentation:

- `README.md`: project purpose, stack, prerequisites, install/setup/run steps, API and website local addresses, and links to testing/architecture docs.
- `docs/ARCHITECTURE.md`: describe the small initial boundaries and state that architecture/infrastructure should grow only for real requirements.
- `docs/TESTING.md`: pragmatic TDD, unit/integration distinction, local database setup, safe test database creation and configuration, migration flow, and canonical commands.
- `AGENTS.md`: repository-specific contributor guidance, contracts-first ownership, test database isolation, documentation expectations, and Git safety.

Keep docs aligned with implemented behavior. Avoid product requirements, copied historical references, and process documentation beyond what is needed to build and maintain this foundation.

## Git and initialization safety

- Initialize the repository only if it is not already a Git repository. Inspect existing files and status first; preserve user work and never overwrite unrelated content.
- Add only this foundation to an empty/new repository. Do not generate application-specific features or replace pre-existing files without a clear need.
- Run `bun run check` successfully before creating the initial commit. Keep repository content and commit messages in English.
- Do not use destructive Git operations, rewrite history, push, deploy, or create cloud resources unless explicitly requested.

## Explicit exclusions

The baseline is deliberately limited to Bun workspaces, TypeScript, Hono, Prisma/PostgreSQL, Astro, shared contracts, localization readiness, health/readiness endpoints, isolated local databases, tests, and documentation. Do not add authentication, accounts, admin UI, React, cookies, analytics, email, payments, storage, Redis, queues, background jobs, OpenAPI, Storybook, CI workflows, cloud deployment, Terraform, or other infrastructure speculatively. Add a system only when a concrete project requirement justifies it. The generated project may evolve independently after initialization.
