# OSM #MapCupAPAC2026 hourly quality check

This is a direct replica of the working `#tt_event` quality-check pipeline
(sourced directly from `KingVik-Planet/OSM_Quality_Check`, verified
byte-for-byte identical to the original `#tt_event` deployment at the
time this was built), changed in exactly two places:

1. **Hashtag**: watches for `#MapCupAPAC2026` instead of `#tt_event`
   (`config.py` → `HASHTAG`).
2. **Start date**: `data/state.json` is pre-seeded with
   `"last_run_end_utc": "2026-07-01T00:00:00"`, so the very first run
   begins its scan there instead of "one hour before now."

Nothing else about the pipeline, its checks, or its resilience layer has
been changed.

## Important: this will backfill slowly, on purpose

Every run processes **at most one hour** of data, and the schedule fires
**once per hour** (see `main.py`'s `determine_window()` for why this cap
exists — it's what stops a missed run from snowballing into an
ever-growing, unstable backlog). With the structure kept identical, a
backfill from **July 1, 2026** to today will take roughly as many
real-world hours as there are hours in that gap before it catches up to
live, real-time monitoring — on the order of weeks to months, not days.

This is a deliberate trade-off: the exact same design that keeps the live
pipeline fast, bounded, and crash-safe is what makes a long historical
backfill slow. If you ever want the backfill itself to go faster without
touching anything else, the one lever is the window size inside
`determine_window()` — everything downstream (retry queue, circuit
breaker, per-run retry cap) already scales with it automatically.

## Setup

1. Push this repository to GitHub.
2. Confirm `.github/workflows/hourly.yml` is at the repo root under
   `.github/workflows/` (not nested inside a subfolder).
3. In **Settings → Actions → General → Workflow permissions**, select
   **Read and write permissions** — required for the pipeline to commit
   its own findings back to the repo.
4. Optionally add repository secrets: `OSMCHA_TOKEN`, `SLACK_BOT_TOKEN`,
   `SLACK_CHANNEL_ID` (Slack posting stays off until `QC_SLACK_ENABLED`
   is set to `"1"` in the workflow file).
5. Trigger the workflow once manually (Actions tab → the workflow →
   **Run workflow**) to confirm it runs cleanly, rather than waiting for
   the next scheduled `:07` slot.

## What it checks

All 17 categories from the original pipeline, unchanged: overlapping/
crossing/contained buildings, duplicated nodes and ways (both within an
upload and against pre-existing map data), crossing and overlapping
highways, node-connects-highway-and-building (with a `covered=yes`
exception), way-end-node-near-other-way, dense node clusters, sudden
highway classification changes, broken highway continuity, floating
highways, missing primary tags, wrong tagging, mass upload/delete
without a revert signature, and unclear changeset comments.

## Output

Same rotating CSV structure under `data/`:
`s_no, error_type, username, user_id, osm_location_link, changeset_id,
changeset_link, osm_object_type, osm_object_id, time_utc, country, detail`

`quality_check_1.csv` rotates to `_2.csv` at 40MB, same as the original.
