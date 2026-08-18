---
title: "What It Takes to Run Agentic Systems in Production: A Systems Guide"
author: Murali Kotharamban
pubDatetime: 2026-08-18T00:00:00.000Z
featured: false
draft: false
tags:
  - AI
  - GenAI
  - Agents
  - Platform Engineering
  - LangGraph
  - MCP
  - RAG
  - Observability
description: "A map of the runtime, memory, tools, knowledge, security, observability, evaluation, and infrastructure required to turn an agent loop into a production system."
---

An agent can look convincing long before it is safe or reliable.

A model receives a prompt, chooses a tool, observes the result, and produces an answer. In a controlled demo, that loop may be enough. In production, every step opens a different engineering concern:

- **What should the model see for this step, and what should be retained for later?** That is **context engineering**, **state**, and **short- or long-term memory**.
- **Which user or workload identity authorizes a tool call, and where do its credentials live?** That is **tool calling**, **authorization**, **secret management**, and the wider **agent security** boundary.
- **What survives a process restart, and where does the run resume?** That is the **agent runtime**, **durable execution**, **checkpointing**, and **session or run state**.
- **Does retrieved material actually support the generated claim, and was the caller allowed to retrieve it?** That is **RAG**, **grounding**, **provenance**, and access-aware retrieval.
- **What happens if a tool changes an external system but the local run fails before recording success?** That is **side-effect safety**, **idempotency**, **reconciliation**, and recovery.
- **Can we reconstruct the model calls, tool calls, state transitions, and errors from one run?** That is **agent observability**, usually built from traces, logs, metrics, and correlated events.
- **How many tokens, model calls, tool calls, and euros did the run consume?** That is **usage attribution**, **cost observability**, and budget enforcement.
- **Did a new prompt, model, retrieval strategy, or harness change improve the system rather than merely change it?** That is **evaluation**, usually shortened to **evals**, together with regression testing and quality gates.

Those questions are sometimes described as infrastructure around the agent. I think that understates their role. They determine what the agent is allowed to know, what it can do, how failure is contained, and whether its output deserves to be used. They are part of the agentic system itself.

This series is a systems guide to that larger problem. It begins with the concepts and design choices that apply across frameworks and cloud providers. It then tests those ideas through a working reference project called Groundwork Agent. The project makes the discussion concrete; it is not a substitute for the theory.

## An Agentic Application Is More Than a Model Loop

The familiar diagram of an agent is a circle:

```text
model -> tool -> observation -> model
```

That diagram describes one decision loop. It does not describe the production system responsible for running it.

A more useful mental model has several interacting planes:

```text
Human oversight
       |
Identity, policy, and guardrails
       |
Agent runtime and orchestration ---- observability, evals, usage, and cost
       |
Agent harness
  |-- context engineering
  |-- model inference
  |-- tool calling and MCP
  `-- RAG and grounding
       |
State, memory, sessions, and artifacts
       |
Platform infrastructure
```

The exact boundaries vary, but each plane answers a different class of questions.

**Model inference and reasoning** decide what to do next or generate structured output. They introduce nondeterminism, provider limits, regional availability, latency, token usage, and changing model behaviour.

**The agent harness** prepares each model call and manages the immediate reasoning-and-action cycle. It includes system prompts, context assembly, tool definitions, response schemas, model selection, tool routing, and loop controls. Harness design changes as models and product requirements change; it should not silently own durable business state or broad credentials.

**Tool calling and execution** let the system affect the world. Local functions, service APIs, and MCP tools need schemas, authorization, timeouts, error contracts, and policies around consequential actions. A valid tool schema is not permission to execute it, and a model's tool call is a proposal rather than an authorization decision.

**Context engineering, state, and memory** determine what the model can use now and what the application preserves for later. Request context, graph state, conversation summaries, user preferences, and audit history solve different problems. Treating all of them as “memory” makes retention, correctness, and privacy difficult to reason about.

**RAG and grounding** connect generated claims to source material. Production retrieval involves ingestion, access control, provenance, freshness, retrieval and reranking, citation validation, and defences against untrusted content. Similarity alone does not establish support, so retrieval quality and grounded-answer quality need separate evals.

**Orchestration and the agent runtime** turn a loop into an operating lifecycle. Orchestration defines the workflow; the runtime gives a run identity, explicit state, checkpoints, scheduling, budgets, cancellation, retries, durable pauses, concurrency rules, recovery behaviour, and isolation boundaries.

**Identity, policy, guardrails, and security** decide which actor can cause which action over which data. They also cover secret brokerage, tenant boundaries, prompt injection, egress, sandboxing, human approval, and auditability. Guardrails are enforcement points within this larger security model, not a substitute for authorization or isolation.

**Agent observability and evals** answer two related but different questions: what happened during one run, and whether the system is good enough across many runs. Traces, events, logs, metrics, evaluation datasets, deterministic checks, model-based judges, and human calibration all have roles. Observability supplies evidence from runs; evals turn representative behaviour into repeatable quality signals and release decisions.

**Platform infrastructure and data** make the other layers real: deployment, queues, databases, artifact storage, network policy, secret systems, backups, CI/CD, observability backends, capacity, and cost controls.

No framework owns this whole picture. Frameworks provide useful primitives, but the application team still owns the contracts between them.

## Why Production Changes the Problem

Traditional services already deal with timeouts, authentication, partial failure, and observability. Agentic applications inherit all of that and add several pressures.

### Nondeterminism sits inside the control flow

The same input can produce different plans, tool arguments, and outputs. That does not mean the entire system must be unpredictable. Durable state transitions, policy checks, schemas, budgets, and side-effect handling can remain deterministic around the model. The design challenge is deciding where flexibility is useful and where an invariant must hold.

### Runs can be long-lived

An agent may wait for a person, a provider, a scheduled retry, or new evidence. Keeping an HTTP request or worker alive is not a lifecycle strategy. Long-lived work needs persistent identity and state, durable suspension, resume authorization, cancellation, and a clear owner.

### Tools create real side effects

A repeated read may be harmless; a repeated CRM update, email, purchase, or infrastructure change may not be. A process can fail after an external system accepts a write but before the local workflow records success. Checkpointing computation does not close that gap. Idempotency, operation records, reconciliation, and human confirmation become part of tool design.

### Context can be untrusted and sensitive

Documents, CRM fields, web pages, tool results, and previous messages may contain secrets, personal data, stale facts, or instructions that conflict with system policy. Context is data, not authority. Access controls must be enforced before retrieval and tool execution, not delegated to the prompt.

### Quality is multidimensional

A fluent answer can still use the wrong source, omit a requirement, call an unnecessary tool, violate a schema, expose data, or cost too much. One aggregate “quality score” hides those failure modes. Production evaluation needs a set of checks aligned with the actual product contract.

### Cost and capacity accumulate inside the loop

One user request can expand into many model calls, retrieval queries, tool calls, retries, and delegated tasks. Measuring the final request latency or total token count is not enough. Limits and attribution need to exist at run, step, provider, model, tool, and tenant boundaries.

These are the reasons a successful demo is evidence of capability, not evidence of readiness.

## The Questions This Series Will Work Through

The parts are organised as one system, not as a catalogue of AI tools.

### The agent loop, harness, orchestration, and runtime

We will start with the basic observe–decide–act loop, then separate the model-facing agent harness from workflow orchestration and the wider agent runtime. We will examine explicit state machines, checkpoints, side effects, idempotency, retries, budgets, cancellation, durable interrupts, and recovery after process failure.

### Context engineering, state, and memory

We will distinguish data needed for one model call from durable workflow state, conversation memory, learned preferences, and external knowledge. The design must include ownership, freshness, retention, deletion, summarisation, and protection against a memory becoming an unreviewed source of truth.

### Tool calling, tool execution, and MCP

We will compare in-process functions, service APIs, and MCP tools. The important questions are capability discovery, input and output contracts, authorization, credential isolation, audit, timeouts, error classification, human confirmation, and how a tool failure returns to the reasoning loop safely.

MCP standardises how tools can be described and invoked; its specification still leaves access control, input validation, rate limiting, output sanitisation, timeouts, and user control to the implementation. That is the boundary the series will make explicit. [The versioned MCP tools specification describes both the protocol shape and these security considerations](https://modelcontextprotocol.io/specification/2026-07-28/server/tools).

### RAG, grounding, and evidence

Retrieval-augmented generation is often shown as chunking, embedding, similarity search, and prompt injection. We will extend that model to source ingestion, document identity, versions, ACL filtering, hybrid retrieval, reranking, provenance, citation spans, freshness, and evaluation of whether evidence actually supports a claim.

### Structured data across boundaries

Schemas will appear at model outputs, tool inputs, tool results, graph state, APIs, events, and stored artifacts. We will examine where validation belongs, how versions evolve, how repair attempts are bounded, and why syntactically valid JSON can still be semantically wrong.

### Models, routing, and regional inference

The system will use models through AWS Bedrock and Google Vertex AI. We will separate provider integration from routing policy and consider capability, latency, availability, price, structured-output support, fallback behaviour, and EU regional requirements. Residency is a property of the whole data path, not only the region string on one model call.

### Identity, authorization, and secrets

The model should receive capabilities, not reusable credentials. We will follow identity from the user through the control plane, runtime, and tool boundary; look at short-lived or brokered credentials; limit secret exposure in prompts and traces; and define authorization for both reads and consequential writes.

### Error handling, retries, and recovery

Provider failures, invalid structured output, rejected tool arguments, authorization denials, missing evidence, rate limits, and unexpected exceptions need different destinations. Some are safe to retry, some should return to the model with sanitised feedback, some require a person, and some must stop the run.

### Agent observability, usage, and cost

We will trace the run, graph transition, model request, prompt template version, safe parameters, retrieval query, tool call, tool response metadata, retry, token usage, latency, and cost. Telemetry also has a data-governance problem: blindly recording prompts and tool results can turn the observability system into a second sensitive database. OpenTelemetry provides common signal concepts and context propagation; application-level agent semantics still need to be designed. [OpenTelemetry documents its traces, metrics, logs, and baggage signals](https://opentelemetry.io/docs/concepts/signals/).

### Evals and continuous improvement

We will build a versioned dataset and combine deterministic contract checks, retrieval and citation measures, tool-use checks, model-based judges, and human review. The goal is not to declare an LLM judge objective. It is to create repeatable evidence, calibrate it against human decisions, expose regressions, and make architecture or model changes earn their way through a quality gate.

### Sandboxing, multi-agent design, and operations

We will separate a long-lived orchestration service from short-lived risky execution. That includes egress, filesystem, credentials, CPU and memory limits, artifacts, cleanup, and the difference between a container and a meaningful sandbox. Only after a single workflow is measurable will we introduce specialist agents and test whether delegation improves quality enough to justify the extra coordination, state, cost, and failure surface.

Finally, we will bring the system together through deployment, security controls, SLOs, incident handling, backup and recovery, supply-chain controls, and a complete run with a deliberate failure drill.

## What the Reader Should Take Away

The aim is not to produce one “correct” reference architecture. Agentic systems serve different risk levels and workloads. A customer-support assistant, a research workflow, and an infrastructure-changing agent should not have identical controls.

The aim is to make the decisions legible. By the end, a reader should be able to:

- decompose an agentic application into model inference, the agent harness, orchestration and runtime, context and memory, tools, RAG, security, observability, evals, and platform concerns;
- identify which guarantees come from a framework and which remain application responsibilities;
- choose state, tool, retrieval, model, and runtime patterns based on failure and trust boundaries;
- define agent observability and evals that measure behaviour rather than only service availability;
- reason about identity, secrets, data residency, isolation, and human control across the whole run;
- build an incremental path from capability demonstration to an operable system.

To keep that discussion grounded, the series needs a project with consequences beyond producing plausible text.

## The Running Project: Groundwork Agent

Groundwork Agent is an evidence-backed workspace for responding to RFPs and solution-assurance questionnaires.

A solutions engineer begins with an opportunity in Twenty CRM and a questionnaire from a prospective customer. Groundwork brings together permitted account context, approved policies, technical documents, and repository evidence; decomposes the questionnaire into requirements; drafts cited answers; identifies unsupported commitments; routes review; produces an artifact; and eventually writes only approved status back to the CRM.

This use case makes the concepts above visible:

- an unsupported answer can become a contractual or security commitment;
- correct evidence shown to the wrong account is still a data leak;
- CRM and knowledge tools require narrow identity and permissions;
- questionnaires create structured inputs and outputs with partial, reviewable progress;
- people must be able to pause, correct, approve, and resume work;
- answer quality, evidence quality, tool behaviour, latency, and cost can be evaluated separately.

The project is synthetic and open-source oriented. The series will not claim real customers, production scale, savings, or incidents that have not occurred.

## Why Twenty Is a Source, Not the Agent Platform

Twenty provides a relatable open-source CRM and generated workspace APIs. Its API keys can be associated with roles that constrain access. [Twenty documents its REST and GraphQL APIs and role-scoped API keys](https://docs.twenty.com/developers/extend/api).

The agent stays outside the CRM because the assurance workflow has a different lifecycle and trust boundary. Twenty owns commercial context. Groundwork owns questionnaire files, extracted requirements, evidence versions, citations, draft history, review decisions, run state, evaluations, and generated artifacts. It also needs sources that do not belong in a CRM record.

Groundwork accesses Twenty through a dedicated MCP service using a scoped key rather than connecting directly to the CRM database. The foundation exposes only two read capabilities, `get_opportunity` and `get_company`. There is no CRM mutation tool yet. This makes Twenty a useful source and system of record without granting the model general CRM access.

## The Reference Architecture

The initial boundary looks like this:

```text
Browser
   |
React application
   |
FastAPI control plane
   |
LangGraph orchestration runtime ------> platform state and artifacts
   |                   |
   |                   +-------------> isolated worker (planned)
   |
   +----> scoped Twenty MCP service ----> Twenty CRM
   |
   +----> retrieval and model gateway (planned)
```

The API owns the public authentication and authorization boundary. The runtime owns workflow transitions and, later, checkpoints, policies, retries, and interrupts. MCP adapters hold external credentials and expose narrow capabilities. Platform databases and object storage hold operational state and artifacts. Work that processes hostile files or executes generated code belongs in a bounded, short-lived worker rather than the long-running runtime.

This architecture is not presented as finished. It is a set of boundaries that the series will test and revise.

## What Exists Today

At commit `f795200`, the repository contains independently packaged React, FastAPI, LangGraph, and Twenty MCP services; a local topology for platform data and Twenty; typed proposal-run and evidence contracts; a minimal `START -> accept_run -> END` graph; two read-only Twenty tools that fail closed without a credential; synthetic evaluation cases; and initial architecture, threat-model, and operations documentation.

The graph is compiled without a persistent checkpointer. The retrieval pipeline, model gateway, production telemetry, isolated worker, and CRM write-back are planned. A vector-capable database or container boundary should not be mistaken for a working RAG pipeline or a production sandbox.

The current repository check is:

```bash
make check
```

It validates Python and TypeScript code, tests, the frontend build, and Compose configuration. It does not prove that the full distributed topology, authenticated CRM integration, or future agent workflow works end to end. Those checks must be completed before later implementation articles claim end-to-end behaviour.

## How the Series Will Use the Project

Each part will follow the same progression:

```text
concept -> production pressures -> patterns and trade-offs
        -> Groundwork design -> implementation -> evaluation
```

The concept should stand on its own. The project then gives us somewhere to make a choice, expose its cost, implement it, inject failure, and measure the outcome. When an implementation is missing, the article will say so. When it exists, the article will point to a named revision and reproducible evidence.

That distinction matters. A Compose file parsing successfully is evidence about configuration. It is not evidence that a distributed agent recovers correctly. A model returning valid JSON is evidence about syntax. It is not evidence that the answer is supported. A trace arriving in a backend is evidence about instrumentation. It is not evidence that the system is observable enough to diagnose a failed run.

## Starting With the Loop

The first detailed part begins with the smallest familiar object: the agent loop. We will define what the loop actually does, separate model reasoning from orchestration and runtime, and examine the controls needed when it can pause, retry, create side effects, and outlive a process.

Only then will we return to Groundwork's one-node graph and turn it into a recoverable, bounded run.

That order is the method for the whole series: understand the system problem first, then make the implementation prove what we think we understand.
