# Plan: Integrate Advanced Agent Design, Security, Caching, Routing & Enterprise RBAC

## Phase 1: Planning & Specialized Architecture Design
- [x] Create specialized subagent analyses for:
  - Intent classification & Semantic Query Caching
  - Dynamic Frontier vs SLM Model Routing
  - Llama Guard, Prompt Injection & PII Redaction
  - Enterprise RBAC & Team Multi-Tenancy
  - Pipeline Integration & MVP Phasing Strategy

## Phase 2: MVP Ingestion & Prompt Refinements (Immediate Scope)
- [x] Enhance Section 4.2 of `docs/PROGRESS.md` with explicit Agent Persona, Tone, Pre-Retrieval Intent Taxonomy, and Negative Grounding rules.
- [x] Specify URL and Note PII sanitization hooks in API Gateway (Phase 2 / Phase 4).

## Phase 3: Post-MVP Architecture Specifications (Phased Expansion)
- [x] Add Section 7: Enterprise Safety, Caching & Dynamic Routing Layers in `docs/PROGRESS.md`:
  - Layer 4.3: Pre-Retrieval Intent Classifier & Semantic Vector Cache (Redis/GPTCache)
  - Layer 4.4: Dynamic Complexity Router (Fast SLM vs Frontier LLM)
  - Layer 4.5: Llama Guard 3 & Adversarial Defense Gate
  - Layer 5.4: Enterprise RBAC, Workspace Tenants & ACL Metadata Schema
- [x] Add Flow F: Guarded, Cached & Dynamically Routed Retrieval Sequence in Section 6 of `docs/PROGRESS.md`.

## Phase 4: Implementation Phases & Roadmap Registration in docs/PROGRESS.md
- [x] Register `PHASE_08: Enterprise Safety, PII Redaction & Llama Guard (PHASE_08)`.
- [x] Register `PHASE_09: Semantic Query Cache & Dynamic Frontier/SLM Routing (PHASE_09)`.
- [x] Register `PHASE_10: Multi-Tenant RBAC & Organization Collaboration (PHASE_10)`.
- [x] Register new roadmap tasks `V1_003` (Enterprise Security & Guardrails) and `V1_004` (Semantic Cache & Adaptive Routing) under `# Roadmap & Tasks`.

## Phase 5: High-Level Architecture Diagram Updates
- [x] Update `docs/diagrams/high_level_design.puml` with Semantic Cache, Llama Guard, Model Router, and RBAC components.

## Phase 6: Verification & Plan Archiving
- [x] Verify markdown formatting, links, and PlantUML syntax integrity.
- [x] Archive `plan_advanced_agent_architecture.md` into `docs/history/`.
