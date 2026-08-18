---
title: "From API Tokens to AWS Bedrock: My Real-World Journey Configuring an AI Agent"
author: Murali Kotharamban
pubDatetime: 2026-04-12T20:00:00.000Z
slug: from-api-tokens-to-bedrock
featured: true
draft: false
tags:
  - AI
  - AWS
  - Bedrock
  - LLM
  - Platform Engineering
  - GenAI
  - Agents
  - LLM Gateways
  - Model Routing
description: "I set up a personal AI agent and spent weeks fighting cost overruns, local model disappointments, and AWS throttling. Here's the real story — every wrong turn, every fix, and the multi-model architecture that finally worked."
---

I run a personal AI agent on a server. Not a demo. Not a weekend project I spun up and forgot. An actual always-on agent that monitors my calendar, sends me daily AI briefings, helps me write code, and handles tasks I throw at it via Telegram.

Getting the model configuration right took weeks of trial, error, and real money. This is that story.

---

## Phase 1: Anthropic API — Clean Start, Harsh Reality

The obvious starting point: grab an Anthropic API key, point OpenClaw at it, done.

It worked. Claude Opus was impressive. The agent was capable and responsive. And then the bill arrived.

Anthropic's API pricing on the direct plan is not cheap for always-on agent use. The problem isn't a single conversation — it's the architecture. An agent that runs heartbeat checks every 30 minutes, fires scheduled tasks, handles multi-turn conversations, and loads workspace context files into every session accumulates tokens fast. Really fast.

Daily scheduled tasks alone — each one spinning up a session, loading context, making multiple LLM calls — were burning through my monthly budget in days.

I hit my quota. The API started rejecting requests. Time to rethink.

> **Lesson 1:** Direct API pricing is fine for apps with predictable, bounded usage. For always-on agents with ambient workloads, the math breaks down quickly.

---

## Phase 2: Go Cheap — Hetzner + Local Models

The obvious next move for a platform engineer: run it yourself. Hetzner offers cheap, decent hardware in Europe. Spin up a box, run Ollama, use open-weight models. Zero per-token cost.

I tried Qwen 3 first. Then Gemma 4 when it dropped (the "workstation" model, pitched as capable enough to run locally). Both are genuinely impressive models for their size. I was not happy with either.

Here's the real problem with local models for agent orchestration:

**They're good. They're not good enough for this.**

The main agent isn't just answering questions. It's:
- Reading 10+ workspace files on every session startup
- Making tool calls, parsing results, chaining reasoning
- Managing long multi-turn conversations with full context
- Orchestrating sub-agents and interpreting their outputs
- Writing and editing code with real correctness requirements

Local models at 7B-14B struggle with sustained multi-step reasoning. They drift. They hallucinate tool call syntax. They lose track of context mid-conversation. The smaller models that could run at acceptable speed on affordable hardware weren't reliable enough. The models that were reliable enough required hardware I wasn't willing to pay for on Hetzner.

I used local models for sub-tasks and scheduled jobs — they handled those fine. But I kept pointing the main agent back at Opus 4.6 because nothing else held up for orchestration.

> **Lesson 2:** Local open-weight models are not a drop-in replacement for frontier models in agentic orchestration. They're great for bounded, well-defined tasks. For the main reasoning loop, you pay the frontier tax or you feel it.

---

## Phase 3: AWS Bedrock — The Right Tool for Each Job

That's when I decided to try AWS Bedrock hosted models. I already had an AWS account, Bedrock access was straightforward to enable, and the EU region had all the models I needed. The bonus: running on EC2 with an IAM role means clean auth — no API keys in config files, no credential rotation headaches.

The insight that changed everything: **not every task needs the same model**.

Here's the architecture I landed on:

### Scheduled Tasks & Cron Jobs → Amazon Nova Lite

Nova Lite is extraordinarily cheap. It's fast. It handles well-structured, bounded tasks reliably — summarize this, format that, check this condition.

My daily 6 AM AI briefing? Nova Lite. Heartbeat checks? Nova Lite. Any cron that just needs to process structured data and produce structured output.

Cost: nearly zero.

### Sub-Agents & Parallel Tasks → Claude Sonnet 4.5 / 4.6

Sub-agents handle focused, isolated work — implementing a feature, writing a blog post, analysing a codebase, running a GitHub issue fix. They need to be capable but they don't need to be Opus.

Sonnet 4.6 is an excellent model. It's fast, it's capable, and it costs a fraction of Opus. For the majority of actual work that gets done, Sonnet is the right call.

### Main Agent & Orchestration → Claude Opus 4.6

Opus handles the main conversation, context management, tool orchestration, and any task that requires sustained multi-step reasoning across a long session. It's the most expensive tier, but it's what actually needs to be there.

The key is that Opus only runs when I'm actively in conversation. Cron jobs, heartbeats, sub-tasks — those never touch Opus.

> **Lesson 3:** Match model capability to task complexity. Paying Opus prices for a cron job that summarizes an RSS feed is waste. Using Nova Lite for your main reasoning loop is frustration. Fit the tier to the job.

---

## Phase 4: The Config Pain I Didn't Expect

Getting Bedrock working wasn't just "plug in AWS credentials." A few specific issues that cost me time:

### Bedrock Model Discovery Lies About Context Windows

AWS Bedrock's model discovery API returned `contextWindow: 32000` for every Claude model in my setup. Claude Opus 4.6 has a **200K context window**. With 32K reported, the compaction system was kicking in constantly — the system prompt + workspace files alone nearly maxed out the reported limit.

Fix: manually override the model context windows in `openclaw.json` config. The discovery-generated `models.json` gets overwritten on every gateway restart, so the override needs to live in the persistent config.

```json
"models": {
  "providers": {
    "amazon-bedrock": {
      "models": [
        {
          "id": "eu.anthropic.claude-opus-4-6-v1",
          "contextWindow": 200000,
          "maxTokens": 16384
        }
      ]
    }
  }
}
```

### Daily Token Quotas on Bedrock EU Region

AWS Bedrock has per-model daily token quotas (TPD limits) that vary by region and account tier. Hit the limit and you get:

```
Throttling error: Too many tokens per day, please wait before trying again.
```

Which surfaces to the user as "Something went wrong while processing your request."

Not obvious. Not well documented. Fix: request a quota increase in Service Quotas, and reduce unnecessary Opus usage by routing non-orchestration tasks to cheaper models (which you should be doing anyway — see Phase 3).

### IAM Auth Is Cleaner Than You Think

One thing that actually worked well from day one: IAM-based auth. No API keys in config files. The EC2 instance has an IAM role with Bedrock permissions, and the SDK picks it up automatically. Zero credential management.

```json
"providers": {
  "amazon-bedrock": {
    "auth": "aws-sdk",
    "api": "bedrock-converse-stream"
  }
}
```

---

## Where I Landed

Current setup, running stable:

| Workload | Model | Why |
|----------|-------|-----|
| Cron jobs, heartbeats | Nova Lite (eu) | Cheap, fast, reliable for structured tasks |
| Sub-agents, parallel tasks | Sonnet 4.6 (eu) | Capable, cost-effective, fast |
| Main agent, orchestration | Opus 4.6 (eu) | Best sustained reasoning, worth the cost |
| Fallback | Sonnet 4.6 (Anthropic direct) | When Bedrock throttles |

EU region throughout — data residency matters when you're running personal context through a model.

---

## What I'd Tell Someone Starting Today

1. **Don't start with direct Anthropic API for always-on agents.** Use Bedrock or another inference provider with more predictable pricing at scale.
2. **Local models are a complement, not a replacement.** Use them for bounded sub-tasks. Don't make them your orchestrator.
3. **Build the multi-tier model architecture from day one.** It's not premature optimisation — it's the right architecture.
4. **Watch for Bedrock model discovery bugs.** Check that reported context windows match the actual model specs.
5. **Request quota increases proactively.** Don't wait until you're throttled in production.

The agent works well now. The daily briefing runs at 6 AM, Telegram messages get intelligent responses, sub-agents handle complex tasks in parallel. It took longer than it should have to get here — but every wrong turn was a real lesson.

That's usually how it goes with infrastructure.

---

*Murali Kotharamban is a Senior Platform Engineer at Quantagonia GmbH, Munich, building AI inference platforms and agentic systems on AWS Bedrock.*
