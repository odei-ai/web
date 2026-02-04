# Stage 3 Migration - Test Plan Overview

## Document Structure

Three comprehensive test documents have been created to validate Stage 3 migration of the ODEI Memory Atlas system:

### 1. **STAGE3_TEST_PLAN.md** (41 KB) - Comprehensive Reference

**Best for:** Detailed specification of each test, understanding test logic, troubleshooting

**Contents:**

- Full specification for 17 tests (Tests 1-17)
- Test objectives, commands, expected outputs
- Pass/fail criteria for each test
- Remediation steps (what to do if test fails)
- Related files and Neo4j queries
- 5 sections covering:
  1. Vector Index Health (Tests 1-3)
  2. Embedding Quality (Tests 4-7)
  3. MCP Tools Availability (Tests 8-10)
  4. Session Start Protocol (Tests 11-14)
  5. Hybrid Search Functional (Tests 15-17)

**Key Features:**

- Exact commands to run for each test
- Expected output examples
- Root cause analysis for failures
- Performance benchmarks
- Rollback procedures

---

### 2. **STAGE3_TEST_PLAN_SUMMARY.md** (8.4 KB) - Quick Reference

**Best for:** Quick lookup, test overview, executive summary

**Contents:**

- What was migrated (1-page summary table)
- Test structure overview (17 tests, 5 sections)
- Quick command reference
- Pass/fail criteria summary (bullet format)
- Expected results for each section
- Troubleshooting table (Issue → Root Cause → Fix)
- Performance targets
- Rollback procedure levels

**Key Features:**

- Compact, scannable format
- Command cheat sheet
- Quick decision tree for failures
- Success metrics at a glance

---

### 3. **STAGE3_TEST_EXECUTION_GUIDE.md** (22 KB) - How to Run Tests

**Best for:** Actually running the tests, step-by-step execution

**Contents:**

- Pre-test checklist (verify prerequisites)
- Individual bash scripts for Tests 1-9, 15-17
- Manual execution steps for Tests 10-14 (interactive)
- Complete master test execution script
- Result interpretation guide
- Logging and documentation steps
- Troubleshooting during execution

**Key Features:**

- Copy-paste ready bash scripts
- Color-coded output (green/red/yellow)
- Step-by-step instructions
- Automated test runner
- Error handling and diagnostics

---

## Quick Start

### Step 1: Read the Overview

```bash
# Understand what's being tested
cat /Users/ai/ODEI/STAGE3_TEST_PLAN_SUMMARY.md
```

### Step 2: Verify Prerequisites

```bash
# Run pre-flight checks
bash << 'EOF'
# 1. Neo4j running?
docker ps | grep neo4j

# 2. MCP server built?
ls /Users/ai/ODEI/servers/odei-neo4j/dist/index.js

# 3. Database accessible?
cypher-shell -u neo4j -p juehxbr413390 "RETURN 1;"

# 4. OPENAI_API_KEY set?
echo $OPENAI_API_KEY | head -c 20

# 5. Foundation nodes exist?
cypher-shell -u neo4j -p juehxbr413390 \
  "MATCH (n:OdeiNode {status: 'active'}) RETURN count(n);"
EOF
```

### Step 3: Run Tests

```bash
# Option A: Run individual test
cd /Users/ai/ODEI/servers/odei-neo4j
bash /tmp/test_1.sh    # Vector indexes exist
bash /tmp/test_4.sh    # Embeddings are 3072D
bash /tmp/test_8.sh    # v2 tools registered

# Option B: Run all automated tests
bash /tmp/run_all_tests.sh

# Option C: Manual tests (require Discuss agent)
# See STAGE3_TEST_EXECUTION_GUIDE.md for Tests 10-14
```

### Step 4: Interpret Results

```
✓ All tests PASS → Stage 3 migration successful
⚠ Some tests FAIL → See STAGE3_TEST_PLAN.md "Fail Actions"
✗ Critical tests FAIL → Follow rollback procedure in STAGE3_TEST_PLAN_SUMMARY.md
```

---

## Test Coverage Matrix

| Section | Test # | Component                              | Status                 |
| ------- | ------ | -------------------------------------- | ---------------------- |
| 1       | 1      | 14 vector indexes at 3072D             | Automated              |
| 1       | 2      | Quantization enabled                   | Automated              |
| 1       | 3      | All indexes ONLINE                     | Automated              |
| 2       | 4      | Foundation nodes have 3072D embeddings | Automated              |
| 2       | 5      | No zero-valued embeddings              | Automated              |
| 2       | 6      | Semantic search works                  | Automated              |
| 2       | 7      | Similarity scores reasonable           | Automated              |
| 3       | 8      | v2 tools registered                    | Automated              |
| 3       | 9      | v2 tools in allow list                 | Automated              |
| 3       | 10     | v2 tools callable                      | Manual (needs Discuss) |
| 4       | 11     | Session start loads context            | Manual (needs Discuss) |
| 4       | 12     | Context completeness verified          | Manual (needs Discuss) |
| 4       | 13     | Token usage <25K                       | Manual (needs Discuss) |
| 4       | 14     | No authorization errors                | Manual (needs Discuss) |
| 5       | 15     | Hybrid search returns results          | Automated              |
| 5       | 16     | RRF scoring functional                 | Automated              |
| 5       | 17     | Context expansion works                | Automated              |

---

## What Each Test Validates

### Section 1: Vector Index Health

**Why it matters:** Indexes are the foundation for fast vector similarity search. Wrong dimensions or offline indexes break semantic search entirely.

**Tests 1-3 confirm:**

- All 14 indexes created (Value, Principle, Guardrail, Goal, Vision, etc.)
- Each has exactly 3072 dimensions (not 1536, not variable)
- Quantization enabled (75% memory reduction)
- All indexes ONLINE and fully populated

### Section 2: Embedding Quality

**Why it matters:** Garbage embeddings → garbage search results. Must verify all data actually migrated to 3072D properly.

**Tests 4-7 confirm:**

- Active Foundation nodes have 3072D embeddings (not legacy 1536D)
- Zero zero-valued embeddings (no failed generations)
- Semantic search returns relevant results
- Similarity scores follow expected distribution (0.65+ for related, 0.3-0.5 for unrelated)

### Section 3: MCP Tools Availability

**Why it matters:** v2 tools with field exclusion reduce token usage by 66%. Tools must be registered and callable.

**Tests 8-10 confirm:**

- foundation.list.v2 and vision.list.v2 registered
- workload.assess.v1 and hybrid.search.v1 available
- All 4 tools in Discuss agent's allow list
- Tools callable without authorization errors

### Section 4: Session Start Protocol

**Why it matters:** Context must load automatically at session start with full foundation/vision/goals in <25K tokens.

**Tests 11-14 confirm:**

- Discuss Agent detects new session and loads context
- Values, Principles, Guardrails, Goals all loaded
- Context complete in <10 seconds
- Token usage <25K (budget 130K total)
- No authorization or connection errors

### Section 5: Hybrid Search Functional

**Why it matters:** Hybrid search (semantic + structural via RRF) provides better relevance than semantic alone.

**Tests 15-17 confirm:**

- Hybrid search returns blended results
- RRF formula correctly implements: score = 1/(k+rank)
- Weights: 60% semantic, 40% structural
- Context expansion finds related nodes (1-hop neighbors)

---

## Success Criteria

**Stage 3 migration is SUCCESSFUL if:**

1. ✓ All 17 tests PASS
2. ✓ Vector indexes: 14 exist, 3072D, ONLINE
3. ✓ Embeddings: All active nodes have 3072D (zero 1536D)
4. ✓ MCP Tools: v2 tools registered, in allow list, callable
5. ✓ Session Start: Context loads <25K tokens, no errors
6. ✓ Hybrid Search: RRF merges semantic + structural results
7. ✓ Performance: <1s response time for searches

**Migration requires REMEDIATION if:**

- Tests 1-3 fail (index infrastructure broken)
- Tests 4-5 fail (embeddings not properly migrated)
- Tests 8-9 fail (MCP configuration incomplete)
- Tests 11-14 fail (session protocol broken)

**Migration requires ROLLBACK if:**

- Critical data corruption detected
- > 50% of tests fail
- Neo4j integrity compromised

---

## File Locations

**Test Documentation:**

- `/Users/ai/ODEI/STAGE3_TEST_PLAN.md` - Detailed specification (41 KB)
- `/Users/ai/ODEI/STAGE3_TEST_PLAN_SUMMARY.md` - Quick reference (8.4 KB)
- `/Users/ai/ODEI/STAGE3_TEST_EXECUTION_GUIDE.md` - How to run tests (22 KB)

**MCP Server Implementation:**

- `/Users/ai/ODEI/servers/odei-neo4j/src/tools/foundation-list.ts` - v2 tool
- `/Users/ai/ODEI/servers/odei-neo4j/src/tools/vision-list.ts` - v2 tool
- `/Users/ai/ODEI/servers/odei-neo4j/src/tools/workload-assess.ts` - token-optimized
- `/Users/ai/ODEI/servers/odei-neo4j/src/tools/hybridSearch.ts` - RRF implementation (280+ lines)

**Configuration:**

- `/Users/ai/ODEI/.claude/mcp.json` - EMBEDDING_DIMENSIONS: 3072
- `/Users/ai/ODEI/agents/discuss/.claude/mcp.json` - toolAllowList
- `/Users/ai/ODEI/agents/discuss/.claude/CLAUDE.md` - Session start protocol

---

## Test Execution Timeline

| Phase           | Time       | Action                               |
| --------------- | ---------- | ------------------------------------ |
| Pre-flight      | 5 min      | Verify Neo4j, MCP build, credentials |
| Automated tests | 20 min     | Run Tests 1-9, 15-17                 |
| Manual tests    | 15 min     | Run Tests 10-14 in Discuss Agent     |
| Analysis        | 10 min     | Interpret results, document findings |
| **Total**       | **50 min** | Complete test validation             |

---

## Troubleshooting Workflows

### If a test fails:

1. **Read the test** in STAGE3_TEST_PLAN.md
2. **Check "Expected Output"** section - does your output match?
3. **Review "Pass Criteria"** - which condition failed?
4. **Follow "Fail Actions"** - specific remediation steps
5. **Re-run the test** to confirm fix
6. **Continue to next test** if fixed

### If multiple tests fail:

1. Check if they're related (e.g., Tests 1-3 all about indexes)
2. Fix the root cause (usually Index creation or Embedding migration)
3. Re-run the group of related tests
4. If still failing, refer to "Rollback Procedure" section

### If in doubt:

1. Check Pre-flight checklist
2. Review Neo4j logs: `docker logs <neo4j_container>`
3. Check MCP server logs: stdout/stderr from `node dist/index.js`
4. Verify credentials: `cypher-shell -u neo4j -p <password>`
5. Verify OPENAI_API_KEY: `echo $OPENAI_API_KEY | head -c 20`

---

## Document Usage Guide

**I want to understand what's being tested:**
→ Read `STAGE3_TEST_PLAN_SUMMARY.md` (5 min read)

**I want detailed specifications for a specific test:**
→ Open `STAGE3_TEST_PLAN.md`, search for "Test N"

**I'm ready to run the tests:**
→ Follow `STAGE3_TEST_EXECUTION_GUIDE.md` with bash scripts

**A test failed, what do I do?**
→ Find test in `STAGE3_TEST_PLAN.md`, read "Fail Actions" section

**I need quick command reference:**
→ Check `STAGE3_TEST_PLAN_SUMMARY.md` "Quick Command Reference" section

**I want to understand the architecture:**
→ Read STAGE3_TEST_PLAN_SUMMARY.md "What Was Migrated" table

---

## Next Steps After Passing All Tests

Once Stage 3 migration is validated:

1. **Update documentation:**
   - Update Migration notes
   - Document any deviations from expected results
   - Archive test results

2. **Deploy to production:**
   - Push changes to main branch
   - Deploy MCP server updates
   - Verify in production with same tests

3. **Monitor performance:**
   - Track embedding generation time
   - Monitor search response times
   - Verify token savings in production

4. **Plan Stage 4:**
   - Next phase of optimization
   - Additional enhancements based on validation

---

## Contact for Issues

If tests fail and you need support:

1. Document exact test failure (screenshot or copy output)
2. Include error messages verbatim
3. Note any environment differences (Neo4j version, etc.)
4. Provide the test command that failed
5. Include output of pre-flight checks

---

**Created:** 2025-11-04
**Version:** 1.0
**Status:** Ready for execution

Three comprehensive documents provided for Stage 3 migration validation.
