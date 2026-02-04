# ODEI System Architecture

**ODEI (Open Digital Evolution Intelligence)** is a hybrid local-cloud operating system built for high-performance AI orchestration.

## The Stack

- **Core:** Electron (Chromium + Node.js)
- **Memory:** Neo4j (Graph Database) - *Local Instance*
- **Intelligence:** LLM Agnostic (supports local Llama 3 or cloud GPT/Claude via Gateway)
- **Telemetry:** Apple HealthKit Integration (via shortcut/API)

## Component View

### 1. The Kernel (Main Process)
Handles the OS-level integrations.
- **Biometric Monitor:** Listens for HRV/Heart Rate data.
- **Finance Guard:** Checks account balances via read-only APIs.
- **System Tray:** continuous presence.

### 2. The Cortex (Memory Graph)
A persistent knowledge graph that maps:
- **Entities:** People, Projects, Assets.
- **Relationships:** "Depends On", "Funded By", "Blocked By".
- **Temporal States:** "Past", "Now", "Future".

### 3. The Agents (Renderer Processes)
Modular AI workers that execute specific tasks:
- **@Discuss:** Constitutional AI for ethical alignment.
- **@Plan:** Strategic decomposition of goals.
- **@Execute:** Tactical implementation and terminal control.

## The ODAVL Loop

The system operates on a 5-step control loop, typically running at 1Hz or Event-Driven:

1.  **OBSERVE:** Gather state (Bio, Market, System).
2.  **DECIDE:** Evaluate priorities based on user-defined Constitution.
3.  **ACT:** execute tool calls or prompt user.
4.  **VERIFY:** Check `stdout` or API response for success.
5.  **LOOP:** Update the Graph with new state.

## Security Model

*   **Local-First:** All sensitive keys are stored in system keychain.
*   **Sandboxed Agents:** Verification agents run in isolated contexts.
*   **Human-in-the-Loop:** High-stakes actions (financial trends > $X) require explicit "Y" confirmation.
