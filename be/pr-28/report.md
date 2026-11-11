# PR #28 — Swagger browser verification

PR: [rewear-be#28](https://github.com/buihaiduongdev/rewear-be/pull/28).

Tested on 2026-11-11 against the local Rewear NestJS server at
`http://localhost:5001/api/docs`, using **agent-browser** and a real JWT acquired
through the credentials login endpoint. Application revision:
[`f953047`](https://github.com/buihaiduongdev/rewear-be/commit/f953047).

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
- `pnpm build`: passed separately.
- `pnpm test --runInBand`: **18 suites, 128 tests passed**.
- Real database E2E plus HTTP contract suite: **6 suites, 108 tests passed**.
  The database suites covered product lifecycle, cart, checkout, pickup locations,
  and marketplace seeding. The HTTP contract suite uses mocked persistence;
  database suites use dedicated PostgreSQL databases and fixture identity guards.
- `docker compose config --quiet` and `git diff --check`: passed.
- Category regression was observed failing with the old `Object.assign` mock,
  then passing after the mock skipped undefined values. SSL schema tests were
  also observed failing before configuration support was added.
- Direct PostgreSQL assertions verified the browser results were persisted,
  entity IDs and ownership remained unchanged, and the rejected ID payload did
  not update the category.
- All synthetic browser fixtures, their refresh tokens, addresses, memberships,
  shop and pickup location were removed after verification. No fixture records
  remained in the checked tables.

## Local setup

Start the database from `rewear-be`:

```bash
docker compose up -d --wait db
```

Use `DB_SSL=false` with the local database. The committed `.env.example` shows
the local connection defaults. The running backend in this session used
`PORT=5001`; port 3001 belonged to another project and was not used for proofs.

Compose initializes `rewear` and `rewear_pickup_test` on a fresh volume.
For the full E2E run, dedicated local databases `rewear_product_test`,
`rewear_cart_test`, `rewear_checkout_test` and `rewear_seed_test` were also
created. The suites create and clean their own schemas. Run with:

```bash
PICKUP_TEST_DATABASE_URL=postgresql://postgres:postgres@127.0.0.1:5432/rewear_pickup_test \
PRODUCT_TEST_DATABASE_URL=postgresql://postgres:postgres@127.0.0.1:5432/rewear_product_test \
CART_TEST_DATABASE_URL=postgresql://postgres:postgres@127.0.0.1:5432/rewear_cart_test \
CHECKOUT_TEST_DATABASE_URL=postgresql://postgres:postgres@127.0.0.1:5432/rewear_checkout_test \
SEED_TEST_DATABASE_URL=postgresql://postgres:postgres@127.0.0.1:5432/rewear_seed_test \
  pnpm test:e2e --runInBand --testTimeout=30000
```

## Screenshot handling and scope

The cURL block containing the Authorization header was hidden with a temporary
CSS rule before capture; the static response schema table was also hidden for
readability. The request editor, Request URL, HTTP status, actual response body
and response headers were left unchanged. These are real Swagger screenshots,
with no constructed API response images.

Screenshots contain no passwords or session tokens. These results verify local
behavior, not a deployed environment or external shipping/storage services.
