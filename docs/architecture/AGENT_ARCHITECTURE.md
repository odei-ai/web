# ODEI: Cybernetic POMDP Agent OS

> **Version:** 1.0
> **Status:** Active Blueprint
> **Source:** GPT-5.2-Pro conversation synthesis + Anton review
> **Graph Node:** `405ca685-e329-8d6f-f9c0-db3c11de7f90` (Mind layer, Note)

---

## 0) One Sentence That Holds The Entire System

ODEI — кибернетический агент в частично наблюдаемой среде, который управляет вниманием/временем/энергией через ограниченное пространство действий, строгую конституцию полномочий, мониторинг исполнения и обучение на обратной связи — с доказуемыми ограничениями безопасности и аудируемостью.

**If any ODEI decision cannot be explained through:**

```
(observations → belief-state → goal/constraints → choice (ask/act) → action → verify effect)
```

**— then it's not an agent, it's chatter.**

---

## 1) Main Loop: ODEI "Nervous System"

This is the single cycle around which everything is built:

```
1. Sense      → perceive event (percept)
2. Update Belief → update belief-state (hypotheses + uncertainty)
3. Decide     → choose ask vs act vs do nothing (risk/authority/VoI)
4. Plan       → create/update plan (HTN)
5. Execute    → perform actions transactionally
6. Verify     → check effects against facts
7. Learn      → adjust thresholds/preferences from approve/deny/regret
8. Audit      → log everything (receipt + provenance)
```

**If you implement only this — you already have "agency".**

---

## 2) Three Architecture Planes

### A) Memory Plane (Memory and Truth)

Stores facts, hypotheses, relationships, history, sources. Capable of: conflicting, aging, revision.

**Components:**

- Facts (with provenance, timestamps, confidence, decay)
- Beliefs (hypotheses with probability, evidence, cost_of_being_wrong)
- Intents (user intention candidates with confidence, confirmation timestamps)

### B) Control Plane (Decision and Management)

Holds: goals/utility, constraints, authority, risk gates, stability, intervention policies.

**Components:**

- Authority Matrix (domain × action type × impact → authority level)
- Constraints (hard limits that cannot be violated)
- Risk Gates (confidence × impact × reversibility → ask/act decision)
- Stability Controls (budgets, cooldowns, hysteresis, deadband)

### C) Execution Plane (Action)

Holds: action registry, transactions, repeatability, rollbacks, effect verification, limits.

**Components:**

- Action Registry (every action with preconditions, effects, verify, risk profile)
- Transaction Protocol (prepare → execute → verify → commit/rollback)
- Rate Limits and Cooldowns

---

## 3) Invariants (Iron Laws of ODEI)

**If you violate these — you build a system that cannot be trusted.**

| #   | Invariant                               | Meaning                                                                                  |
| --- | --------------------------------------- | ---------------------------------------------------------------------------------------- |
| 1   | No action outside Action Registry       | Any "initiative" is a call to registered action with declared risk/preconditions/effects |
| 2   | No fact without provenance              | Any statement in memory has source, update date, confidence                              |
| 3   | No auto-execute without authority check | Authority is config, not "model boldness"                                                |
| 4   | No plan without monitoring              | Any plan has triggers for replanning                                                     |
| 5   | No execution without verify             | "Done" counts only if effect confirmed in state                                          |
| 6   | No learning without explicit signal     | Preference model updates on approve/deny/regret, not guessing                            |
| 7   | No stability → no autonomy              | Any autonomy must have budgets/cooldowns/hysteresis                                      |

---

## 4) Data Core: What ODEI Stores

### 4.1 Fact (Atom of Truth)

Each fact is an object with lifecycle:

```yaml
fact:
  fact_id: string
  claim: 'meeting with X tomorrow 14:00'
  source: calendar | manual | email | sensor | inferred
  timestamp_observed: datetime
  timestamp_updated: datetime
  confidence: 0..1
  freshness_decay: rule
  conflicts: [fact_id references]
  tags: [domain, project, person]
```

### 4.2 Belief (Hypothesis)

Belief is not fact, but "model of world":

```yaml
belief:
  hypothesis: 'probably today energy is low'
  probability: 0..1
  evidence: [fact_ids]
  cost_of_being_wrong: high | medium | low
  update_rule: bayesian | heuristic
```

### 4.3 Intent (User Intention)

Most dangerous place. ODEI must acknowledge uncertainty.

```yaml
intent:
  intent_candidate: 'user wants to focus on project X'
  confidence: 0..1
  last_confirmed_at: datetime
  evidence: [fact_ids]
```

**Rule:** Intent without confirmation degrades over time, agent asks more often.

---

## 5) Action Space: What ODEI Is Allowed To Do

You're not building "universal agent". You're building **limited machine of levers**.

### Action Types

| Type                | Description                                    | Examples                      |
| ------------------- | ---------------------------------------------- | ----------------------------- |
| informational       | Show context, summary, options                 | Morning brief, status report  |
| planning            | Create plan, decompose tasks, set spotlight    | Week planning, task breakdown |
| coordination        | Prepare draft message (not send)               | Email draft, meeting notes    |
| execution low-risk  | Create task, set reminder, block slot          | Todo creation, calendar block |
| execution high-risk | Messages on your behalf, money, public actions | **ONLY with approve**         |

### Action Registry (Required)

```yaml
actions:
  - id: create_task
    domain: work
    risk:
      impact: low
      reversibility: high
      reputation: low
    authority_required: auto_ok
    preconditions:
      - user_identity_known
    effects:
      - task_exists_in_system
    verify:
      - query_task_exists
    cooldown: '5m'

  - id: reschedule_meeting
    domain: social
    risk:
      impact: high
      reversibility: medium
      reputation: high
    authority_required: explicit_approve
    preconditions:
      - meeting_has_owner
      - reschedule_window_allowed
    effects:
      - calendar_updated
    verify:
      - query_calendar_updated
    cooldown: '24h'
```

---

## 6) Authority Constitution: Mixed-Initiative Is Configured, Not Discussed

This is not "character". This is a table of laws.

### Axes

- **domain:** work / health / family / admin / finance
- **action type:** informational / planning / coordination / execution
- **impact:** low / medium / high
- **reversibility:** high / medium / low
- **on whose behalf:** system vs user

### Authority Levels

```yaml
authority_levels:
  silent: 0 # Don't even mention
  suggest: 1 # Propose, user decides
  one_tap: 2 # Single confirmation
  explicit: 3 # Detailed approval
  auto: 4 # Execute autonomously

matrix:
  work:
    create_task: auto
    propose_plan: auto
    publish_public: explicit
  social:
    draft_message: one_tap
    send_message: explicit
    reschedule_meeting: explicit
  finance:
    categorize_spending: one_tap
    move_money: explicit
```

**Key Principle:** Higher impact + lower reversibility → lower autonomy.

---

## 7) Decision "Ask vs Act" — This Is Risk Math, Not Mood

### 7.1 Three Scalars

- **confidence** (in state/intent)
- **impact** (harm of error)
- **reversibility** (cost of rollback)

### 7.2 Gates

| Decision       | When                                                                 |
| -------------- | -------------------------------------------------------------------- |
| **Act**        | high confidence + low impact + high reversibility + authority allows |
| **Ask**        | low confidence OR low reversibility OR authority requires explicit   |
| **Do nothing** | low impact AND no value of intervention (anti-spam)                  |

### 7.3 Value of Information (Question as Action)

Question is asked only if:

```
VoI > Cost(question)
```

Where `Cost(question)` = annoyance / attention / context switch.

**Practically:**

- If question doesn't change action → stay silent
- If question changes action AND risk is high → ask

---

## 8) Planning: HTN + Monitoring + Replanning

### 8.1 Why HTN (Hierarchical Task Network)

Life isn't solved by optimal search. It's solved by hierarchy:

```
Goal → Projects → Stages → Steps → Actions
```

### 8.2 Each Step Must Have

- **preconditions** (what must be true)
- **effects** (what becomes true)
- **success_criteria** (how we know step is done)
- **timeout**
- **rollback** (if applicable)

### 8.3 Execution Monitoring (Most Important)

ODEI doesn't "execute plan". It watches preconditions and effects.

**Replanning Triggers:**

- Events (meeting appeared)
- Time delay
- Energy drop
- Constraint conflict (sleep/windows)
- Blocker appeared

---

## 9) Cybernetics: Stability To Not Become Noise

**If you don't build stability layer, you'll turn off ODEI in a week.**

### Required Mechanisms

```yaml
stability:
  budgets:
    interventions_per_day: 8
    high_risk_prompts_per_day: 3
  cooldowns:
    same_suggestion: '6h'
    replanning: '30m'
  deadband:
    schedule_slip_minutes: 20
  hysteresis:
    enter_brake_mode_energy: 0.35
    exit_brake_mode_energy: 0.55
```

---

## 10) Goodhart Defense: Protection From Proxy Optimization

ODEI will always try to "win" on your metrics. So you need:

### 10.1 Tripwires (Red Flags)

| Tripwire                            | Detection          | Action            |
| ----------------------------------- | ------------------ | ----------------- |
| tasks_done↑ but artifacts_shipped≈0 | busywork alarm     | Review task value |
| productivity↑ but sleep↓ and mood↓  | brake mode         | Reduce load       |
| context_switches↑                   | attention lockdown | Protect focus     |

### 10.2 Brake Actions

- Lower initiative
- Simplify plan
- Lock spotlight
- Propose recovery instead of "push through"

---

## 11) Option Preservation

**Rule:** With high uncertainty, avoid irreversible decisions.

**Technically:**

- Each action has `irreversibility_score`
- Higher score → more "ask", higher confidence requirements

---

## 12) Out-of-the-Loop Prevention

**ODEI must not make you weaker.**

### Built-in Modes

| Mode                     | Description                                    |
| ------------------------ | ---------------------------------------------- |
| **Coach mode**           | Explains and proposes, you decide              |
| **Doer mode (low-risk)** | Does routine within rules                      |
| **Manual days**          | 1 day/week — no auto-actions, only suggestions |

This is not UX whim. This is protection of your agency.

---

## 13) Reliability and Safety

**Without this, autonomy is forbidden.**

### 13.1 Transaction Protocol

```
prepare → execute → verify → commit/rollback
```

### 13.2 Idempotency

Repeated action call must not break world.

### 13.3 Least Privilege

ODEI must have minimum rights.
Especially: "write to people" and "move money" — only approve.

---

## 14) Receipts + Audit: Trust Built on Explainability

Every ODEI decision must generate "receipt":

```json
{
  "decision_id": "d_2025_12_13_0930",
  "mode": "Doer_low_risk",
  "trigger": "calendar_event_added",
  "beliefs_used": [{ "hypothesis": "today_is_meeting_heavy", "p": 0.78 }],
  "facts_used": [{ "fact": "meeting at 14:00", "source": "calendar", "ts": "2025-12-13T08:02:00Z" }],
  "constraints_checked": ["sleep_protected", "no_messages_without_approve"],
  "authority_check": "auto_ok",
  "action_taken": "create_task",
  "expected_outcome": "task exists and shown in today's plan",
  "verify_plan": "query_task_exists",
  "confidence": 0.82,
  "impact": "low",
  "reversibility": "high"
}
```

**If receipt doesn't exist — decision doesn't exist.**

---

## 15) Implementation Path: 9 Steps

| Step | Name                        | Duration | Definition of Done                                                 |
| ---- | --------------------------- | -------- | ------------------------------------------------------------------ |
| 1    | **Skeleton**                | 2 days   | event_bus + state storage + audit_log                              |
| 2    | **Action Registry**         | 1-2 days | All actions through registry with risk/preconditions/verify        |
| 3    | **Authority + Constraints** | 1-2 days | High-risk blocked without approve, hard constraints never violated |
| 4    | **Ask/Act Gate**            | 1-2 days | Ask only when changes action or reduces risk                       |
| 5    | **HTN Planning v1**         | 2-4 days | Goals decompose to stages, steps have success criteria             |
| 6    | **Monitoring + Replan**     | 2-4 days | After disruption, minimal replan (not "shake day every 5 min")     |
| 7    | **Stability Layer**         | 1-2 days | Interventions limited, same suggestions don't repeat               |
| 8    | **Feedback + Learning**     | 2-5 days | Accept rate grows, overrides fall                                  |
| 9    | **Evaluation Harness**      | 2-5 days | 20 scenarios run "in shadow", log shows why asked/acted            |

---

## 16) Minimum "Ideal Version"

**If you want to call ODEI an agent, you must have:**

- [ ] Action Registry
- [ ] Authority Matrix
- [ ] Constraints
- [ ] Ask/Act gate (confidence + impact + reversibility + VoI)
- [ ] Planning (HTN)
- [ ] Monitoring + Replan
- [ ] Stability controls
- [ ] Receipts + provenance
- [ ] Feedback learning loop
- [ ] Evaluation harness

**Without this, "ODEI" is an interface of ideas. Not an operating system.**

---

## Current State vs Blueprint

| Component         | Blueprint                      | Current ODEI                            | Gap                                    |
| ----------------- | ------------------------------ | --------------------------------------- | -------------------------------------- |
| Memory Plane      | Facts + Beliefs + Intents      | Neo4j nodes with provenance             | ⚠️ No Belief/Intent types, no decay    |
| Control Plane     | Authority matrix, constraints  | Guardrails, preflight.validate          | ⚠️ No formal YAML matrix               |
| Execution Plane   | Action Registry                | Implicit in MCP tools                   | ❌ No registry, no verify              |
| Main Loop         | 8 steps                        | Partial (Sense, Decide, Execute, Audit) | ❌ Belief, Plan, Verify, Learn missing |
| Stability         | Budgets, cooldowns, hysteresis | None                                    | ❌                                     |
| Feedback Learning | approve/deny/regret → tuning   | None                                    | ❌                                     |

---

## Implementation Approach

**NOT:** "Generate everything from prompt"
**YES:** "Step by step together"

1. Each step: AI proposes → Human reviews → iterate → commit
2. Context 200k = decomposition required, not monolith
3. Verify each step before next
4. Reference this document, don't auto-generate from it

---

_Last updated: 2025-12-13_
_Graph sync: Note `405ca685-e329-8d6f-f9c0-db3c11de7f90`_
