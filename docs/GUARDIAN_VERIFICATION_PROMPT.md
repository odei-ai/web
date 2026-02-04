# Guardian Proposal Gate Verification Prompt

**For: GPT-5.2-Codex**
**Purpose: Verify implementation of Actor Attestation and Guardian Proposal Gate**

---

## Context

ODEI implemented Guardian Proposal Gate to prevent autonomous agents from bypassing governance when writing to Neo4j. The key security feature is **Actor Attestation** - deriving effective actor from trusted caller context instead of user payload.

## Files Changed

Please verify these implementations:

### 1. Actor Attestation Module
**File:** `servers/odei-neo4j/src/guardian/attestation.ts`

**Verify:**
- [ ] `CallerOrigin` type includes: `agent_runtime`, `renderer`, `conductor`, `system`, `external_sync`
- [ ] `EffectiveActor` type includes: `human`, `agent`, `joint`, `system`, `external`
- [ ] `getCallerOrigin()` reads from `process.env.ODEI_CALLER_ORIGIN`, defaults to `agent_runtime` (most restrictive)
- [ ] `resolveEffectiveActor()` logic:
  - `agent_runtime` → ALWAYS returns `agent` regardless of payload (CRITICAL)
  - `renderer` → allows `human`/`joint` from payload, defaults to `human`
  - `conductor` → returns `agent` unless payload is `system`
  - `system` → returns `system`
  - `external_sync` → returns `external`
- [ ] `attestProvenance()` overrides `provenance.actor` with effective actor when payload is spoofed
- [ ] `isSudoAllowed()` only permits sudo from `renderer` with user present or `system` with valid token
- [ ] Agent runtime CANNOT use sudo even if payload claims `sudo: true`

### 2. Immutable Entity Protection
**File:** `servers/odei-neo4j/src/guardian/immutable.ts`

**Verify:**
- [ ] `IMMUTABLE_AUDIT_ENTITIES`: AuditLogEntry, GraphOp, ActionOp
- [ ] `IMMUTABLE_FACT_ENTITIES`: Decision, Action, Observation, Event, Signal
- [ ] `enforceImmutability()` blocks UPDATE/DELETE for audit entities
- [ ] Fact entities allow only whitelisted field updates (verified, metadata, tags)
- [ ] `requiresProposalForAgent()` correctly identifies policy/evidence bucket exemptions

### 3. Guardian Index Updates
**File:** `servers/odei-neo4j/src/guardian/index.ts`

**Verify:**
- [ ] Imports attestation and immutable modules
- [ ] `enforceNodeGuard()`:
  - Calls `buildCallerContext()` at start
  - Calls `attestProvenance()` with callerOrigin and payload actor
  - Logs warning if payload actor was overridden
  - Calls `enforceImmutability()` before writes
  - Passes `effectiveActor` to `enforceProposalGate()`
- [ ] `enforceProposalGate()`:
  - Checks `isSudoAllowed(callerContext)` before allowing sudo bypass
  - Throws `GUARD_SUDO_NOT_ALLOWED` if sudo from unauthorized origin
  - Allows human/joint actors to bypass proposal gate
  - Uses `requiresProposalForAgent()` for entity type check
  - Error messages include effective actor and caller origin for debugging
- [ ] `enforceRelationshipGuard()` has same attestation pattern

### 4. Error Codes
**File:** `servers/odei-neo4j/src/guardian/errors.ts`

**Verify:**
- [ ] `GUARD_SUDO_NOT_ALLOWED` error code added
- [ ] `GUARD_IMMUTABLE_AUDIT` error code added
- [ ] `GUARD_IMMUTABLE_FACT` error code added

### 5. MCP Configuration
**Files:** All `agents/*/.claude/mcp.json` and `.claude/mcp.json`

**Verify:**
- [ ] All agent mcp.json files have `"ODEI_CALLER_ORIGIN": "agent_runtime"` in odei-neo4j env
- [ ] Agents covered: commander, plan, execute, health, mind, finance, builder
- [ ] Root workspace mcp.json also has `"ODEI_CALLER_ORIGIN": "agent_runtime"`

---

## Security Test Cases

Please verify these attack scenarios are BLOCKED:

### Test 1: Agent Spoofing Actor
```
Caller: agent_runtime (from mcp.json env)
Payload: { provenance: { actor: 'joint' } }
Entity: Value (Foundation layer)

Expected: GUARD_PROPOSAL_REQUIRED
Reason: effective_actor resolves to 'agent' regardless of payload
```

### Test 2: Agent Sudo Bypass Attempt
```
Caller: agent_runtime
Payload: { provenance: { actor: 'agent', sudo: true } }
Entity: Goal (Vision layer)

Expected: GUARD_SUDO_NOT_ALLOWED
Reason: agent_runtime cannot use sudo
```

### Test 3: Agent Creating Audit Entity
```
Caller: agent_runtime
Payload: Create AuditLogEntry
Operation: CREATE

Expected: ALLOWED (audit entities allow CREATE from agents)
```

### Test 4: Agent Updating Audit Entity
```
Caller: agent_runtime
Payload: Update AuditLogEntry
Operation: UPDATE

Expected: GUARD_IMMUTABLE_AUDIT
Reason: Audit entities are append-only
```

### Test 5: Agent with Valid Proposal
```
Caller: agent_runtime
Payload: { provenance: { actor: 'agent', proposal_id: 'valid-approved-proposal' } }
Entity: Task (Execution layer)
Proposal Status: APPROVED

Expected: ALLOWED
Reason: Agent has approved proposal
```

### Test 6: Renderer Human Write
```
Caller: renderer (hypothetically set via env)
Payload: { provenance: { actor: 'human' } }
Entity: Value (Foundation layer)

Expected: ALLOWED
Reason: renderer allows human claims, human can write directly
```

### Test 7: Policy/Evidence Bucket Exemption
```
Caller: agent_runtime
Payload: { provenance: { actor: 'agent' } } // No proposal_id
Entity: Proposal (Policy bucket)

Expected: ALLOWED
Reason: Policy bucket doesn't require proposal for agents
```

---

## Critical Questions

1. **Is there any code path where `provenance.actor` from payload is used directly for governance decisions?**
   - Expected: NO - all governance decisions use `effectiveActor` from attestation

2. **Can an agent set `actor: 'human'` or `actor: 'joint'` and bypass proposal gate?**
   - Expected: NO - `agent_runtime` always resolves to `agent`

3. **Can an agent delete AuditLogEntry, GraphOp, or ActionOp?**
   - Expected: NO - GUARD_IMMUTABLE_AUDIT blocks all non-CREATE operations

4. **Is the default caller origin restrictive?**
   - Expected: YES - defaults to `agent_runtime` if ODEI_CALLER_ORIGIN not set

5. **Are all agent mcp.json files configured with agent_runtime?**
   - Expected: YES - 7 agents + root workspace

---

## Output Format

Provide:
1. **Pass/Fail** for each verification item
2. **Security assessment**: Any remaining attack vectors?
3. **Code quality**: Any issues with the implementation?
4. **Recommendations**: Further hardening needed?

---

## Files to Read

```
servers/odei-neo4j/src/guardian/attestation.ts
servers/odei-neo4j/src/guardian/immutable.ts
servers/odei-neo4j/src/guardian/index.ts
servers/odei-neo4j/src/guardian/errors.ts
agents/commander/.claude/mcp.json
agents/plan/.claude/mcp.json
agents/execute/.claude/mcp.json
agents/health/.claude/mcp.json
agents/mind/.claude/mcp.json
agents/finance/.claude/mcp.json
agents/builder/.claude/mcp.json
.claude/mcp.json
```
