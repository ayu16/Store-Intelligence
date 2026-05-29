# CHOICES.md — Three decisions with full reasoning

Each decision lists the options I weighed, what AI suggested, and what I chose
and why. These are written so I can defend each one in the follow-up interview.

---

## Decision 1 — Detection model

**Options considered**
- **YOLOv8** (ultralytics): mature, fast, trivial ByteTrack integration, huge
  community for debugging edge cases.
- **YOLOv9 / RT-DETR**: marginally higher accuracy, heavier, less battle-tested
  tooling for tracking hand-off.
- **MediaPipe**: light, but weaker on the crowded/occluded retail cases.

**What AI suggested**
The assistant recommended YOLOv8n as the starting point and proposed using a VLM
(e.g. Claude Vision / GPT-4V) for staff-vs-customer classification by prompting
on cropped detections — "is this person wearing a store uniform?".

**What I chose and why**
YOLOv8 for person detection + ByteTrack for tracking, because the scoring rewards
*how I handle uncertainty*, not raw model novelty, and YOLOv8's tooling lets me
iterate on the seven edge cases fastest. On the VLM suggestion: I evaluated it and
chose a **hybrid** — rule-based staff detection first (uniform colour histogram +
zone-access patterns: staff move through back/billing zones in ways customers
don't), with a VLM as a fallback for ambiguous crops only. A VLM call per
detection is too slow and costly for 1 hour × 5 stores × 15fps, but a VLM on the
handful of low-confidence crops is affordable and improves the `is_staff` flag.
The prompt I tested: *"This is a cropped CCTV frame from a retail store. Is the
person a store employee (uniform/lanyard/behind-counter) or a customer? Answer
employee/customer/unsure with one reason."* It worked well on clear uniforms,
returned "unsure" honestly on blurred crops — which is exactly the calibration
behaviour the challenge rewards, so I keep those as low-confidence rather than
forcing a guess.

---

## Decision 2 — Event schema design

**Options considered**
- **Thin schema** (type, timestamp, visitor, store) with everything else inferred
  later from raw detections.
- **Rich self-describing schema** carrying zone, dwell, confidence, queue depth,
  staff flag, and a metadata block — the events are the analytics substrate.
- **Per-event-type schemas** (separate shapes for ENTRY vs ZONE_DWELL).

**What AI suggested**
The model leaned toward per-event-type schemas for type-safety.

**What I chose and why**
One **rich, uniform schema** for every event type (`StoreEvent` in
`app/models.py`), with type-specific fields nullable (e.g. `zone_id` is null for
ENTRY/EXIT, `queue_depth` populated only for billing events). I overrode the
per-type suggestion because a single shape makes ingest, dedup, and storage one
code path instead of eight, and the API can compute every metric with plain SQL
over one table. Critically, **`confidence` is a first-class field and low-
confidence events are never dropped at emission** — the spec explicitly penalises
silent suppression, so calibration is the consumer's decision, not the
pipeline's. `event_id` is a UUID-v4 and the storage PRIMARY KEY, which is what
makes the whole ingest path idempotent for free.

---

## Decision 3 — API storage / idempotency architecture

**Options considered**
- **In-memory store**: fastest to build, but loses idempotency across restarts
  and fails the "safe to call twice" requirement under a real deploy.
- **SQLite**: zero-config, file-backed, real SQL, portable schema.
- **PostgreSQL from day one**: production-grade concurrency, heavier setup,
  risks the `docker compose up` acceptance gate on a clean machine.

**What AI suggested**
The assistant suggested PostgreSQL for "production realism".

**What I chose and why**
**SQLite by default, Postgres-ready by DSN.** I overrode the Postgres-first
suggestion because the acceptance gate is binary — if `docker compose up` has any
friction on a clean machine, the submission scores zero regardless of quality.
SQLite removes that risk entirely while still giving me real SQL for the
aggregation queries and durable idempotency via the `event_id` PRIMARY KEY +
`INSERT OR IGNORE`. The schema uses no SQLite-specific types, so flipping
`DATABASE_URL` to Postgres is the only change needed when write concurrency at 40
live stores actually demands it. This is the honest trade: I optimised for *the
graded path runs flawlessly* now, with a clearly documented scale path later
(see DESIGN.md §4).
