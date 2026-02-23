# Constitutional AI Guardrails

**How ODEI validates every agent action before execution.**

## The Problem

AI agents can take harmful, unauthorized, or duplicate actions — especially when operating autonomously. Standard prompt-based safety fails in production. You need a structured validation layer that runs BEFORE the agent acts.

## ODEI's 7-Layer Solution

### Layer 1: Immutability
Checks if the target entity or resource is locked or immutable. Prevents agents from rewriting history, modifying completed transactions, or altering protected state.

### Layer 2: Temporal Context
Validates that the action is timely. Checks `createdAt`, `updatedAt`, and optional `expiresAt` timestamps on all referenced entities. Catches stale instructions from previous sessions.

### Layer 3: Referential Integrity
Verifies all referenced entities exist in the world model. Catches hallucinated references — where the LLM invents wallet addresses, user IDs, or project references that don't exist.

### Layer 4: Authority
Checks whether the requesting agent has the authority scope for this action. Validates against governance rules stored in the FOUNDATION layer.

### Layer 5: Deduplication
Uses content hashing to detect if this exact action has already been taken. Prevents double-sending messages, double-executing transactions, or creating duplicate entries.

### Layer 6: Provenance
Traces the instruction back to its origin. Verifies it came from a trusted principal, not injected via an untrusted input.

### Layer 7: Constitutional Alignment
Highest-level check. Compares the action against constitutional principles stored in the FOUNDATION layer — the things an agent must never do regardless of instructions.

## API Usage

```python
import requests

response = requests.post(
    "https://api.odei.ai/api/v2/guardrail/check",
    headers={"Authorization": "Bearer YOUR_TOKEN"},
    json={
        "action": "transfer 500 USDC to 0x8185ecd4170bE82c3eDC3504b05B3a8C88AFd129",
        "context": {"requester": "trading_agent_v2", "reason": "performance fee"},
        "severity": "high"
    }
)

result = response.json()
print(result["verdict"])  # "APPROVED", "REJECTED", or "ESCALATE"
print(result["reasoning"])  # Full explanation
```

## Via Virtuals ACP

```
Service: guardrail_check
Price: $0.10 per call
Agent: #3082 at app.virtuals.io/acp/agent-details/3082
```

## Production Results

From live deployment since January 2026:
- **65%** → APPROVED (routine operations)
- **15%** → REJECTED (duplicates, unauthorized, violations)
- **20%** → ESCALATE (edge cases requiring human review)

The ESCALATE category creates the most value — catching edge cases that simple rules would approve but humans should review.
