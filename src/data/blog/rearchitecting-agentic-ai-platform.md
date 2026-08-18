---
title: "Rearchitecting an Agentic App: From ClickOps Across Five Clouds to an AI Platform"
author: Murali Kotharamban
pubDatetime: 2026-08-11T00:00:00.000Z
slug: rearchitecting-agentic-ai-platform
featured: true
draft: false
tags:
  - AI
  - GenAI
  - Platform Engineering
  - Terraform
  - Observability
  - DevOps
  - Agents
  - Multi-Agent Orchestration
  - Agent Runtime
description: "An agentic app grew out of an MVP with components scattered across five clouds, all provisioned by hand. Here's how I rearchitected it — Terraform first, agent-level observability second, consolidation last — and what it taught me about building a harness for AI agents."
---

*Every fast-moving MVP eventually wakes up as a distributed system nobody designed. This is the story of rearchitecting one — and what agentic applications specifically taught me about platform engineering.*

---

## The Architecture Nobody Designed

I've been working on an agentic application — a product where users interact with an AI agent in real time, and that agent can trigger a multi-agent system to do heavier work behind the scenes.

Like most products that find traction, it started as an MVP. Different developers built different components, and each of them made a locally sensible choice with the tools they knew best:

- **Backend** → GCP Cloud Run
- **Main customer-facing agent** → Cloudflare Sandbox
- **Multi-agent system** (triggered by the main agent) → Modal Sandbox
- **Frontend** → Vercel
- **Database** → Neon
- **Analytics** → PostHog

Here's the thing: **every single one of these choices is defensible.** Vercel is great for frontends. Neon is excellent serverless Postgres. Modal is genuinely good at burstable, sandboxed compute. Cloudflare gives you low-latency stateful agent sessions at the edge. Cloud Run is a solid default for a conventional backend.

Nobody made a bad decision. But nobody made a *system-level* decision either. The architecture was the org chart — Conway's Law, applied to cloud platforms. And every one of those resources had been created by hand, in a dashboard, by whoever was working on that piece at the time.

I want to be clear about the framing here, because it matters: **this is not a story about mistakes.** Shipping an MVP on whatever each developer could move fastest with is exactly how you should ship an MVP. The mistake would have been leaving it that way after the product proved itself.

## What Actually Hurts in an Agentic App

Multi-cloud sprawl is a known tax: five dashboards, five billing accounts, five secret stores, five deployment stories. That part is well-documented and boring.

What surprised me is how much *worse* the tax gets when the application is agentic.

In a traditional web app, the unit of debugging is a request. It hits one service, maybe fans out to two more, and returns. In an agentic app, the unit of debugging is an **agent run** — a long-lived, multi-step execution that in our case crossed *every single platform*: frontend on Vercel → main agent in Cloudflare → backend on Cloud Run → multi-agent swarm on Modal → database on Neon.

When an agent run hangs, misbehaves, or burns tokens doing something weird — which dashboard do you open? We had product analytics in PostHog, and PostHog is great at what it does, but product analytics is not agent observability. It can tell you *that* users are dropping off; it cannot show you the chain of prompts, tool calls, and sub-agent handoffs that made a run go sideways.

The other structural problems were the classics, amplified:

- **ClickOps was the biggest risk of all.** No reproducibility, no staging parity, invisible config drift, and disaster recovery that lived in people's heads. If the person who configured a platform moved on, that knowledge moved on with them.
- **Every hop crossed the public internet** between different providers' networks — user-visible latency for an app that streams agent output, plus egress costs on each side.
- **Secrets everywhere.** Five platforms, five secret stores, five sets of access controls, zero shared network boundary.

## Step 1: Codify Before You Consolidate

The tempting move is to start migrating things immediately. I deliberately didn't.

The first phase was pure codification: **get every manually created resource into Terraform, changing nothing.** Cloudflare, Neon, Vercel, and GCP all have Terraform providers; Modal deployments are already code-defined. Import the existing resources, reconcile the drift, make `terraform plan` come back clean.

This felt slow. It was the most valuable thing I did, for one reason:

> **Infrastructure you've codified is a refactoring problem. Infrastructure you haven't is an archaeology problem.**

Once the entire estate was in Terraform, every later decision — consolidating compute, changing networking, standing up a proper staging environment — became a reviewable diff instead of a sequence of irreversible dashboard clicks. It also killed the bus-factor risk on day one, before a single workload moved.

## Step 2: Observe the Agent Run, Not the Services

The second phase was monitoring across all the components that make up the agentic path — the backend, the main agent in Cloudflare, and the multi-agent system in Modal — stitched together so a single run can be followed end to end.

The principle: **trace context has to travel with the agent, across every platform boundary.** When the main agent triggers the multi-agent system, the trace ID goes with it. When the multi-agent system calls the backend, same thing. The platforms are different; the run is one.

For an agentic app, I'd now argue this is the single most important operational capability — above autoscaling, above deployment speed, above almost everything. Agents fail in ways services don't: they don't just error, they *wander*. They retry things that shouldn't be retried, loop on a tool call, or produce a confidently wrong answer after a perfectly healthy-looking execution. You cannot debug that from per-service logs on five different platforms. You can only debug it from a unified view of the run.

PostHog stayed, by the way. Product analytics and agent observability are different jobs, and it's the right tool for its job.

## Step 3: Collapse Redundant Compute

Only after codifying and instrumenting did I touch the topology — because by then, I could actually *see* what each platform was contributing.

The question I asked for each component was not "which platform is best?" but: **"is this platform's unique capability load-bearing for this workload — and is it worth what we're paying for it?"**

The obvious overlap was the two sandbox runtimes: the main agent in Cloudflare, the multi-agent system in Modal. Modal's superpower is burstable, heavy, compute-bound work. But when I actually looked at what our multi-agent system was doing, it was orchestration and LLM calls — I/O-bound waiting, not number crunching. We were paying for a capability we weren't exercising, and a capability you're not exercising is just a bill.

So the multi-agent system moved into Cloudflare alongside the main agent, and Modal was retired. That's not a knock on Modal — for workloads that genuinely need what it offers, it earns its price. Ours didn't, and the move paid off three ways at once: a meaningfully smaller bill, the hottest path in the app (main agent → multi-agent handoff) now stays inside one platform instead of crossing clouds, and one fewer deploy target, secret store, and dashboard to operate.

Everything else stayed. The backend kept running on Cloud Run, the frontend on Vercel, the database on Neon — each passed the capability-and-cost test, and migrating them would have added risk while buying nothing. That's worth underlining: **consolidation isn't about minimizing your vendor count.** It's about removing the layers that don't pay for themselves, and having the discipline to leave the rest alone.

The final piece was **one unified CI/CD pipeline in GitHub Actions** deploying everything — infrastructure via Terraform plan/apply gates, application components alongside. One pipeline, one deploy story, one place to look. It also unlocks the nice compositions this stack was always capable of, like per-PR preview environments backed by Neon's database branching.

## What I Learned About Building an Agentic Harness

Stepping back, the lessons that stick aren't really about any single vendor:

1. **Agentic apps punish platform sprawl harder than traditional apps.** The unit of debugging is a run that crosses everything, so every extra platform boundary is a place your visibility dies.
2. **Codify before you consolidate.** Terraform-importing a mess doesn't fix it, but it converts every future fix from a risky manual operation into a reviewable diff. Sequence matters: codify → observe → consolidate. Each phase de-risks the next.
3. **Choose runtimes by capability, not familiarity — and price the capability.** In an MVP, familiarity wins and that's fine. In a platform, every runtime must justify itself with something the others can't do, at a cost the workload actually justifies. Profile what your agents *actually do* before paying for what a platform *can* do: our "heavy compute" layer turned out to be I/O-bound orchestration.
4. **Product analytics ≠ agent observability.** You need both, and they're different tools. Trace the run, not just the services.
5. **The harness *is* the product.** In an agentic application, the quality your users experience is bounded by the quality of the harness the agent runs in — its latency, its observability, its reliability. Platform engineering isn't overhead on the AI work; it *is* the AI work.

And maybe the meta-lesson: MVP sprawl is not a failure. It's a stage. The failure mode is pretending you're still in that stage after your product has left it.

---

*If you're staring at your own accidental multi-cloud agent architecture: start with Terraform imports, not migrations. Future you will thank present you at the first `terraform plan`.*
