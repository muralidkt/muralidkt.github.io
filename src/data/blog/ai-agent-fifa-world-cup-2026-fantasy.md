---
title: "I Let an AI Agent Run My FIFA World Cup 2026 Fantasy Squad"
author: Murali Kotharamban
pubDatetime: 2026-06-06T00:00:00.000Z
slug: ai-agent-fifa-world-cup-2026-fantasy
featured: true
draft: false
tags:
  - AI
  - GenAI
  - Python
  - LangGraph
  - Operations Research
  - Side Project
description: "The World Cup starts next week. 736 players. 48 teams. 8 matchdays. So I built an AI agent that uses Poisson models, integer linear programming, and a bounded Claude layer to pick the mathematically optimal fantasy squad."
---

*The World Cup starts next week. 736 players. 48 teams. 8 matchdays. So I did what any reasonable engineer would do: I built an agent to think for me.*

---

## The Problem (Or: Why Fantasy Football Is Secretly an Optimization Problem)

Here's the thing about FIFA World Cup Fantasy that most people don't think about: your perfect team on Matchday 1 is probably terrible on Matchday 2. The opponent changes every round. That goalkeeper who kept a clean sheet against Saudi Arabia? He's facing France next. Your budget striker who scored a hat-trick? He's up against the best defence in the tournament.

Every single matchday is a fresh optimization problem with shifting constraints:

- **$100M budget** (rising to $105M in knockouts)
- **2 GKs, 5 DEFs, 5 MIDs, 3 FWDs** — exactly. No negotiation.
- **Max 3 players from one nation** in group stages (scales up as teams get eliminated)
- **Valid formations only** — 3-4-3, 4-4-2, 5-3-2, etc.
- **Limited free transfers** — take a hit beyond that, and it costs you -3 points per move

Most people eyeball this. Pick the big names, hope for the best. I wanted math.

## The Architecture: No Magic, Just Good Engineering

Let me be upfront: this is **not** a chatbot that vibes its way to a team. There's no "Hey Claude, who should I pick?" loop. The intelligence here is operations research — a transparent model you can inspect, question, and override.

Here's the pipeline:

```
FIFA public JSON ──┐
(prices, fixtures, ├─► team strength ─► opponent model ─► expected points ─► ILP optimizer ─► report
 ownership, form)  │   (per nation)     (goals / clean    (per player,        (legal 15 + XI
Polymarket / Elo  ─┘                     sheet, Poisson)    per matchday)        + captain)
SoFIFA ratings ──────────────────────────────┘ (quality prior)        │
manual overrides ────────────────────────────┘ (start risk/news)      │
                                                              optional Claude layer
                                                              (bounded score adjustments)
```

Every piece earns its place. Let me walk you through it.

## Step 1: The Data (It's Just... There?)

FIFA publishes everything you need via public JSON endpoints. No auth. No API key. Just three URLs:

- `squads.json` — all 48 teams and their players
- `rounds.json` — 8 matchdays with every fixture (so you know *who plays whom*)
- `players.json` — prices, positions, ownership %, form, and (after games start) official points

```bash
fantasy refresh   # pulls all three, caches locally
```

That's it. Your entire dataset, refreshed in under a second.

## Step 2: Who's Actually Good? (Team Strength)

Not all opponents are created equal. Conceding against Brazil is more likely than conceding against, well, let's not name names. The agent rates each team's strength using one or more signals:

- **Default:** sum of player values in the squad (crude but surprisingly predictive)
- **`--odds`:** Polymarket implied probabilities for "reach the Quarterfinals" (free, live, forward-looking — and it correctly rates cohesive South American sides that the price-sum undervalues)
- **`--elo`:** World Football Elo ratings (the gold standard for national team strength)

These get normalised and merged. The result: a single strength score per nation that feeds into...

## Step 3: The Opponent Model (Poisson Does the Heavy Lifting)

This is where it gets fun. A Poisson model takes the strength gap between two teams in a specific fixture and converts it into:

- **Expected goals for** each side
- **Expected goals against**
- **Clean sheet probability**

Why Poisson? Because goals in football genuinely follow a Poisson distribution — it's one of those rare cases where the textbook model actually fits reality. A team expected to score 1.5 goals has ~22% chance of scoring 0, ~33% chance of 1, ~25% chance of 2, and so on. Clean sheet probability drops straight out of P(X=0).

The key insight: **because the opponent changes every matchday, the projections change too.** Your Matchday 1 squad might need 4-5 transfers for Matchday 2, even if everyone played well.

## Step 4: Expected Points (The Money Model)

For each of the 736 players, every matchday, the agent computes an expected fantasy score by blending:

1. **Player quality** — price percentile within position (a $10M striker is probably better than a $5M one), in-game form once matches start, and optionally SoFIFA ratings
2. **Start probability** — a heuristic based on price rank within their national squad and position. The 4th-choice centre-back on a team probably isn't starting.
3. **Opponent expectation** — the Poisson output. Defenders facing weak attacks get clean sheet bonuses; attackers facing weak defences get goal bonuses.
4. **Manual overrides** — because sometimes you *know* a player is injured, rested, or playing out of position. A `data/overrides.yaml` file lets you set start probabilities, score multipliers, or captain avoidance flags.

The output: one number per player per matchday. Their expected fantasy points given *that specific fixture*.

## Step 5: The Optimizer (Where ILP Earns Its Keep)

This is the heart of the system. You can't just sort by expected points and pick the top 15 — you'd end up with 8 forwards and blow the budget. It's a **constrained optimization problem**, and the right tool is Integer Linear Programming.

Using PuLP (with the CBC solver), the optimizer finds the mathematically best legal squad:

```python
# The objective: maximize starting XI points (+ a small bench bonus)
maximize: sum(xpts[p] * starts[p] for p in players) + bench_weight * sum(xpts[p] * bench[p] for p in players)

# Subject to:
# - Exactly 15 players in squad
# - Exactly 2 GK, 5 DEF, 5 MID, 3 FWD
# - Starting XI = 11 (valid formation: 3-5 DEF, 3-5 MID, 1-3 FWD)
# - Total cost ≤ budget
# - Max N players per nation
# - For transfers: respect free allowance, -3 per extra hit
```

It's not an approximation. It's the **provably optimal** solution given the projections. Every legal squad in existence is in the feasible set — the solver finds the best one.

Captain selection is handled separately with a reliability-aware score:

```
captain_score = xPts * (weight + (1 - weight) * start_prob)
```

Because the worst thing in Fantasy is captaining someone who gets benched.

## The Claude Layer (Optional, Bounded, Honest)

Here's where the LLM comes in — but on a tight leash. When you run with `--llm`, Claude can read team news (injury updates, rotation hints, press conferences) and suggest **score multipliers** for flagged players.

Crucially:
- It never picks a team directly
- It never bypasses the optimizer
- Its adjustments are bounded (no wild 10x multipliers)
- It operates on a stable base (no compounding across passes)
- It's capped by `--max-research` iterations

The LLM is an *adjustment layer*, not a decision-maker. The math stays in charge.

## The LangGraph Orchestration (Why a State Machine?)

For interactive use, the whole pipeline runs as a LangGraph state machine:

```
START → ensure_data → assess_situation → analyze → risk_check ──┐
                                                    ↓            │
                                              synthesize    research (bounded loop)
                                                    ↓
                                            human_approval → persist → END
```

Why LangGraph? Because this flow has real branching (is the round fully drawn?), a real cycle (the research subloop), and a human-in-the-loop approval gate. The deterministic path is plain function calls — no framework needed. LangGraph earns its place only where orchestration logic actually exists.

```bash
fantasy run --round 1                          # build with approval gate
fantasy run --round 2 --advise --llm --news team_news.txt   # full agent mode
```

## The "Recommend-Only" Design (And Why That's a Feature)

FIFA has no write API. And even if they did, automating team submission violates their ToS — that's an account ban waiting to happen.

So the agent computes the perfect team and **you enter it manually**. Takes about 60 seconds.

This captures ~95% of the value. The analysis is the hard part. Clicking 15 names in an app is not. And you keep your account intact.

## Sample Output

```
$ fantasy build --odds --ratings

strength: Polymarket implied 'reach Quarterfinals' odds
SoFIFA ratings: 21 loaded, matched 21 players
projection overrides: 3 loaded from data/overrides.yaml
╭──────────────────────────────────────────────────────────────────────────────────╮
│ Recommended squad · Matchday 1 · formation 3-4-3                                 │
│ cost $99.9m · projected 93.9 pts (with captain)                                  │
│ Deadline: 2026-06-11 20:00:00+01:00 · enter this team at play.fifa.com/fantasy   │
╰──────────────────────────────────────────────────────────────────────────────────╯
Starting XI
┏━━━━━┳━━━━━━━━━━━━━━━━━━━┳━━━━━━━━━━━┳━━━━━━━━━━━━━━┳━━━━━━━━┳━━━━━━┳━━━━━━━┳━━━━━┓
┃ Pos ┃ Player            ┃ Team      ┃ vs           ┃ Price  ┃ xPts ┃ Start ┃     ┃
┡━━━━━╇━━━━━━━━━━━━━━━━━━━╇━━━━━━━━━━━╇━━━━━━━━━━━━━━╇━━━━━━━━╇━━━━━━╇━━━━━━━╇━━━━━┩
│ GK  │ Emiliano Martínez │ Argentina │ Algeria      │ $5.0m  │ 5.49 │ 100%  │     │
│ DEF │ Antonio Rüdiger   │ Germany   │ Curaçao      │ $5.5m  │ 7.89 │ 100%  │     │
│ DEF │ Joshua Kimmich    │ Germany   │ Curaçao      │ $5.5m  │ 7.58 │ 88%   │     │
│ DEF │ Johan Vásquez     │ Mexico    │ South Africa │ $4.7m  │ 6.65 │ 100%  │     │
│ MID │ Pedri             │ Spain     │ Cabo Verde   │ $8.1m  │ 8.85 │ 92%   │ (V) │
│ MID │ Lamine Yamal      │ Spain     │ Cabo Verde   │ $10.0m │ 8.54 │ 100%  │     │
│ MID │ Florian Wirtz     │ Germany   │ Curaçao      │ $7.5m  │ 8.12 │ 92%   │     │
│ MID │ Enzo Fernández    │ Argentina │ Algeria      │ $7.5m  │ 6.93 │ 100%  │     │
│ FWD │ Lionel Messi      │ Argentina │ Algeria      │ $10.0m │ 8.75 │ 100%  │ (C) │
│ FWD │ Ferran Torres     │ Spain     │ Cabo Verde   │ $7.8m  │ 8.74 │ 72%   │     │
│ FWD │ Raúl Jiménez      │ Mexico    │ South Africa │ $7.0m  │ 7.59 │ 100%  │     │
└─────┴───────────────────┴───────────┴──────────────┴────────┴──────┴───────┴─────┘
Bench
┏━━━━━┳━━━━━━━━━━━━━━━━━┳━━━━━━━━━━┳━━━━━━━━━━━━━━┳━━━━━━━┳━━━━━━┳━━━━━━━┳━━┓
┃ Pos ┃ Player          ┃ Team     ┃ vs           ┃ Price ┃ xPts ┃ Start ┃  ┃
┡━━━━━╇━━━━━━━━━━━━━━━━━╇━━━━━━━━━━╇━━━━━━━━━━━━━━╇━━━━━━━╇━━━━━━╇━━━━━━━╇━━┩
│ GK  │ Guillermo Ochoa │ Mexico   │ South Africa │ $4.2m │ 5.38 │ 100%  │  │
│ DEF │ Thomas Meunier  │ Belgium  │ Egypt        │ $4.8m │ 6.42 │ 100%  │  │
│ DEF │ Daniel Muñoz    │ Colombia │ Uzbekistan   │ $4.6m │ 6.34 │ 100%  │  │
│ MID │ Martin Ødegaard │ Norway   │ Iraq         │ $7.7m │ 6.46 │ 100%  │  │
└─────┴─────────────────┴──────────┴──────────────┴───────┴──────┴───────┴──┘
```

## What I Learned Building This

1. **The "AI Agent" label is overused.** Most of this system is deterministic math. The LLM is 5% of the value. The other 95% is data modeling and constrained optimization. Call it what it is.

2. **Poisson models for football are shockingly good.** Textbook statistics, applied correctly, beats most people's gut feeling. The hard part isn't the model — it's getting the inputs right.

3. **ILP solvers are criminally underused in hobby projects.** PuLP + CBC is free, fast, and gives you *provably optimal* solutions. Any time you're picking N things from M options with constraints, consider ILP before writing greedy heuristics.

4. **"Recommend-only" is a legitimate design pattern.** Not every agent needs to *do* the thing. Sometimes the highest-value output is "here's exactly what to do and why" — and the human does the clicking.

5. **LangGraph earns its place only when you have real control flow.** If your pipeline is a straight line, just call functions. Frameworks should solve actual orchestration problems, not add ceremony.

## Tech Stack

- **Python** — because data + math + CLI
- **PuLP/CBC** — integer linear programming (the optimizer)
- **Poisson model** — opponent strength → goal/clean sheet expectations
- **LangGraph** — state machine orchestration (optional `fantasy run` mode)
- **Claude** (Anthropic) — bounded research subloop for team news (optional)
- **Polymarket API** — live tournament odds as a strength signal
- **Click** — CLI framework

## What's Next

- **Dixon-Coles model** — a more sophisticated opponent model that handles home advantage and low-scoring draws better than vanilla Poisson
- **FBref form ingestion** — pre-tournament club form as a signal (via soccerdata)
- **Predicted lineup signals** — from reliable public sources
- **Backtest harness** — once matchdays are played, grade the agent's picks against reality

## Try It Yourself

The World Cup deadline is **Thursday, June 11 at 20:00 UK time**. If you want to try it:

```bash
git clone https://github.com/muralidkt/fifa-fantasy-agent
cd fifa-fantasy-agent
python -m venv .venv && source .venv/bin/activate
pip install -e ".[dev]"
fantasy refresh && fantasy build --save --odds
```

Your mathematically optimal squad, in about 10 seconds.

---

*The code is open source: [github.com/muralidkt/fifa-fantasy-agent](https://github.com/muralidkt/fifa-fantasy-agent)*

*Built the night before the deadline, like all good side projects.*
