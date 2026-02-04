# ODEI Subagent System (Current Architecture)

> Legacy note: this file kept the name `SKILLS.md` so existing references continue to work. The actual
> architecture is now **Agent → Subagents → MCP** with no `.claude/skills/*` directories.

Subagents are autonomous Task-tool workflows defined in `.claude/agents/{subagent}.md`. Each file contains YAML
front matter (name, description, tools, token budgets) plus deterministic instructions. When a trigger fires, the agent
invokes Claude Code's **Task** tool, attaches the relevant `.claude/agents` file via the `files` parameter, and passes a
succinct instruction block. The Task executes in an isolated mini-context, calls the listed MCP tools, and returns a
compressed summary for the agent to weave into its reply.

## Active Subagent Inventory

| Agent   | Subagent                   | Purpose                                                           | Typical Triggers                                            |
| ------- | -------------------------- | ----------------------------------------------------------------- | ----------------------------------------------------------- |
| Discuss | `sopliv-guard`             | Detect wishful thinking initiatives lacking a $100M critical path | "I want to…" goals, strategic pivots                        |
| Discuss | `evening-critic`           | Block or defer decisions made 20:00-02:00 Doha time              | Any constitutional/strategic edit during the evening window |
| Discuss | `constitutional-validator` | Score proposals against Values/Principles/Guardrails              | "Should we…?", new commitments, goal edits                  |
| Discuss | `alignment-auditor`        | Multi-horizon audit that proves proposals ladder up day → decade  | When alignment or workload impact is unclear                |
| Discuss | `emotional-sampler`        | Sample history threads to sense stress/energy before advising     | Long gaps, unclear tone, after intense sprints              |
| Plan    | `memory-loader`            | 15K-token strategic dossier (identity → workload)                 | Session start, on-demand context refresh                    |
| Plan    | `risk-assessor`            | Probability × impact matrix + mitigations                         | Every initiative before documentation                       |
| Plan    | `capacity-analyzer`        | Week/month/quarter workload + guardrail verdict                   | Any new scope / accelerated timeline                        |
| Plan    | `dependency-mapper`        | Build critical path + precursor map, detect SOPLIV                | After milestones drafted, before Execute handoff            |
| Plan    | `visual-brief`             | Generate deterministic Miro/Notion layout + annotations           | Before calling Miro APIs or sharing decks                   |
| Execute | `capacity-calculator`      | Compute utilization %, available hours, verdict                   | Before accepting new work / daily planning                  |
| Execute | `blocker-escalator`        | Classify stalled tasks and propose 3-option recoveries            | Session start, "any blockers?" prompts                      |
| Execute | `focus-window-guard`       | Match tasks to proven energy windows + guardrails                 | Scheduling deep work / rearranging calendar                 |
| Execute | `capacity-wall`            | Hard-stop gating once utilization >85% (requires trade-offs)      | New commitments that push workload into Red                 |
| Mind    | `anomaly-scanner`          | 2-sigma anomaly detection with severity + root causes             | Weekly scan, after temporal issues, "see anomalies"         |
| Mind    | `pattern-detector`         | 7/14/30-day behaviour pattern synthesis w/ confidence             | Weekly Mind turn, retro, "what are we learning"             |
| Mind    | `feedback-loop-builder`    | Package insights into owner+deadline+KPI loops                    | After insights confirmed, before handing off                |
| Mind    | `estimation-calibrator`    | Compare estimates vs actuals, produce multipliers/bias alerts     | Weekly or whenever estimation accuracy questioned           |

## Execution Model

1. **Task Invocation** – Agent calls `Task({ label, files: ['.claude/agents/<name>.md'], instructions })`. The referenced
   file provides the full methodology and allowed MCP tools.
2. **Input Discipline** – Keep the Task `instructions` tight (topic, lookback, constraints). Subagents expect curated
   briefs, not raw conversation logs.
3. **Isolated Context** – Each Task receives a fresh context window (usually 1–15K tokens). This prevents heavy data
   retrieval from polluting the main agent thread while still enabling deep analysis.
4. **Output Integration** – Agents never dump raw Task logs. They summarize key findings (scores, tables, verdicts) in
   their reply, mention which subagents were invoked, and store any structured records via the appropriate MCP tool.
5. **Audit Trail** – Because subagents are markdown files in-repo, reviews are simple (diffs + PRs). Changes automatically
   propagate the next time an agent reads the file inside a Task.

## Maintenance

1. **Add** a subagent by dropping `agents/{agent}/.claude/agents/{name}.md` with YAML front matter + instructions.
2. **Reference** the new subagent inside the agent’s `CLAUDE.md` roster (purpose, triggers, Task usage pattern).
3. **Retire** old behaviour by deleting the `.claude/agents/{name}.md` file and updating documentation.
4. **No `.claude/skills` directories remain.** If other docs reference them, treat that material as historical.

## Migration Notes

- The November 2025 skills experiment has been rolled back. Skills were optimized for inline checklists but created
  constant token overhead and manual loading requirements.
- Subagents restore the clean two-layer execution model described in `PRODUCTION_READY_VERIFICATION.md`:
  `Agent → Subagents → MCP`.
- Discuss still relies on guard subagents for constitutional protection; Plan/Execute/Mind now run exclusively through
  Task-based subagents.
- If a future UI needs to trigger subagents automatically, reuse these markdown definitions rather than reintroducing the
  SkillExecutor layer.
