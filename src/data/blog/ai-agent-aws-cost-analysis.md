---
title: "How I Use AI Agents to Analyse Costs Across Large AWS Environments"
author: Murali Kotharamban
pubDatetime: 2026-04-10T00:00:00.000Z
slug: ai-agent-aws-cost-analysis
featured: true
draft: false
tags:
  - AWS
  - Cost Optimisation
  - AI
  - GenAI
  - Platform Engineering
  - FinOps
description: "Managing AWS costs across dozens of accounts is painful. Here's how I use AI agent skills to trace billing spikes to root cause, find waste, and recover thousands per month — without clicking through the console."
---

Managing AWS costs across a large number of accounts is one of those problems that sounds straightforward until you're actually doing it. A single account with a handful of services is manageable. Ten accounts across multiple teams and regions, each with their own workloads, tagging conventions, and billing quirks? That's where things get out of hand fast.

This post walks through how I use AI agent skills — specifically `aws-cost-analyser` and `aws-resource-analyser` from my [aws-agent-skills](https://github.com/muralidkt/aws-agent-skills) repo — to cut through that complexity.

---

## The Real Problem With Multi-Account Cost Management

Most teams reach for AWS Cost Explorer directly. It works, but it has friction:

- You're clicking through a UI, not running repeatable workflows
- Cross-account visibility requires a management/payer account and the right IAM setup
- Drilling down from "EC2-Other is expensive" to "this specific NAT Gateway in this specific account is the culprit" takes 15-20 minutes of manual investigation per account
- The same investigation next month starts from scratch

Multiply that by 20, 50, or 100 accounts and the math breaks down. You end up with a cost report that tells you the total, but not the story.

---

## What the Skills Do Differently

The `aws-cost-analyser` and `aws-resource-analyser` skills teach your AI agent the full investigation workflow — not just the API calls, but the reasoning: where to look next, what spikes mean, how to connect a billing line item to an actual resource.

You describe what you want in plain language. The agent runs the right CLI commands, interprets the output, and tells you what it found.

---

## A Real Investigation: $2,600 Spike Across Accounts

Here's a scenario that plays out regularly in teams managing large AWS environments.

Your consolidated bill jumped $2,600 month-over-month. No deployment happened. No new workloads spun up. Just a bill that's suddenly higher.

**Start with the payer account view:**

> *"Break down my AWS costs by linked account for last month vs the month before"*

```
Account                    Apr        Mar        Delta
---------------------------------------------------------
prod-platform (123456)     $2,840     $2,790     +$50
prod-data (234567)         $1,920     $680       +$1,240  ⚠️
staging (345678)           $890       $870       +$20
shared-services (456789)   $1,150     $460       +$690   ⚠️
```

Two accounts with significant spikes. Drill into each:

> *"Drill into account 234567 — what service is driving the cost increase?"*

```
Service         Apr      Mar      Delta
----------------------------------------
EC2-Other       $1,310   $180     +$1,130  🚨
RDS             $420     $400     +$20
S3              $190     $100     +$90
```

EC2-Other again. Always EC2-Other.

> *"What's inside EC2-Other for account 234567?"*

```
Usage Type               Apr      Mar      Delta
-------------------------------------------------
NatGateway-Bytes         $980     $60      +$920  🚨
NatGateway-Hours         $180     $90      +$90
DataTransfer-Out-Bytes   $150     $30      +$120
```

NAT Gateway data processing: $920 spike in one account.

**Now switch to resource analyser:**

> *"Find what's routing large volumes of traffic through the NAT Gateway in account 234567, eu-west-1"*

```
NAT Gateway traffic sources (last 30 days):
  10.0.12.45 → i-0a3f8b2c prod-data-pipeline  → 5.1 TB
  10.0.14.22 → EKS node group                 → 0.9 TB
```

`prod-data-pipeline` — a batch EC2 instance — processed 5.1 TB through NAT to reach S3. Fix: add an S3 VPC Gateway Endpoint (free) and the NAT processing charge drops to near zero.

> *"Now check account 456789 — same question, what's driving the $690 spike?"*

```
Service         Apr      Mar      Delta
----------------------------------------
EC2-Instances   $510     $110     +$400  ⚠️
ELB             $380     $200     +$180
```

> *"Find idle or unused resources in account 456789"*

```
STOPPED EC2 INSTANCES (still incurring EBS costs)
  i-0b7e9d1a  r5.2xlarge  us-east-1  ml-training-old  $280/mo (EBS only)
  i-0c8f2b3d  m5.xlarge   us-east-1  perf-test-jan    $60/mo  (EBS only)

UNUSED LOAD BALANCERS (0 healthy targets for 30+ days)
  app/old-api-alb     us-east-1  $18/mo
  app/legacy-internal us-east-1  $18/mo

UNATTACHED EBS VOLUMES
  vol-0d9e3c4f  1000 GB gp2  us-east-1  $100/mo
  vol-0e4f5a6b   500 GB gp2  us-east-1   $50/mo
```

The `ml-training-old` instance was stopped after a one-off training run but never terminated. The 1TB gp2 volume was from a data migration project six months ago. Classic accumulation of forgotten resources across a shared-services account.

---

## Full Savings Summary Across Both Accounts

```
ACCOUNT 234567 — prod-data
  Add S3 VPC Gateway Endpoint (eliminates NAT traffic)  → ~$920/mo
  -------------------------------------------------------
  Subtotal:                                               ~$920/mo

ACCOUNT 456789 — shared-services
  Terminate ml-training-old + detach EBS                → $280/mo
  Terminate perf-test-jan + detach EBS                  → $60/mo
  Delete orphaned 1TB + 500GB EBS volumes               → $150/mo
  Remove unused load balancers                          → $36/mo
  -------------------------------------------------------
  Subtotal:                                               $526/mo

TOTAL RECOVERABLE:                                        ~$1,446/mo
```

Found in one session. Across two accounts. Without logging into the AWS console once.

---

## Doing This at Scale

For teams with many accounts, the workflow extends naturally:

**Weekly cost review across all accounts:**
> *"Show me accounts where costs increased more than 10% week-over-week"*

**Monthly waste sweep:**
> *"Scan all accounts for unattached EBS volumes, unused load balancers, and stopped instances older than 30 days"*

**Tag compliance audit (critical for chargeback):**
> *"Which accounts have EC2 and RDS resources missing cost-centre or team tags?"*

**Anomaly investigation:**
> *"Account 789012 has a data transfer spike — trace where the traffic is going"*

The agent runs the AWS CLI commands, interprets the output, and surfaces what matters. You decide what to action.

---

## Getting Started

Both skills work with any AI coding agent that supports the Agent Skills format — Claude Code, Cursor, or any agent with `.agents/skills/` support.

```bash
git clone https://github.com/muralidkt/aws-agent-skills.git /tmp/aws-agent-skills

# Claude Code
cp -r /tmp/aws-agent-skills/skills/aws-cost-analyser ~/.claude/skills/
cp -r /tmp/aws-agent-skills/skills/aws-resource-analyser ~/.claude/skills/
```

Your AWS CLI needs to be configured with credentials that have Cost Explorer and read-only resource access. For multi-account use, run from your management/payer account or use AWS Organizations with appropriate cross-account roles.

Full documentation and the IAM skill (for access auditing across accounts) at: **[github.com/muralidkt/aws-agent-skills](https://github.com/muralidkt/aws-agent-skills)**

---

*Murali Kotharamban is a Senior Platform Engineer at Quantagonia GmbH, Munich, building AI inference platforms and agentic systems on AWS.*
