# Plan: Multi-Repository Topology, Service Boundaries & Naming Conventions Specification

## Phase 1: Task Registration & Sprint Initialization
- [x] Register task `V1_006: Multi-Repository Architecture, Service Boundaries & Naming Conventions` under `# Roadmap & Tasks` in `docs/PROGRESS.md`.
- [x] Initialize `SPRINT.md` with `Author/Assignee: @siddhantdeshwal1`, `Task: V1_006 - Multi-Repository Architecture, Service Boundaries & Naming Conventions`, and `Started: 2026-11-22 16:38:00`.

## Phase 2: Architecture Documentation Updates in docs/PROGRESS.md
- [x] Update Section 1 (The Frontend Layer) to establish the Web Client (`wikime-web`) as the primary presentation edge for MVP and defer `wikime-mobile` to Stage 3.
- [x] Add Section 8 (Multi-Repository Topology, Service Boundaries & Naming Conventions) in `docs/PROGRESS.md` detailing:
  - Repository naming rules and why repositories are environment-agnostic.
  - Where `prod`, `staging`, and `dev` belong (branches, image tags, K8s namespaces, configs).
  - Strict in-scope vs. forbidden responsibilities for `wikime-web`, `wikime-backend`, `wikime-ai`, `wikime-db`, `wikime-extractor`, and `wikime-mobile`.
  - Staged roadmap (Stage 1: Core Web MVP, Stage 2: Deep Extraction, Stage 3: Mobile Ecosystem).
  - Inter-service communication matrix (HTTP/2 SSE, gRPC, RESP3, native Qdrant gRPC).

## Phase 3: High-Level Architecture Diagram Synchronization
- [x] Update PlantUML architecture diagram (`docs/diagrams/high_level_design.puml`) reflecting the web-first client (`wikime-web`), microservice repository boundaries, and inter-service protocols.

## Phase 4: Verification, Post-Task Summary & Git Commit
- [x] Update `Finished: YYYY-MM-DD HH:MM:SS` in `SPRINT.md` and mark `V1_006` completed in `docs/PROGRESS.md`.
- [x] Generate root `GIT.md` change summary conforming to the `git-commit` skill.
- [x] Execute git commit with message `GENAI=YES docs(architecture): [V1_006] document multi-repository topology, service boundaries, and naming conventions`.
- [x] Archive `GIT.md` to `docs/history/GIT_V1_006.md` and move `plan_repository_structure_spec.md` to `docs/history/`.
