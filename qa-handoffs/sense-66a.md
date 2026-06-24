# QA Handoff — sense-66a: /curate snippet curation LiveView

**Build:** f82d0ae on `claude/sense-66a/sense-66a.1-curate-liveview`
**Date:** 2026-06-25

---

## What shipped

First Sense LiveView at `/curate` (behind Plug.BasicAuth). A curator:

- Lists ~240 `code.snippet` nodes from the ash_knowledge_graph Sense graph — each card shows the code, language badge, provenance (`creation_method` / `source` / `license`), current tags/edges (joined from `projection.relations`), `curator_note`, and the active review-branch label.
- Edits the `curator_note` inline (free-text, phx-change/phx-submit form).
- Tags a snippet to a construct/pattern node (`code.snippet_of` edge, deterministic edge ID).
- Sets a review status: `approved` | `weak` | `regenerate`.

Every write lands on the configured Dolt review branch (default `curate-review`) via the AKG MCP HTTP seam — never on `main`. The sense→graph boundary is HTTP/MCP only; no library path-dependency.

The client-side LiveSocket is bundled from vendored UMD files (`mix assets.bundle`, no Node/esbuild step) so all `phx-` bindings are live in the browser.

---

## How to test (manual)

### Prerequisites

- `SENSE_CURATOR_USERNAME` / `SENSE_CURATOR_PASSWORD` set (or use dev defaults `curator` / `sense-dev`).
- `ASH_KG_MCP_URL` pointing to a running ash_knowledge_graph MCP server (e.g. `http://localhost:4283/`).
- Review branch pre-provisioned: `dolt checkout -b curate-review` in the Dolt repo before first write.

### Auth / routing

- `GET /curate` without credentials → 401 Unauthorized (BasicAuth challenge).
- `GET /curate` with valid credentials → 200; snippet cards render.
- `GET /health` → 200 (unaffected by curator pipeline).
- `GET /` (landing) → 200 (unaffected).

### Snippet listing

- ~240 cards render with: code block, language badge, provenance row (`method` / `source` / `license`), any existing tags, any existing `curator_note`, and the review-branch label in the header.
- "Reload" button re-fetches from MCP.

### Curation loop (browser)

1. Click **Note** on any card → inline textarea opens → type a note → Save → flash "Note saved on branch curate-review" appears and is dismissible (phx-click lv:clear-flash) → note text renders in the card.
2. Click **Tag** → type an existing construct/pattern ID (e.g. `construct:js:error-handling`) → Add tag → flash "Tag added on branch curate-review" → tag badge appears on the card.
3. Click **Status** → click `approved` / `weak` / `regenerate` → flash confirms → status badge updates on the card.
4. Confirm flashes survive re-renders and are dismissible by clicking the × button.

### Safety

- After any write, confirm **graph main is clean**: the change does NOT appear when reading the main graph (read_context always targets main; the review branch is uncommitted in Dolt working set).
- Tag a non-existent target ID → error flash "Tag failed: object node not found".
- Invalid status → rejected by `valid_statuses/0` before reaching MCP.

### Auth depth (LiveSocket)

- The WS `/live` endpoint independently requires the curator session flag set by BasicAuth. An unauthenticated WebSocket connect is halted by the `SenseWeb.CuratorAuth.on_mount/4` hook (redirects to `/`). This prevents curator writes via a crafted WS connection even if the HTTP layer were bypassed.

---

## Automated evidence (build f82d0ae)

| Check | Result |
|---|---|
| `mix test` | **52 tests, 0 failures** |
| `mix format --check-formatted` | exits 0 |
| Frontend verifier | PASS — live browser walkthrough (note → tag → status), 240 cards rendered, WS-auth halt proven |
| Backend verifier | PASS — AXIS-1 gate (write to main/null rejected fail-closed), review-branch-not-live, provenance preserved, tag pos/neg, status enum, 240-edge relations join, blank/main config guard |
| Semantic-drift verifier | PASS — never-write-to-live invariant intact; no drift |
| Test-practices verifier | PARTIAL (non-blocking) — auth tests behavior-pinned; minor follow-up noted below |
| Audit | PASSED — reality=real, significance=significant |
| Evidence screenshots | Published to handgemacht-ai/sense-evidence-previews (PR #5) |

---

## Known limitations / follow-ups

| # | Item | Blocking for deploy? |
|---|---|---|
| 1 | **Prod secrets missing** on Fly `sense-prod`: `SENSE_CURATOR_USERNAME`, `SENSE_CURATOR_PASSWORD`, `ASH_KG_MCP_URL` must be set before/at epic→main deploy or the app boot-loops (fail-fast guard). | Yes — set before deploy |
| 2 | **No prod-reachable AKG MCP endpoint yet**: `/curate` is functional in dev/live verification but the production graph endpoint is a separate follow-up. | Soft — `/curate` can be deployed but unusable until endpoint is live |
| 3 | Malformed-payload tests assert no-crash but not the specific error-flash text (test-strengthening, non-blocking). | No |
| 4 | No PR CI workflow in the sense repo; inline evidence substituted. Follow-up: add GitHub Actions to sense repo. | No |
| 5 | `mix assets.bundle` must be re-run after bumping Phoenix/LiveView dep versions (pre-committed static-asset convention). | No |
