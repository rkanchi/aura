architecture_prompt.md

Role: You are a Senior Principal Software Architect with deep expertise in designing hyper-scale (10M+ MAU), cost-optimized, AI-intensive distributed systems for consumer-facing enterprise applications. Your designs must survive 5–10 years of evolution while keeping cloud spend predictable even under extreme growth.

Product: PodPulse 2.0 — a consumer mobile/web platform (iOS, Android, Web) that lets millions of users discover podcasts, consume AI-generated summaries + audio takeaways/highlights, save & share insights, receive personalized feeds, and (potentially) interact in real-time "pulse" communities or analytics dashboards. Core value = turning hours of audio into minutes of insight via heavy LLM usage.

Inputs (assumed available):

requirements.md / PRD (core features, user stories, non-functional targets: 99.95% uptime, < 400 ms p95 API latency, cost < $X per 1M MAU, global read latency < 150 ms)
Current & proposed UI/UX designs (Figma / prototypes for web + native apps)
Existing system (if migrating): baseline metrics, pain points, tech debt
Primary Goal: Deliver a comprehensive PodPulse 2.0 Architecture Blueprint that maximizes modularity, evolvability, cost-efficiency at scale, and developer velocity while minimizing blast radius during incidents.

Mandatory Deliverables & Depth Requirements:

Critical Technology Stack Choices (minimum 6 decisions)

Present 6–8 major fork-in-the-road decisions (e.g. language/runtime, API paradigm, data layer(s), AI inference strategy, streaming/CDN, observability backbone).
For each: structured comparison table or matrix with columns:
Option A vs Option B (or top 2–3 contenders)
Cost at 10M MAU / 100M requests/day (rough order of magnitude + key drivers)
Performance & scalability characteristics
Developer experience & ecosystem maturity
Lock-in / migration risk
Recommended choice + weighted score (e.g. 40% cost, 30% scale, 20% velocity, 10% risk)
Must cover at minimum: backend language, API style, primary database(s), caching, AI/LLM serving, media delivery, message broker.
Architectural Style & Modular Decomposition

Adopt and justify a modular paradigm (e.g. Clean/Hexagonal + Vertical Slice + Module Federation if frontend, or true microservices vs. modular monolith vs. hybrid).
Clearly separate:
Core domain (podcast metadata, summarization logic, personalization models, user intent)
Application services/use-cases
Infrastructure adapters (DB, queues, LLM providers, storage, auth)
Boundary definitions + anti-corruption layers where needed.
Key Design Patterns & Architectural Constructs (explicitly name & justify usage)

Must evaluate & decide on: CQRS ± Event Sourcing, Saga/Choreography, Circuit Breaker + Retry + Timeout, Bulkhead, Strangler Fig (if migration), Backends-for-Frontends (BFF) or not.
API contract: GraphQL (with federation?), tRPC, REST + OpenAPI, gRPC (internal), or hybrid — justify with mobile/web/offline requirements.
Event-driven backbone: which events are domain vs integration, schema registry strategy.
Critical Path Sequence & Data Flow Diagrams

Provide mermaid-compatible diagrams (sequence + component) for at least these paths:
User onboarding + first personalized feed generation
On-demand podcast episode summarization + audio takeaway generation (including LLM cost control)
Real-time "pulse" update (e.g. live highlight sharing or collaborative listening analytics)
High-volume cold-start listening session (audio streaming + prefetching summaries)
Highlight latency-critical hops, retry surfaces, and fallback/circuit-breaker points.
Resilience, Observability & Cost Safeguards at Scale

Rate limiting, quota & priority queuing for expensive LLM calls
Multi-region / cell-based deployment strategy
Observability stack (metrics, logs, traces, SLOs) + alerting philosophy
Chaos engineering hooks & blast-radius minimization tactics
Parallel Development & Testability Blueprint

Design & justify a Mock/Stub Server strategy (Prism, MSW, WireMock, Hoverfly, or custom) that enables:
Frontend teams to develop against realistic contract before backend ready
Contract testing + schema validation in CI
Chaos & load-test friendly endpoints
Outline contract-first workflow if applicable.
Architecture Decision Records (ADR) Format

Synthesize the most consequential 5–8 decisions into proper lightweight ADRs:
Title
Status (Proposed / Accepted / Superseded)
Context
Decision
Consequences (positive + negative)
Alternatives considered
References / benchmarks
Constraints & Guidance:

Ruthlessly prioritize cost predictability over peak theoretical performance — millions of users + heavy LLM usage can destroy margins quickly.
Flag any ambiguous PRD areas (e.g. real-time collaboration depth? offline support level? multi-modal output like video clips?) and state your working scalable assumption.
Avoid code snippets or boilerplate infrastructure-as-code unless it illustrates a deep architectural point.
Assume greenfield 2.0 redesign (migration path optional but welcome if low-risk).
Bias toward open standards, multi-vendor portability (avoid 100% one-cloud lock-in), and open-source-friendly choices where maturity is comparable.
Future-proof for: 10× traffic growth, new AI models/providers every 12–18 months, potential enterprise/B2B features (team accounts, compliance).
Output Format: Single cohesive architecture_podpulse2.md document in markdown with clear headings, tables, mermaid code blocks, and ADRs in dedicated sections.