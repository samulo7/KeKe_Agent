# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository status

This repository currently contains planning/specification documents rather than an implemented application codebase.

Primary source documents:
- `Company_Agent_PRD_v1.5.md`
- `tech-stack.md`

There is currently no verified application source tree, `README.md`, package manager manifest, build config, test config, Cursor rules, or Copilot instructions checked into the repository.

## Source-of-truth rules

- Treat `Company_Agent_PRD_v1.5.md` as the authoritative requirements document.
- Use `tech-stack.md` as implementation guidance only when it does not conflict with the PRD.
- If PRD and tech stack guidance differ, follow the PRD.

## Always apply before writing code

These rules are mandatory for any future implementation work:

- Before writing any code, fully read `memory-bank/@architecture.md`.
- Before writing any code, fully read `memory-bank/@game-design-document.md`.
- If either required memory-bank file is missing, stop and ask the user before generating code.
- After each major feature or milestone, update `memory-bank/@architecture.md` so it reflects the current structure and data model.
- Prefer modular, multi-file implementations. Split by responsibility and layer instead of concentrating logic in one file.
- Do not create or extend monolithic files that mix unrelated concerns (for example UI, state, network access, business rules, and persistence in one place).
- When adding functionality, favor small focused modules such as routes/controllers, services, data-access, schemas/types, UI components, hooks, and utilities in separate files where appropriate.

## Modularity guardrails

- Avoid single giant files for pages, components, services, or workflows.
- If a change introduces multiple concerns, split them into separate files instead of growing one file into the system boundary.
- Keep database structure, business rules, integration clients, state management, and UI rendering decoupled.
- Refactors that reduce monolith structure are preferred over piling new code into an oversized file.

## Verified commands

No build, lint, test, or dev commands are currently defined in the checked-in repository.

Before suggesting or running implementation commands, first verify that application code and its toolchain files have been added (for example `package.json`, lockfiles, Vite config, test config, or backend app entrypoints).

For now, the only verified repo-level command is:

```bash
git status
```

## Product architecture to preserve

The documents describe a single internal company AI agent platform with three phase-1 business capabilities:
- leave request agent
- expense reimbursement agent
- knowledge-base Q&A agent

Planned high-level architecture:
- Frontend: React + Vite + Tailwind CSS for a conversational UI plus admin back office.
- Backend: Node.js + Express as a single application responsible for REST APIs, agent orchestration, DingTalk callback handling, and scheduled/background triggers.
- Queue/async processing: BullMQ + Redis for email retries, DingTalk status polling fallback, and OCR jobs.
- Data layer: PostgreSQL + pgvector for both transactional business data and document embedding search.
- File storage: Alibaba Cloud OSS for invoice images.
- LLM/RAG: Qwen models via Alibaba DashScope's OpenAI-compatible API, plus `text-embedding-v3` for document embeddings.
- External integrations: DingTalk for SSO/org/approval/work notifications and NetEase enterprise SMTP for approval-result emails.
- Deployment target: single-machine ECS deployment with Nginx + PM2.

## Preferred implementation patterns

Use these as default guidance when code is added to the repo:
- Frontend state: prefer React Context + local state before introducing Redux/Zustand.
- Frontend networking: keep API clients separated from UI components; do not embed request logic directly into large page components.
- Backend API shape: prefer clear REST-style Express routes with business logic extracted into services.
- Integration boundaries: keep DingTalk, SMTP, OCR, OSS, and LLM clients in dedicated integration modules instead of scattering SDK calls across the app.
- Data access: isolate SQL/repository access from route handlers and orchestration logic.
- Validation: define request/data schemas in dedicated modules and reuse them across handlers/services.
- RAG pipeline: keep ingestion, chunking, embedding, retrieval, and citation formatting as separate concerns.
- Favor the documented stack; do not introduce extra frameworks or infrastructure unless the checked-in code or the user explicitly requires it.

## Core business boundaries

These rules are repeatedly emphasized across the docs and should not be changed casually:
- DingTalk owns approval chains. The system initiates approvals, syncs results, and displays status, but does not define approval routing.
- Authentication is DingTalk SSO with server-side session plus HttpOnly cookie; no separate local account system.
- System roles express backend permissions and data scope only; they do not define approval-chain semantics.
- Leave and expense flows are conversational wrappers around existing DingTalk forms/templates, not newly invented workflows.
- The system state machine is a coarse-grained business status model (`DRAFT`, `APPROVING`, `APPROVED`, `REJECTED`, `CANCELLED`) that mirrors DingTalk results rather than driving workflow nodes.
- Knowledge-base answers must cite source location. Required citation format is document name + business category + upload time + location (PDF page, or Word heading/paragraph).
- When no relevant knowledge-base document exists, the system must return an explicit “not found” style result rather than filling the gap with generic LLM knowledge.

## Planned backend domains

The PRD/tech stack consistently imply these main backend areas:
- auth/session and DingTalk SSO
- DingTalk org sync and approval instance creation
- leave request domain
- expense report domain, including invoice OCR and duplicate-invoice checks
- knowledge-base ingestion, chunking, embeddings, retrieval, and citations
- callback idempotency and polling fallback
- email delivery and replay logs
- admin audit and manual compensation tools

## Important data model concepts

Expected core tables called out in the documents:
- `users`
- `leave_requests`
- `expense_reports`
- `invoices`
- `documents`
- `document_chunks`
- `approval_logs`
- `email_logs`
- `audit_logs`

Important shared status/sync fields for leave and expense records:
- `status`
- `approval_instance_id`
- `current_node_name`
- `current_approver`
- `last_synced_at`
- `rejection_reason`

## Integration constraints

- DingTalk callbacks must use idempotency keyed by `approval_instance_id + event_id`.
- Callback handling must include signature verification, timestamp window checks, and nonce replay protection.
- OCR failures should remain recoverable: mark `ocr_failed = true` and allow manual entry instead of blocking the business flow.
- Email sending is asynchronous and retryable; failures should be logged for admin replay instead of being surfaced as business-state corruption.

## Working in this repo

Because this repo currently stores requirements rather than implementation:
- read the PRD before making product or architecture decisions
- do not invent commands, file paths, or framework structure that are not yet checked in
- if application code is later added, update this file with the actual build/test/dev commands and the real directory structure
