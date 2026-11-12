# PR #28 — Swagger browser verification

PR: [rewear-be#28](https://github.com/buihaiduongdev/rewear-be/pull/28).

Tested on 2026-11-11 against the local Rewear NestJS server at
`http://localhost:5001/api/docs`, using **agent-browser** and a real JWT acquired
through the credentials login endpoint. The screenshots document application
revision [`f953047`](https://github.com/buihaiduongdev/rewear-be/commit/f953047).
The subsequent CI/test setup revision is
[`330cb14`](https://github.com/buihaiduongdev/rewear-be/commit/330cb14); it preserves
the application behavior shown in the screenshots. The later test simplification
is [`060dec8`](https://github.com/buihaiduongdev/rewear-be/commit/060dec8).

Requests below were executed using Swagger's **Try it out → Execute** controls.
The backend connected to the Docker PostgreSQL 16 development database. A
synthetic active admin fixture was inserted locally for authentication; the
category, shop and address fixtures were then created through the real API.
No existing user records were edited.

| Case                                                                                                                             | Actual result                                                                       | Screenshot                                            |
| -------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------- | ----------------------------------------------------- |
| Category partial update: `name: null` is omitted by validation, while `isActive: false` and `requiredMeasurements: []` are saved | HTTP 200; original name/slug preserved, false and empty array persisted             | [View](screenshots/01-category-partial-update.png)    |
| Shop update: empty description, omitted status                                                                                   | HTTP 200; description becomes null and status stays ACTIVE                          | [View](screenshots/02-shop-empty-description.png)     |
| Address update: nullable label, omitted recipient, canonical administrative names and default flag                               | HTTP 200; label cleared, recipient retained, canonical names refreshed, default set | [View](screenshots/03-address-canonical-and-null.png) |
| Personal pickup location first save                                                                                              | HTTP 200; location created with canonical names and country VN                      | [View](screenshots/04-pickup-create.png)              |
| Personal pickup location second save                                                                                             | HTTP 200; same location ID updated, canonical names refreshed, country remains VN   | [View](screenshots/05-pickup-update.png)              |
| Category body attempts to set entity ID                                                                                          | HTTP 400; strict request schema rejects the extra field                             | [View](screenshots/06-reject-entity-id.png)           |

## Verification

- `pnpm check`: lint, formatting, TypeScript and Nest build passed.
- `pnpm test`: **25 suites, 242 tests passed, zero skipped**.
- [GitHub Actions run 43159360964](https://github.com/buihaiduongdev/rewear-be/actions/runs/43159360964): passed with the same **25 suites and 242 tests**, confirmed from the CI log.
- Focused unit run: **19 suites, 134 tests passed**.
- Focused integration run: **1 suite, 4 tests passed**. It directly tests
  marketplace seed behavior against PostgreSQL.
- Focused HTTP E2E run: **5 suites, 104 tests passed**. These suites exercise
  cart, checkout, pickup location, product lifecycle and request validation.
- The integration suite uses `rewear_test.public` and resets entity tables before
  running. HTTP suites are in `test/e2e`; folder READMEs summarize their scope.
- Missing DB configuration and an unavailable PostgreSQL server were each
  checked through a real Jest run: both failed, with zero skipped tests.
- `pnpm test:seed`: all four marketplace seed tests passed.
- `docker compose config --quiet` and `git diff --check`: passed.
- The temporary PostgreSQL test server was stopped after verification.
- Direct PostgreSQL assertions verified that the earlier browser results were
  persisted, IDs and ownership remained unchanged, and the rejected ID payload
  did not update the category. All synthetic browser fixtures were cleaned up.

## CI and local setup

CI now creates a disposable PostgreSQL 16 service with the database
`rewear_test`, provides the standard `DB_*` environment variables, and runs
unit, direct database integration and HTTP E2E tests through `pnpm test`, which
defaults to sequential execution. The previous
CI setup did not provide test database URLs: it ran 29 HTTP contract tests while
skipping 79 PostgreSQL tests. Those skip gates and per-suite URL variables have
been removed.

Development Compose now creates only `rewear`; there is no test database init
SQL. The integration suite uses `rewear_test.public`; the helper resets its
entity tables with TypeORM `synchronize(true)`. Because this reset is
destructive, only point tests at the disposable test database. This repository's
test setup does not use migrations.

For local tests, use a disposable server separate from the dev volume:

```bash
docker run --rm -d --name rewear-test-db \
  -e POSTGRES_USER=rewear_test -e POSTGRES_PASSWORD=rewear_test \
  -e POSTGRES_DB=rewear_test -p 127.0.0.1:55432:5432 \
  postgres:16-alpine

# Run after PostgreSQL is ready.
DB_HOST=127.0.0.1 DB_PORT=55432 DB_USERNAME=rewear_test \
DB_PASSWORD=rewear_test DB_NAME=rewear_test DB_SSL=false \
  pnpm test

docker stop rewear-test-db
```

Tests read explicit `DB_*` variables, require the local `rewear_test` database,
and do not fall back to `.env` or `DATABASE_URL`. Unit tests can run without
PostgreSQL using `pnpm test:unit`.

The Swagger browser session used `http://localhost:5001/api/docs`; port 3001
belonged to another project and was not used for Rewear proofs.

## Screenshot handling and scope

The cURL block containing the Authorization header was hidden with a temporary
CSS rule before capture; the static response schema table was also hidden for
readability. The request editor, Request URL, HTTP status, actual response body
and response headers were left unchanged. These are real Swagger screenshots,
with no constructed API response images.

Screenshots contain no passwords or session tokens. These results verify local
behavior, not a deployed environment or external shipping/storage services.
