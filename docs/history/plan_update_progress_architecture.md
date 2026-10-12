# Plan: Update docs/PROGRESS.md with Architecture Refinements & Remove Roadmap

## Phase 1: Planning & Verification
- [x] Review current sections in `docs/PROGRESS.md` to map all insertion and modification points.

## Phase 2: Updating Ingestion, Scraping & Structuring (Phases 1–3 & Layers 1–3)
- [x] Update Phase 1 and API Gateway specs with URL Sanitization, Canonicalization, and rapid Vector DB Idempotency check.
- [x] Update Phase 2 and Microservice Layer specs to strictly enforce zero-video-download architecture (microservices take URL and return transcripts/captions/markdown).
- [x] Update Phase 3 to remove video frame extraction and introduce Parent-Child chunking (300–500 token overlapping chunks).

## Phase 3: Updating Storage, Retrieval & Operational Feedback (Phases 4–5 & Layers 4–6)
- [x] Update Phase 4 and Database Layer specs with the Parent-Child Retrieval Model (Summary vector, Chunk vectors, HyDE vectors linked via `item_id`) and Fire-and-Forget client feedback model.
- [x] Update Phase 5 and Retrieval Flows with deduplicated retrieval by `item_id`, parent payload assembly, and Jev filtering.
- [x] Update Flow A, Flow B, and Flow D execution paths in Section 6 to match the refined mechanics.

## Phase 4: Roadmap Cleanup & Final Verification
- [x] Remove the `# Roadmap & Tasks` section from `docs/PROGRESS.md`.
- [x] Verify document integrity, consistency, and formatting across `docs/PROGRESS.md`.
