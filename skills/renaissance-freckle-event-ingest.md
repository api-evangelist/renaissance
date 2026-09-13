---
name: renaissance-freckle-event-ingest
description: >-
  Post a Freckle practice event into Renaissance's Student Pathway Event Proxy correctly and safely —
  the one write operation in Renaissance's published contract set, with no idempotency key and no way
  to take an event back.
generated: '2026-09-13'
method: generated
source: openapi/renaissance-student-pathway-event-proxy-openapi.yml
api: Student Pathway Event Proxy
base_url: https://events.proficiency.renaissance.com
operations:
  - post_event_freckle_events_post
  - health_health_get
---

# Ingesting a Freckle practice event

`POST /freckle-events` (`post_event_freckle_events_post`) accepts one practice event into the student
pathway pipeline. It answers **202 Accepted** — acknowledgement, not confirmation of processing.

## Read this before you write any retry logic

**There is no idempotency key and no reversal.** The contract declares no `Idempotency-Key` header,
returns no identifier in the 202 body, and publishes no delete, void or retraction operation. A blind
retry after a timeout can double-count a student's answer, and nothing in the published surface lets
you undo it. If you must retry, deduplicate on your own side using `answerToken` plus `occurredAt`
before re-sending, and record what you sent.

## The event body

`FreckleEvent` requires the shape the contract declares; the fields are:

- identity: `studentId`, `studentRgpId`, `answerToken`, `questionId`
- skill: `rlSkillId`, `skillRslId`
- context: `product` (one of the `FreckleProduct` values — `math_adaptive`, `math_targeted`,
  `focus_skills_practice`, `math_assessments`, `math_ren_intel_pathway`,
  `ela_adaptive_skills_practice`, `ela_targeted_skills_practice`, `ela_focus_skills_practice`,
  `ela_word_study`, `ela_articles_reading`, `science_articles_reading`,
  `social_studies_articles_reading`, `ela_ren_intel_pathway`)
- outcome: `correctness`, `occurredAt`, `version`
- `metadata` (`FreckleEventMetadata`): `sessionId`, `durationSeconds`, `attempts`,
  `answerPositionInSession`, `assignmentType`, `context`, `standardSetId`, `skillProgressionOrder`,
  `skillProgressionSuperOrder`, `startedFGP`, `automaticHintShown`, `clickedOnHint`, `usedHint`,
  `usedVideo`, `usedKeyword`, and `answer` (an `AnswerValue` of `tag` + `value`).

## Steps

1. `GET /health` (`health_health_get`) — open, no token. Use it to confirm reachability before a batch.
2. Acquire a bearer JWT from `https://auth.renaissance.com/oauth2/token`. The base URL is not in the
   contract; send to `https://events.proficiency.renaissance.com`.
3. `POST /freckle-events` with the event body and the bearer token.
4. On **202**, record your own dedupe key and stop. On **422**, read `detail[]`, fix the named field
   and re-send — a validation failure means nothing was ingested. On any other non-2xx, do **not**
   retry automatically; reconcile first.

## Privacy

Freckle events are individual K-12 student learning records. Handle them as FERPA-covered student
data: no logging of raw bodies to shared sinks, no retention beyond what your integration needs.
