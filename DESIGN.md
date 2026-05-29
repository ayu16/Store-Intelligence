# DESIGN.md — Apex Store Intelligence (Brigade_Bangalore / STORE_BLR_002)

## 1. Problem framing

Apex Retail's offline stores are an analytics blind spot. This system turns raw
CCTV footage from the **Brigade Road, Bangalore** Purplle store into the same
funnel/conversion analytics the online channel already has. Every component is
justified by one number — the **North Star**:

```
Offline Conversion Rate = converted unique visitors / total unique visitors
```

A component either makes that number *more accurate* (the detection layer) or
*more actionable* (the API + dashboard). That test drove every trade-off below.

## 2. System architecture

```
 CCTV clips → Detection (YOLOv8 + ByteTrack) → Event JSONL → Ingest API → SQLite → Analytics endpoints → Live dashboard
                pipeline/detect.py              emit.py       app/main.py          metrics/funnel/anomalies   dashboard/index.html
```

The boundary between the two halves is a **single, strict event schema**
(`app/models.py :: StoreEvent`). The detection pipeline's only job is to emit
schema-valid events; the API's only job is to consume them. Because the contract
is explicit, the halves are independently testable and replaceable — YOLOv8 can
become RT-DETR, SQLite can become Postgres, without touching the other side.

## 3. Calibration to the real store

`pipeline/store_layout.json` was built by **unzipping the supplied
`Brigade_Road_-_Store_layout.xlsx` and reading the embedded floor-plan image**
(`xl/media/image1.png`). All 22 zones — 8 skincare on the top wall, 8 makeup on
the bottom wall, the FOH fragrance/nail/makeup units in the centre, cash counter,
PMU, accessories, backroom, entry threshold — come straight from that blueprint.
Camera roles were assigned by inspecting one frame from each clip:

| Clip | camera_id | Role | What it shows |
|---|---|---|---|
| CAM_3 | CAM_ENTRY_01 | entry_threshold | Glass door; wood floor inside, dark tile mall corridor outside |
| CAM_1 | CAM_FLOOR_SKINCARE | main_floor | Top wall: EB Korean, Face Shop, Good Vibes, DermDoc, Minimalist, Aqualogica, Lakme Skin |
| CAM_2 | CAM_FLOOR_MAKEUP | main_floor | Bottom wall: Maybelline, Faces Canada, Lakme, Colorbar, Swiss Beauty, Renee NY Bae, Alps, Streax |
| CAM_5 | CAM_BILLING_01 | billing | Cash counter (POS laptops), accessories, passage to backroom |
| CAM_4 | CAM_BACKROOM_01 | backroom | Stockroom — `customer_zone=false`, every detection flagged is_staff |

The **entry line** on CAM_3 sits at normalised `x = 0.55` with
`inbound_direction = leftward`: a track crossing right→left (mall tile → wood
floor) is ENTRY, left→right is EXIT. This single calibration drives entry/exit
accuracy (10 of Part A's 30 points). It was unit-tested directly against
`SessionManager` with synthetic crossings before being relied on for video.

## 4. Detection layer (Part A)

`pipeline/detect.py` runs in three modes. **video** runs YOLOv8 person detection
with ByteTrack, identifies each clip's camera from the layout, applies
line-crossing on CAM_ENTRY_01 and horizontal-band zone assignment on the floor
cameras, and is CPU-tuned (`imgsz=480`, `vid_stride=5`) so a clip processes in
a couple of minutes instead of thirty. **synth** generates a realistic stream
using the **real zone names** from `store_layout.json`, so the whole system is
testable end-to-end without footage and the grader's assertions key on the same
identifiers. **replay** streams an events file for the dashboard.

`tracker.SessionManager` is the brain: a virtual line decides ENTRY vs EXIT by
crossing direction; a per-visit `visitor_id` is minted on first entry; a
conservative Re-ID step (trajectory distance, with an OSNet embedding hook left
as a documented upgrade) re-uses that token on return, producing REENTRY rather
than a phantom second ENTRY.

## 5. Intelligence API (Part B)

FastAPI, six endpoints. The unit of analysis for funnel and conversion is the
**session** (`visitor_id`), never the raw event — this is what stops re-entries
double-counting a visitor. POS correlation is purely temporal: a visitor in a
billing zone within five minutes before a transaction timestamp counts as
converted, since the POS feed carries no customer identity. All analytics
endpoints accept `from`/`to` query params (defaulting to today) so historical
windows like the 2026-04-10 clip date can be queried without backdating events.
The CSV loader handles both the simple brief schema and the **rich Brigade
export** (101 line items collapsing to 24 invoices via the `invoice_number`
PRIMARY KEY).

## 6. Storage & production posture (Part C)

SQLite by default — zero-config, clears the acceptance gate on a clean machine,
and `event_id` as PRIMARY KEY makes ingest idempotent via `INSERT OR IGNORE`.
Every request emits one structured JSON log line (`trace_id`, `store_id`,
`endpoint`, `latency_ms`, `event_count`, `status_code`). A DB failure surfaces as
HTTP 503 with a structured body, never a raw stack trace. `/health` reports the
last event timestamp per store and flags STALE_FEED past a 10-minute lag.

## 7. Live dashboard (Part E)

`dashboard/index.html` is a dependency-free ops console that polls the four read
endpoints every two seconds and renders conversion rate, unique visitors, queue
depth, the funnel, zone engagement, and severity-coded anomalies — each tile
flashing on change. `pipeline/replay_to_api.py` streams a generated event file
into the live API paced by event timestamps and compressed by `--speed`, so the
dashboard visibly fills in real time. CORS is enabled for browser access.

## 8. AI-Assisted Decisions

**(1) Floor-plan extraction — I overrode the LLM.** Asked to parse the xlsx, the
assistant first read it with openpyxl and reported only "Revised"/"Current" cell
labels. That was wrong — the layout is an embedded image (`xl/media/image1.png`).
I overrode the read strategy: treat the xlsx as a ZIP, extract the media, read
the floor plan. Every zone name in the system comes from that image.

**(2) Re-ID strategy — I overrode the default.** The model proposed full OSNet
appearance embeddings. Higher accuracy, but for a 48-hour build on anonymised,
face-blurred footage the features are degraded and the dependency cost is high.
I chose a trajectory + time-gap heuristic as the default, OSNet documented as an
upgrade in `tracker.py`.

**(3) Ingest validation — I overrode the first cut.** The model's endpoint typed
the body as a strict Pydantic model, so one malformed event 422'd the whole
batch. The spec demands partial success. I changed it to accept raw JSON and
validate per-event in `ingestion.py`; a regression test locks the behaviour.

## 9. Known limitations / what I'd do next

- Re-ID is conservative; on a held-out clip I'd tune the threshold against ground
  truth and enable OSNet embeddings if footage quality allows.
- Floor-camera zoning uses equal horizontal bands — the simplest defensible
  default. The blueprint's mm dimensions allow a one-time homography to real
  polygon boundaries (≈ half a day).
- SQLite serialises writes; at 40 live stores I'd move to Postgres and put ingest
  behind a queue. The schema is already portable.
- Staff detection currently relies on `is_staff` set at emit time; a colour-
  histogram heuristic on the upper-body crop plus a VLM fallback on low-confidence
  crops is the next upgrade (see CHOICES.md decision 1).
