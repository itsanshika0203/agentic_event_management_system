# Smart Event Management – Member 2: Venues, Events and RAG

Everything in the Member 2 guide, implemented inside `backend/app/`.

## Quick start
```bash
docker compose up -d                  # PostgreSQL + pgvector
cd backend
cp .env.example .env                  # set OPENAI_API_KEY (or EMBEDDING_PROVIDER=local for offline)
uv sync                               # or: pip install -e .
uv run alembic upgrade head           # venues, events, knowledge_* tables + pgvector extension
uv run python -m scripts.seed         # sample venues + "AI Workshop" this weekend
uv run python -m app.rag.pipeline     # index documents/*.pdf (safe to re-run)
uv run uvicorn app.main:app --reload  # http://localhost:8000/docs
uv run pytest                         # 33 tests, no database server needed
```
Until Member 1's auth is merged, send headers `X-User-Id: 1` and `X-User-Role: ADMIN` (or `USER`).

## What is where
| Guide item | File |
|---|---|
| Venue / Event models | `app/models/venue.py`, `event.py` |
| knowledge tables | `app/models/knowledge.py` |
| Schemas | `app/schemas/` |
| Services + `check_venue_availability` | `app/services/venue_service.py`, `event_service.py` |
| APIs | `app/api/venues.py`, `events.py` |
| Migration | `alembic/versions/0001_venues_events_knowledge.py` |
| Seed | `scripts/seed.py` |
| Six policy PDFs | `documents/` (regenerate: `python -m scripts.generate_policy_docs`) |
| RAG | `app/rag/loader.py, chunker.py, embeddings.py, retriever.py, pipeline.py` |
| Agent tool | `app/tools/rag_tools.py` → `search_event_policy(query, top_k)` |

## Shared contracts (Member 3 / 4 use these)
- `search_events(db, keyword, date_from, date_to, venue_id, status)` → list of events
- `get_event(db, id)`, `create_event(db, user, data)`, `update_event(db, user, id, data)`, `cancel_event(db, user, id)` (create/update/cancel admin-only)
- `check_venue_availability(db, venue_id, start, end)` → `{"available": bool, "conflicts": [events]}`
- `search_event_policy(query, top_k)` → `[{content, source, chunk_index, score}]`
- Errors: `{detail, code}` with proper HTTP status.

## Decisions to confirm with the team (week 1)
1. **Event time:** `start_time` + `end_time`, stored as naive UTC.
2. **IDs:** integer `events.id` / `venues.id` (confirm with Member 1).
3. **Policy numbers:** 24-hour cancellation deadline, one registration per user per event. They must match Member 1's code.
4. **Overlap rule:** back-to-back events are allowed; cancelled events never block a venue.

## Open TODOs that depend on others
- `events.created_by` → add FK to `users.id` after Member 1's migration; set `down_revision` of `0001_m2` to Member 1's revision.
- `cancel_event` → mark the event's registrations as cancelled (cancellation cascade, week 4, with Member 1).
- `core/stubs.py` → delete once `core/security.py` exists (`core/deps.py` switches automatically).
- Week 7 tuning: edit `QUESTIONS` in `tests/test_rag.py`, then adjust `CHUNK_SIZE`, `CHUNK_OVERLAP`, `RAG_TOP_K` in `core/config.py`.

## Suggested PRs (small, into `develop`, branch `feature/m2-event-api` etc.)
1. `feat: venue and event models + migration` (push in the first 2 days)
2. `feat: venue/event services and APIs`
3. `test: events, venues, availability` + `feat: seed script`
4. `docs: six policy documents`
5. `feat: rag loader, chunker, embeddings, pipeline`
6. `feat: retriever and search_event_policy`
