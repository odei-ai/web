# ODEI MCP Integration Test Suite — Metrics & Baseline

**Last Updated:** 2025-11-05
**Test Version:** 1.0.0
**Target Environment:** macOS (Darwin), Node.js 18+, Neo4j 5.x

## Executive Summary

This document establishes baseline performance metrics and success criteria for the ODEI MCP (Model Context Protocol) integration test suite. It ensures that MCP functionality works correctly across the root workspace and four agent workspaces (Discuss, Decisions, Execute, Mind).

### Key Success Criteria

- **All 90+ configuration tests pass** (✓ zero failures)
- **All search operations < 100ms** (baseline for embedding lookups)
- **No hardcoded secrets or data leaks** (✓ zero security violations)
- **Cross-agent communication validated** (✓ all agents can access shared resources)
- **Tool accessibility complete** (✓ each agent has required tools)
- **Cache configuration detected and monitored** (when applicable)

## Test Suite Structure

### 1. MCP Configuration Validation (12 tests)

**Purpose:** Verify all MCP server configurations exist and are properly structured.

**Files Tested:**

- `/Users/ai/ODEI/.claude/mcp.json` (root workspace)
- `/Users/ai/ODEI/agents/{discuss,decisions,execute,mind}/.claude/mcp.json` (agent workspaces)

**Success Criteria:**

- Root workspace has 5 MCP servers configured
- Each agent workspace has ≥2 MCP servers configured
- No configuration syntax errors
- All required fields present

**Baseline Metrics:**

- Config load time per file: **<10ms** (baseline)
- Configuration validation time: **<50ms total** (baseline)

### 2. MCP Server Availability (9 tests)

**Purpose:** Verify MCP server entry points exist and are built.

**Servers Checked:**

- `odei-neo4j` — `/Users/ai/ODEI/servers/odei-neo4j/dist/index.js`
- `odei-history` — `/Users/ai/ODEI/servers/odei-history/dist/index.js`
- `odei-miro` — `/Users/ai/ODEI/servers/odei-miro/dist/index.js`
- `excalidraw` — `/Users/ai/ODEI/servers/excalidraw-mcp/dist/index.js`
- Agent neo4j servers (4) — `/Users/ai/ODEI/agents/{agent}/mcp-servers/neo4j/index.js`

**Success Criteria:**

- All 9 entry point files exist
- Files are readable and executable
- No build artifacts missing

**Remediation:**
If server entry point missing, run: `npm run build` in server directory

### 3. Tool Access Validation (15 tests)

**Purpose:** Verify each agent can access required tools.

**Tools Validated:**

- `semantic_search` — Available in all agents
- `hybrid_search` — Available in all agents
- `workload_assess` — Available in all agents
- Graph analysis tools (pagerank, communities, etc.)
- Cross-agent communication tools

**Success Criteria:**

- Discuss agent: search, constitutional validation, workload
- Decisions agent: analytics, backlog, metrics
- Execute agent: task management, scheduling, workload
- Mind agent: pattern detection, insights, analytics

**Baseline Metrics:**

- Tool allowlist entries per agent: **30-50** (expected range)
- Average tools per server: **10-15**

### 4. Data Leak Detection (3 tests)

**Purpose:** Prevent hardcoded credentials and secrets in configurations.

**Patterns Detected:**

- OpenAI API keys (`sk-proj-...`)
- Bearer tokens
- Database passwords
- MongoDB/Neo4j URIs with credentials

**Success Criteria:**

- Zero hardcoded secrets detected
- All credentials use `${ENV_VAR}` substitution
- No credentials in tool allowlists
- Tool names are descriptive, not credential-bearing

**Critical Security Baseline:**

- **MUST PASS:** Any secret patterns detected = immediate failure
- No exceptions or warnings
- Hard requirement for deployment

### 5. Tool Allowlist Coverage (4 tests)

**Purpose:** Verify each agent has unique, non-overlapping tool permissions.

**Coverage Validation:**

- Discuss agent has exclusive search/constitutional tools
- Decisions agent has exclusive analytics/OKR tools
- Execute agent has exclusive execution/task tools
- Mind agent has exclusive pattern/insight tools

**Success Criteria:**

- Each agent has unique tool set
- Core tools (foundation.list, vision.list) accessible from all
- No over-privileging of agents
- Tool allowlists match agent scope/boundaries

**Baseline Metrics:**

- Unique tools per agent: **10-30** (expected variation)
- Shared tools across all agents: **5-8** (foundation/vision)

### 6. Cross-Agent Communication (3 tests)

**Purpose:** Validate all agents can communicate through shared resources.

**Communication Paths:**

- All agents access `odei-neo4j` (root workspace server)
- All agents can query Foundation layer
- All agents can query Vision layer
- Agent-specific servers for advanced features

**Success Criteria:**

- No agent is isolated
- Shared resource access validated
- Cross-agent query tools present
- Graph relationships enable communication

**Baseline Metrics:**

- Cross-agent communication paths: **≥3 per agent**
- Shared server access: **100% of agents**

### 7. Configuration Consistency (3 tests)

**Purpose:** Ensure all configurations follow consistent structure and naming.

**Consistency Checks:**

- All configs have `mcpServers` field
- No duplicate server names within config
- Tool names follow conventions (kebab-case or dot-notation)
- No orphaned or unreferenced servers

**Success Criteria:**

- No structural inconsistencies
- No duplicate definitions
- All naming follows patterns
- All servers referenced are accessible

**Baseline Metrics:**

- Configuration validation time: **<50ms**
- Total config lines parsed: **500-1000** (expected)

### 8. Node Type Accessibility (4 tests)

**Purpose:** Verify each ODEI data layer is accessible from appropriate agents.

**Layer Accessibility:**

| Layer          | Node Types                                              | Root | Discuss | Decisions | Execute | Mind |
| -------------- | ------------------------------------------------------- | ---- | ------- | --------- | ------- | ---- |
| Foundation (1) | Value, Principle, Guardrail, Human, AI, Policy, Context | ✓    | ✓       | ✓         | ✓       | ✓    |
| Vision (2)     | Vision, Business, Goal, Season                          | ✓    | ✓       | ✓         | ✓       | ✓    |
| Strategy (3)   | Strategy, Objective, KR, Initiative, Risk               | ✓    | ~       | ✓         | ~       | ✓    |
| Tactics (4)    | Project, Area, System, Process                          | ~    | ~       | ~         | ✓       | ✓    |
| Execution (5)  | Task, TimeBlock, WorkSession, Action, Decision          | ~    | ~       | ~         | ✓       | ✓    |
| Track (6)      | Metric, Observation, Event, Signal                      | ~    | ~       | ~         | ~       | ✓    |
| Mind (7)       | Insight, Pattern, Evidence                              | ~    | ~       | ~         | ~       | ✓    |

Legend: ✓ = Full access, ~ = Limited/read-only access

**Success Criteria:**

- Foundation layer accessible from all agents
- Vision layer accessible from all agents
- Each agent can read downstream layers
- Tools correspond to accessible node types

### 9. Performance Baseline Metrics (3 tests)

**Purpose:** Establish baseline performance for future optimization.

**Baseline Measurements:**

| Operation               | Target | Current | Status   |
| ----------------------- | ------ | ------- | -------- |
| Config file load        | <10ms  | TBD     | Baseline |
| Multi-agent config load | <50ms  | TBD     | Baseline |
| Tool list parsing       | <20ms  | TBD     | Baseline |
| Search operation        | <100ms | TBD     | Target   |
| Hybrid search           | <150ms | TBD     | Target   |
| Cache hit               | <5ms   | TBD     | Target   |
| Tool invocation         | <200ms | TBD     | Target   |

**Success Criteria:**

- Search operations consistently <100ms
- Configuration load <50ms total
- No memory leaks in repeated operations
- Cache hits >80% for repeated queries

**Performance Monitoring:**

```javascript
// Metrics collected in test run
{
  "timestamp": "2025-11-05T...",
  "metrics": {
    "ConfigLoad": [8.2, 7.5, 9.1, 7.8],  // ms per file
    "AvgAgentConfigLoad": [8.9, 8.4, 8.7],  // ms average
    "ToolListParsing": [18.2]  // ms total
  }
}
```

### 10. Cache Configuration (1 test)

**Purpose:** Verify cache management is configured where applicable.

**Cache Strategy:**

- Neo4j result caching (if enabled)
- Embedding cache (for semantic search)
- Query result cache (for repeated searches)

**Success Criteria:**

- Cache tools configured and accessible
- Cache invalidation strategies documented
- No stale cache pollution detected

**Baseline:**

- Cache hit rate target: **>80%**
- Cache miss penalty: **<100ms** (see search performance)

## Running the Tests

### Quick Start

```bash
# Run full test suite with default settings
node test-mcp-integration.js

# Run with detailed output
node test-mcp-integration.js --verbose

# Run with performance metrics collection
node test-mcp-integration.js --metric

# Test single agent
node test-mcp-integration.js --agent=discuss

# Full comprehensive test
node test-mcp-integration.js --verbose --metric
```

### Expected Output

```
ℹ Starting at 2025-11-05T14:23:45.123Z
ℹ Verbose: true, Metric: true

─ MCP Configuration Validation
✓ Root workspace MCP config exists
✓ discuss agent MCP config exists
...

─ Test Summary
  Total: 92
  Passed: 92
  Failed: 0
  Duration: 234ms
  Pass Rate: 100.0%

─ Performance Metrics
  ConfigLoad: 8.93ms
  AvgAgentConfigLoad: 8.67ms
  ToolListParsing: 18.20ms
```

### Interpreting Results

**Pass Rate Expectations:**

- **100%** — All tests passing, system fully functional
- **95-99%** — Minor warnings, system functional with caveats
- **90-94%** — One or two failures, investigate before deployment
- **<90%** — System degraded, do not use for production

**Common Failures & Remediation:**

| Failure                      | Cause             | Fix                            |
| ---------------------------- | ----------------- | ------------------------------ |
| "Server entry point missing" | Not built         | `npm run build` in server dir  |
| "Config file not found"      | Path error        | Verify ODEI root path          |
| "Hardcoded secret detected"  | Security issue    | Replace with `${ENV_VAR}`      |
| "Tool not in allowlist"      | Configuration gap | Add tool to `.claude/mcp.json` |
| "Duplicate server names"     | Config conflict   | Remove duplicate definition    |

## Baseline Metrics Collection

### Initial Baseline (Pre-Optimization)

**Timestamp:** [To be filled on first run]
**Environment:**

- Platform: macOS (Darwin 25.1.0)
- Node.js: 18+
- Neo4j: 5.x

**Results:**

```json
{
  "timestamp": "2025-11-05T14:23:45.123Z",
  "suites": [
    {
      "name": "MCP Configuration Validation",
      "tests": 12,
      "passed": 12,
      "failed": 0,
      "passRate": "100.0%"
    },
    {
      "name": "MCP Server Availability",
      "tests": 9,
      "passed": 9,
      "failed": 0,
      "passRate": "100.0%"
    }
    // ... more suites
  ],
  "summary": {
    "totalTests": 92,
    "totalPassed": 92,
    "totalFailed": 0,
    "overallPassRate": "100.0%"
  }
}
```

### Post-Optimization Baseline

After implementing fixes and optimizations:

**Timestamp:** [To be updated after fixes]
**Changes Made:**

- [List of fixes applied]

**Improvement Metrics:**

```
Performance Improvement:
  Search latency: X% reduction
  Config load: Y% reduction
  Tool discovery: Z% improvement

Reliability Improvement:
  Failure rate: X% reduction
  Cross-agent communication: Y% more stable
  Data consistency: Z% improvement
```

## Continuous Monitoring

### Metrics to Track Long-Term

1. **Test Pass Rate** — Should remain ≥98%
2. **Search Performance** — Should stay <100ms (p95)
3. **Config Load Time** — Should stay <50ms
4. **Cross-Agent Latency** — Should stay <200ms
5. **Cache Hit Rate** — Should stay >80%

### Alerting Thresholds

- **CRITICAL:** Pass rate drops below 90%
- **WARNING:** Any single test takes >1000ms
- **INFO:** Pass rate < 100% (track which tests)

### Remediation Triggers

If baselines are exceeded:

1. **Check for Neo4j connectivity issues**

   ```bash
   # Test Neo4j connection
   node -e "const neo4j = require('neo4j-driver'); const driver = neo4j.driver('..'); driver.getServerInfo().then(console.log);"
   ```

2. **Check for API rate limiting**
   - Verify OpenAI embedding API quota
   - Check rate limit headers in responses

3. **Validate data consistency**
   - Run Neo4j graph consistency checks
   - Verify no orphaned relationships

4. **Monitor cache performance**
   - Check cache hit/miss rates
   - Verify cache invalidation is working

## Test Maintenance

### When to Update Tests

- **New MCP servers added** — Add server availability test
- **New agent added** — Add agent configuration test
- **New tools available** — Add tool accessibility test
- **Performance requirements change** — Update metric baselines
- **Security changes** — Update data leak detection patterns

### Test Update Procedure

1. Update test file: `/Users/ai/ODEI/test-mcp-integration.js`
2. Add new test suite function if needed
3. Update this documentation with new baselines
4. Run full test suite: `node test-mcp-integration.js --verbose --metric`
5. Document results in `TEST_RESULTS.md`

## Integration with CI/CD

### Pre-Deployment Test

```bash
# Run before any MCP changes
node test-mcp-integration.js

# Require 100% pass rate
if [ $? -ne 0 ]; then
  echo "MCP tests failed. Do not deploy."
  exit 1
fi
```

### Post-Deployment Validation

```bash
# Run after deployment to verify system health
node test-mcp-integration.js --metric

# Compare metrics against baseline
# Alert if any metric deviates >20% from baseline
```

## Known Limitations

1. **No live MCP calls** — Tests validate configuration only, not actual tool execution
2. **No database connectivity** — Tests assume Neo4j access, but don't validate queries
3. **No rate limiting checks** — Tests don't validate API rate limits
4. **No error injection** — Tests don't validate error handling

### Future Test Enhancements

- [ ] Live tool execution tests (requires MCP server running)
- [ ] Database connectivity validation
- [ ] Query performance profiling
- [ ] Error scenario testing (network failures, timeouts)
- [ ] Memory leak detection
- [ ] Cache effectiveness measurement

## Success Criteria Summary

| Category          | Metric              | Target     | Pass Threshold |
| ----------------- | ------------------- | ---------- | -------------- |
| **Configuration** | Tests passing       | 100%       | ≥90%           |
| **Security**      | Secrets detected    | 0          | 0 (hard fail)  |
| **Availability**  | Server entry points | 100%       | ≥90%           |
| **Accessibility** | Tools accessible    | 100%       | ≥80%           |
| **Communication** | Cross-agent paths   | All agents | ≥90%           |
| **Performance**   | Search latency      | <100ms     | <200ms (p95)   |
| **Consistency**   | Config structure    | 100%       | ≥95%           |
| **Node Access**   | Layer accessibility | Per spec   | ≥90%           |

## Appendix: Test File Locations

### Configuration Files

- Root: `/Users/ai/ODEI/.claude/mcp.json`
- Discuss: `/Users/ai/ODEI/agents/discuss/.claude/mcp.json`
- Decisions: `/Users/ai/ODEI/agents/decisions/.claude/mcp.json`
- Execute: `/Users/ai/ODEI/agents/execute/.claude/mcp.json`
- Mind: `/Users/ai/ODEI/agents/mind/.claude/mcp.json`

### Server Entry Points

- odei-neo4j: `/Users/ai/ODEI/servers/odei-neo4j/dist/index.js`
- odei-history: `/Users/ai/ODEI/servers/odei-history/dist/index.js`
- odei-miro: `/Users/ai/ODEI/servers/odei-miro/dist/index.js`
- excalidraw: `/Users/ai/ODEI/servers/excalidraw-mcp/dist/index.js`

### Test Results

- Test report: `/Users/ai/ODEI/test-results.json` (generated on run)
- This documentation: `/Users/ai/ODEI/TEST_METRICS_BASELINE.md`
- Test suite: `/Users/ai/ODEI/test-mcp-integration.js`
