---
name: renaissance-student-proficiency-lookup
description: >-
  Look up what Renaissance's Student Proficiency Service predicts for a student or a class — skill
  proficiency, the next recommended pathway activity, and a student's current reading level — using
  only operations that exist in the published contract.
generated: '2026-09-13'
method: generated
source: openapi/renaissance-student-proficiency-service-openapi.yml
api: Student Proficiency Service
base_url: https://proficiency.renaissance.com
operations:
  - validate_skill_ids_v1_skills_validate_post
  - predict_proficiency_v1_skill_post
  - predict_group_proficiency_v1_skill_groups_post
  - predict_group_proficiency_by_class_v1_skill_groups_by_class_post
  - get_next_activity_v1_pathway_next_activity__student_rgp_id__get
  - get_reading_level_endpoint_reading_level__student_id__get
  - get_class_students_v1_classes__class_id__students_get
---

# Reading Renaissance proficiency predictions

This service answers three questions about a K-12 student: how likely are they to get a skill right,
what should they practice next, and what is their current reading level. Every operation below is
read-only — nothing here changes student state.

## Before you start

- **Base URL is not in the contract.** The published document declares no `servers[]`. Send requests
  to `https://proficiency.renaissance.com`.
- **Every scored operation needs a bearer JWT.** Tokens come from Renaissance's authorization server
  at `https://auth.renaissance.com/oauth2/token` (client credentials). Only `/health` and `/launch`
  are open.
- **Student data here is FERPA-regulated.** `get_class_students_...` and the reading-level operation
  return student names. Do not cache, log or echo them outside the calling system.

## Steps

1. **Check the skills are supported.** `POST /v1/skills/validate`
   (`validate_skill_ids_v1_skills_validate_post`) with your `rl_skill_id` list. It answers with
   `supported_skills` and `unsupported_skills`. Do this first — the prediction operations do not tell
   you that a skill was silently unmodelled.
2. **Predict.** Pick the operation that matches your question:
   - one or more students against specific skills → `POST /v1/skill`
     (`predict_proficiency_v1_skill_post`), body `{ student_rgp_ids, rl_skill_ids }`
   - a skill group → `POST /v1/skill-groups` (`predict_group_proficiency_v1_skill_groups_post`)
   - a whole class → `POST /v1/skill-groups/by-class`
     (`predict_group_proficiency_by_class_v1_skill_groups_by_class_post`), body `{ class_id, rl_skill_ids }`
   Each prediction carries `predicted_pct_correct` (0–100), `prediction_model`, `prediction_ts`,
   `skill_is_practiced` and an `instructional_group` of `APPROACHING`, `INSTRUCTIONAL` or `ENRICHMENT`.
3. **Ask what to do next.** `GET /v1/pathway/next-activity/{student_rgp_id}`
   (`get_next_activity_v1_pathway_next_activity__student_rgp_id__get`) returns a title, a description
   and an `activity` with `appCode` (`APPS_FR`, `APPS_LALILO` or `QUIZ_ENGINE`) and a `deepLink`.
4. **Get the reading level if you need it.** `GET /reading-level/{student_id}`
   (`get_reading_level_endpoint_reading_level__student_id__get`) returns `zpdMin`, `zpdMax`, `lexile`
   and `asOf`. Note the path is unversioned while everything else is `/v1`, and the contract does not
   state that `student_id` is the same identifier space as `student_rgp_id`. Confirm the id before
   assuming.

## Errors you must handle

- **404 on next-activity is expected, not exceptional.** Branch on `detail.code`:
  `invalid_assessment_score`, `cold_start_disabled`, `invalid_grade` or `no_skills_available`. Never
  parse `detail.message`. Keep `detail.request_id` — it is the only correlation handle the service
  hands you.
- **422** is a FastAPI validation envelope: `detail[]` with `loc`, `msg`, `type`. Fix the named field;
  retrying unchanged will fail identically.
- **401/403 are undeclared.** The contract documents neither despite gating every operation, so treat
  any non-2xx that is not 404 or 422 as an auth or transport problem and re-check the token audience.

## What this API does not give you

No pagination, no filtering, no rate-limit headers, no 429 contract, and timestamps in
`YYYY-MM-DD HH:MM:SS` with no timezone stated. Do not compute an instant from a timestamp without
confirming the zone out of band.
