---
name: agent-council
description: "Use when a task needs multi-perspective brainstorming or stress-testing. Two modes: collaborative (default — three subagents build on each other's ideas) and adversarial (subagents debate to find the strongest answer). Triggers: council, siege, swarm, multi-agent, debate, brainstorm, stress-test."
---

# Agent Council — Dual Mode

Dispatch 3 council subagents (`council-alpha`, `council-beta`, `council-gamma`) in parallel, then call `council-orchestrator` for the final output. Two modes: **collaborative** (default) for novel synthesis, **adversarial** for stress-testing.

**Core principle:** Three distinct seats with different model tiers (opus / sonnet / haiku) catch each other's blind spots. The mode determines whether they cooperate or compete.

## Step 0 — Triage (before dispatching anything)

### Complexity gate

If the task is trivial, single-step, or speed-sensitive, **answer directly — do not dispatch the council**. Treat as trivial unless the user explicitly asks for a council:

- arithmetic, factual lookups
- one-line rewrites, command syntax
- narrow yes/no questions
- tasks with one obvious path and no real tradeoff

Treat as council-worthy:

- competing approaches with real tradeoffs
- security or correctness-sensitive reviews
- architecture, research, design-space exploration
- problems where multi-perspective synthesis is likely to beat one model

### Mode detection

**Adversarial triggers:** debate, adversarial, challenge, stress-test, which is better, argue, attack, defend, versus, vs

**Collaborative triggers (default):** council, siege, swarm, brainstorm, multi-agent, collaborate, explore, build on, novel, creative, ideas

**Explicit override wins:** `adversarial council: ...` or `collaborative council: ...`

If both trigger types appear, **adversarial wins** unless explicit override. If no trigger matches → collaborative.

### Domain detection

Classify into: **Code | Architecture | Research | Writing | General**. Resolution order: Code → Architecture → Writing → Research → General.

- **Code** — source code, APIs, tests, bugs, PRs, files, implementation details
- **Architecture** — system-level tradeoffs across services, data, scaling, deployment
- **Research** — investigation/evaluation as primary job (not building or reviewing a concrete artifact)
- **Writing** — drafting, documents, blog, copy, content
- **General** — anything else; omit domain focus

### Verbosity

- **Default:** Hide phases, show only final output
- **"verbose" / "show council":** Show concise phase-by-phase view
- **"raw" / "full":** Show full drafts instead of concise excerpts

## Seat assignments

| Seat | Subagent | Model | Strength |
|------|----------|-------|----------|
| Alpha | `council-alpha` | opus | depth and exploration |
| Beta | `council-beta` | sonnet | practical grounding, validation |
| Gamma | `council-gamma` | haiku | elegance, alternative angles |
| Orchestrator | `council-orchestrator` | opus | synthesis / verdict |

Each seat's persistent role and invariants live in its own subagent file. The skill's job is to inject the **per-phase task framing**.

---

## Collaborative Mode 🤝 (Default)

### Phase 1 — Draft (all 3 in parallel)

Send a **single message** with three `Agent` tool calls (one per seat). Each prompt:

```
Mode: collaborative — Draft phase
Domain: {domain}
Domain focus: {domain_focus_for_this_seat}

TASK:
{user_task}

Follow your persistent role for collaborative-draft. Stay under 1500 words.
```

Domain focus comes from the **Domain Adaptation** table below.

### Phase 2 — Improve (all 3 in parallel)

After all three return, send another single message with three `Agent` calls:

```
Mode: collaborative — Improve phase
Domain: {domain}

ORIGINAL TASK:
{user_task}

YOUR ORIGINAL DRAFT:
{this_seat_draft}

{OTHER_SEAT_1}'S DRAFT:
{other_1_draft}

{OTHER_SEAT_2}'S DRAFT:
{other_2_draft}

Steal the best ideas from the other drafts. Look for novel syntheses. Drop weak material. Keep your seat's natural strength. Add a brief ## Tensions section only if a real disagreement remains unresolved.
```

### Phase 3 — Synthesize

Single `Agent` call to `council-orchestrator`:

```
Mode: collaborative — Synthesis

ORIGINAL TASK:
{user_task}

ALPHA'S IMPROVED VERSION:
{alpha_improved}

BETA'S IMPROVED VERSION:
{beta_improved}

GAMMA'S IMPROVED VERSION:
{gamma_improved}

Author the definitive response. Identify best elements, surface emergent ideas, weigh by confidence ratings, resolve any flagged tensions with a clear call. Output the final synthesis only.
```

---

## Adversarial Mode 🗡️

### Phase 1 — Draft (all 3 in parallel)

Single message with three `Agent` calls:

```
Mode: adversarial — Draft phase
Domain: {domain}
Domain focus: {domain_focus_for_this_seat}

TASK:
{user_task}

Follow your persistent role for adversarial-draft. Stay under 1500 words.
```

### Phase 1.5 — Triage (you, no subagent)

Read all 3 drafts. Assess:

**Full consensus — skip Phase 2 — requires ALL of:**
1. All three recommend the same core approach
2. No CRITICAL or FATAL severity flags anywhere
3. No direct contradictions on key claims
4. Confidence ratings predominantly HIGH

**Leader-selection rubric (when consensus is not full):**
1. Correctness and evidence quality
2. Coverage of task's core constraints
3. Actionability and clarity
4. Severity / count of unresolved risks in critique sections
5. Confidence profile (reward precise confidence, not blanket HIGH)

Tie-breakers: fewer high-severity issues → fewer hidden assumptions → more operationally concrete.

**Outcomes:**
- **Full consensus** → skip to Phase 3 ("Consensus detected — no adversarial round needed")
- **Partial consensus** → forward to Phase 2 with focused attack instruction (see below)
- **No consensus** → forward leader to Phase 2 for open attack

### Phase 2 — Attack (2 non-leader seats in parallel)

Single message with two `Agent` calls (skip the leader):

```
Mode: adversarial — Attack phase
Domain: {domain}

ORIGINAL TASK:
{user_task}

YOUR ORIGINAL DRAFT:
{this_seat_draft}

LEADING POSITION (from {leader_seat}):
{leader_draft}

{partial_consensus_instruction}

Find every weakness, gap, wrong assumption, logical flaw. Where the leader contradicts your draft, argue why yours is better. Severity: FATAL / MAJOR / MINOR. End with STAND / MODIFIED / REJECTED.
```

`{partial_consensus_instruction}` — set by triage:
- **Partial consensus:** `"Focus your attack ONLY on these contested areas: {contested_points}. These points have consensus — do not relitigate: {agreed_points}."`
- **No consensus:** (omit)

### Phase 3 — Verdict

Single `Agent` call to `council-orchestrator`:

```
Mode: adversarial — Verdict

ORIGINAL TASK:
{user_task}

THE LEADING POSITION (from {leader_seat}):
{leader_draft}

ATTACK FROM {attacker_1}:
{attack_1}

ATTACK FROM {attacker_2}:
{attack_2}

ALL ORIGINAL DRAFTS (reference):
Alpha: {alpha_draft}
Beta: {beta_draft}
Gamma: {gamma_draft}

Evaluate each attack. Determine SURVIVED / MODIFIED / OVERTURNED. Build the final answer. Append a brief ## Confidence Assessment.
```

---

## Domain Adaptation

| Domain | Alpha focus | Beta focus | Gamma focus |
|--------|-------------|------------|-------------|
| **Code** | Implementation + security self-review | API accuracy, versions, edge cases | Performance, readability, alternative patterns |
| **Architecture** | System design + failure modes | Tech claims, benchmarks, scalability | Diagram clarity, simplicity, alternatives |
| **Research** | Comprehensive analysis + bias check | Source verification, citations, methodology | Readability, actionability, counter-arguments |
| **Writing** | Content + tone self-critique | Factual accuracy, consistency | Flow, conciseness, formatting |
| **General** | (omit — use seat default) | (omit) | (omit) |

## Verbose output formats

### Collaborative verbose

```
## 🤝 Council — Collaborative Mode

### Phase 1: Independent Explorations
💡 Alpha (Deep Explorer)
🔨 Beta (Practical Builder)
✨ Gamma (Elegant Minimalist)

### Phase 2: Cross-Pollinated Improvements
📝 Alpha (improved)
📝 Beta (improved)
📝 Gamma (improved)

### Phase 3: Final Synthesis
🌟 Orchestrated Synthesis
```

### Adversarial verbose

```
## 🗡️ Council — Adversarial Mode

### Phase 1: Independent Drafts
📝 Alpha (Drafter & Red Teamer)
✅ Beta (Fact-Checker & Validator)
🔧 Gamma (Optimizer & Devil's Advocate)

### Phase 2: Attack Round
🎯 Leading Position: {leader} — reason: {why}
⚔️ {attacker_1} attacks
⚔️ {attacker_2} attacks

### Phase 3: Verdict
🏛️ Status: SURVIVED | MODIFIED | OVERTURNED
   Confidence: HIGH | MEDIUM | CONTESTED
{final_answer}
```

By default show concise excerpts; show full drafts only if user explicitly asks for **raw** or **full** council output.

## Cost & speed

| Mode | Subagent calls | Parallel rounds |
|------|----------------|-----------------|
| Collaborative | 7 (3 + 3 + orchestrator) | 2 |
| Adversarial | 4–6 (3 + 0-2 attackers + orchestrator) | 2 |

For trivial tasks, **skip the council**.

## Common mistakes

- Dispatching for trivial tasks instead of short-circuiting
- Running seats sequentially instead of in parallel (one message, multiple `Agent` calls)
- Forcing disagreements when agents agree
- Skipping the orchestrator and concatenating raw drafts
- Mixing mode prompts (collaborative seats explore, adversarial seats critique — keep distinct)
- Skipping the improve round in collaborative mode (cross-pollination is the point)
- Skipping domain detection
- Ignoring confidence ratings when weighting inputs
- Treating partial consensus as no consensus — focus the attack on contested areas only
