# api-contracts

The versioned, machine-readable API contract shared by every client (`mobile-app`, `web/admin`) and by `backend-api`
itself (which generates it — see below). Nothing here is hand-maintained except `docs/client-guide.md`.

```
openapi/openapi.json                     generated from the FastAPI app; the source of truth for request/response shapes
errors/errors.yaml                       error code -> message, mirrors app/errors.py::ERROR_CODES
schemas/activity-content.schema.json     JSON Schema for an activity, format v1 (13 step types)
schemas/activity-content.v2.schema.json  format v2: exercises own their skills; editable (see docs/activity-format-v2.md)
test-vectors/activity-v2/                valid stored + wire examples and documents that must be rejected
test-vectors/mastery/*.json              the mastery-model and streak-rule worked examples; both backend-api's
                                          pytest suite and (recommended) any reimplementation are checked against these
docs/client-guide.md                     behaviour a schema can't express: auth/refresh rotation, child-mode tokens,
                                          PIN elevation, idempotency, the step-type -> UI mapping, error shape
```

## How this repo is kept in sync

`backend-api` is authoritative for the API. After changing an endpoint or a pydantic schema there:

```
cd backend-api
python scripts/export_contracts.py --out ../api-contracts     # regenerates openapi.json and errors.yaml
```

CI on `backend-api` re-generates the contract and fails the build if it differs from what's committed here, so a
merged backend PR and a stale `api-contracts` checkout cannot silently drift apart for long.

`content-curriculum` is authoritative for `schemas/activity-content.schema.json` and `schemas/activity-content.v2.schema.json`; `backend-api/app/content/` and
`content-curriculum/schemas/` both vendor a copy, refreshed with `backend-api/scripts/sync_contracts.py`. A pytest
in `backend-api` fails if its vendored copy differs from the sibling checkout of this repo.

## Consuming this repo

Point an OpenAPI code generator or client at `openapi/openapi.json` for types; read `docs/client-guide.md` for
everything the schema doesn't say (which is most of the interesting behaviour). `mobile-app` and `web/admin` do not
currently generate code from the schema (hand-written clients, kept honest by `mobile-app/tools/verify_api_paths.py`
and by integration tests) — wiring in a generator is a reasonable follow-up.
