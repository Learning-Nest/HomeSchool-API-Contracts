# Activity format v2

Status: implemented in `backend-api` (release 1 of the activities redesign). The mobile app is **not** changed by it.

An activity is one JSON document made of **exercises** (called `steps` on the wire and in the database). In v2 every
exercise owns its skills, so one activity can cover several skills and each answer only counts for the skills it
actually tests. Documents are edited by people, so the format keeps the *child-visible content*, the *answer key* and
the *scoring rules* in separate blocks.

* Schema: `schemas/activity-content.v2.schema.json` (Draft 2020-12). Version 1 documents (no `schema_version`) stay valid.
* Examples: `test-vectors/activity-v2/` (`*.stored.json` is what is saved, `*.wire.json` is what the app receives,
  `invalid-cases.json` lists documents that must be rejected and the message they must produce).

## Document

| Field | Meaning |
|---|---|
| `schema_version` | Always `2`. Absent = version 1. |
| `slug`, `title`, `summary`, `subject`, `level_from`, `level_to`, `duration_min`, `materials`, `interests`, `recurring` | As in v1. The slug cannot be changed after the activity exists. |
| `locale`, `tags` | Optional. `locale` like `en-IN`. |
| `authoring` | Editor-only notes (`notes`, `educator_reviewed`, `reviewed_by`). **Never sent to the app.** |
| `skills` | **Derived.** The server rewrites it on every save as the skills of the enabled exercises that produce evidence. A client may send anything (or a stale copy) there; the stored value is always the derived one. |
| `steps` | 1 to 40 exercises. |

## Exercise

```jsonc
{
  "id": "q2",                         // stable, [a-z0-9_-]{1,32}; answers and evidence are keyed by it, never reuse
  "type": "numeric_input",            // one of the 13 types (below)
  "enabled": true,                    // false = hidden from children and from scoring, but kept for editors
  "title": "Add within 10",           // label for editors; not shown to the child
  "prompt": "3 + 4 = ?",
  "audio_ref": null, "media_ref": null,
  "config": { },                      // what the child sees / layout. Sent to the app.
  "key":    { "answer": 7, "tolerance": 0 },   // the answer key. NEVER sent to the app.
  "scoring": { "mode": "auto", "partial_credit": false },
  "skills": [ { "code": "MAT.NUM.ADD10", "weight": 1 } ],
  "feedback": { "hints": ["Start from 3."], "correct": "...", "incorrect": "..." }
}
```

Per type, where the fields go:

| type | `config` | `key` |
|---|---|---|
| instruction | none | none |
| media_prompt | `caption`, `alt_text` (required) | none |
| single_choice | `options` (2..8, `{id,label,media_ref}`) | `correct` (one option id) |
| multi_choice | `options` | `correct` (option ids); `scoring.partial_credit` |
| numeric_input | none | `answer`, `tolerance` |
| short_text | `max_len` | `accepted` (case-insensitive; empty = not scored) |
| sequence_order | `items` | `correct_order` (every item id once) |
| match_pairs | `left`, `right` | `pairs` (`[leftId, rightId]`) |
| audio_record | `max_seconds`, `consent_required` | none |
| photo_evidence | `optional`, `consent_required` | none |
| timer_task | `duration_sec`, `checklist` | none |
| parent_checklist | `rating_options` | none |
| reflection | `emoji_options` | none |

### Scoring and skills

* `scoring.mode`: `auto` (the server scores the answer; the default for the six auto types), `parent_rubric` (a parent
  rates it in the review; the default for `parent_checklist`), `none` (never gives evidence; the default for the rest).
* An exercise gives evidence for each skill in its own `skills` and for no other. The evidence weight is
  `base weight x independence factor x skill weight`, where the skill weight is `0 < weight <= 1` (default 1).
  A scored or rated exercise must list at least one skill.
* A disabled exercise credits nothing and is not sent to the app. Answers for it are rejected.
* Reserved, stored but not used yet: `scoring.points`, `feedback.correct`, `feedback.incorrect`, and hints after the first.

## Rules the server enforces on save (admin PUT, bundle import, publish)

JSON Schema first, then: unique exercise ids; unique option/item ids; every `key` reference points at an existing
option/item; scored exercises have a key and at least one known skill; no skill listed twice in one exercise;
`auto` only on the six auto types and `parent_rubric` only on `parent_checklist`; at least one enabled exercise and at
least one that gives evidence; top-level `skills` equals the derived list (the server fixes it before validating).
Failures return `422 content_invalid` with a `problems` list of readable lines.

## What the app receives (no app change in release 1)

The server projects a v2 document back to the v1 wire shape: disabled exercises are dropped; `config` is flattened onto
the exercise; an instruction's `prompt` is sent as `text`; `feedback.hints[0]` is sent as `hint`; a parent checklist
gets `skill_code` (its first skill); `partial_credit` is passed through. `key`, `scoring`, per-exercise `skills`,
`feedback`, `title`, `authoring` and `schema_version` are not sent. Session results gain `result.steps[id].skills`
(the app ignores it).

## Upgrading v1 documents

There is no database migration. A v1 document is upgraded in memory on every read by `upgrade_step` (idempotent): `text`
becomes `prompt`, option and layout fields go into `config`, answer fields go into `key`, `hint` becomes
`feedback.hints[0]`, `scored: false` becomes `scoring.mode: "none"`, a parent checklist's `skill_code` becomes its
skill, and an auto-scored exercise with no skills of its own gets all of the activity's skills (what v1 did at run time).
`content-curriculum/tools/migrate_v2.py` performs the same rewrite on the repo files.

## Versions and the source of truth

The database is the source of truth once an editor exists; `content-curriculum` is seed content. Editing a published
activity (`PUT /v1/admin/activities/{id}/definition`) creates a new version; sessions keep the version they started
with, so changing a key never re-scores a past session.
