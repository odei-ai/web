# Comprehensive Error Handling Enhancements for ODEI Services

## Overview

This document outlines comprehensive error handling improvements for three critical services in the ODEI system to prevent silent failures and ensure graceful degradation under adverse conditions.

## Files Enhanced

1. `/Users/ai/ODEI/servers/odei-neo4j/src/tools/hybridSearch.ts`
2. `/Users/ai/ODEI/servers/odei-neo4j/src/services/embeddingManager.ts`
3. `/Users/ai/ODEI/servers/odei-neo4j/src/services/personalizer.ts`

---

## 1. HybridSearch Error Handling Enhancements

### File: `servers/odei-neo4j/src/tools/hybridSearch.ts`

### Changes Required

#### 1.1 Add Connection Error Detection Utility

```typescript
/**
 * Detects if an error is a connection-related error
 * Used to distinguish temporary failures from permanent ones
 */
function isConnectionError(error: unknown): boolean {
  const message = error instanceof Error ? error.message : String(error);
  return (
    message.includes('ECONNREFUSED') ||
    message.includes('ENOTFOUND') ||
    message.includes('connect') ||
    message.includes('timeout') ||
    message.includes('unavailable') ||
    message.includes('ECONNRESET')
  );
}
```

#### 1.2 Enhanced Session Opening

```typescript
function openSession(driver: Driver) {
  try {
    return driver.session({ database: DEFAULT_DB });
  } catch (error) {
    const err = error instanceof Error ? error.message : String(error);
    throw new Error(`Failed to open Neo4j session: ${err}`);
  }
}
```

#### 1.3 Embedding Generation with Graceful Degradation

- Add input validation for query text
- Validate embedding response before using it
- Log detailed error information with context
- Ensure session cleanup in finally block
- Return null on failure (not empty array) to signal fallback to structural search

**Expected Behavior:**

- If OpenAI API fails, log warning and return null
- Structural search continues automatically
- User still gets results from structural search
- No blank results or crashes

#### 1.4 Semantic Search with Fallback

- Validate embedding input before executing query
- Try-catch around Neo4j query execution
- Try-catch around record conversion
- Filter out null records from conversion failures
- Return empty array on complete failure (triggers fallback)

**Expected Behavior:**

- Individual record conversion failures don't stop entire search
- Neo4j connection errors are logged
- System falls back to structural search
- Returns partial results if some records convert

#### 1.5 Structural Search with Multi-Level Fallback

Three-level fallback strategy:

1. **Level 1:** Full-text index search
2. **Level 2:** CONTAINS-based tokenized search (if Level 1 fails)
3. **Level 3:** Basic node matching by layer (if Levels 1 & 2 fail)

**Expected Behavior:**

- Tracks which search strategy was used in logs
- Returns results at any level rather than failing completely
- Each level has error handling with descriptive logging
- Input validation prevents invalid queries from being sent

#### 1.6 Graph Context Expansion with Graceful Handling

- Individual record processing wrapped in try-catch
- Partial context nodes returned even if some fail to convert
- Returns seed results without context if expansion fails completely
- Session cleanup even on error

**Expected Behavior:**

- Expansion failure doesn't lose original search results
- Partial context is better than no context
- System continues functioning

### Implementation Benefits for HybridSearch

1. **OpenAI API Failures** - Returns partial results with warning
2. **Neo4j Connection Failures** - Falls back through 3 search strategies
3. **Individual Record Failures** - Skips bad records, continues with good ones
4. **Invalid Inputs** - Validates and logs helpful error messages
5. **Never Shows Blank Results** - Always has fallback strategy

---

## 2. Embedding Manager Error Handling Enhancements

### File: `servers/odei-neo4j/src/services/embeddingManager.ts`

### Changes Required

#### 2.1 Enhanced Retriable Error Detection

```typescript
function isRetriableError(error: unknown): boolean {
  const message = error instanceof Error ? error.message : String(error);

  // Retriable: rate limits, timeouts, temporary network issues
  if (
    message.includes('rate limit') ||
    message.includes('timeout') ||
    message.includes('ECONNRESET') ||
    message.includes('ECONNREFUSED') ||
    message.includes('ENOTFOUND') ||
    message.includes('429') || // Too many requests
    message.includes('503') || // Service unavailable
    message.includes('temporarily unavailable')
  ) {
    return true;
  }

  // Non-retriable: auth errors, invalid model, permanent failures
  if (
    message.includes('invalid') ||
    message.includes('unauthorized') ||
    message.includes('not found') ||
    message.includes('401') || // Unauthorized
    message.includes('403') || // Forbidden
    message.includes('404')
  ) {
    // Not found
    return false;
  }

  // Default: retry other errors (assume temporary)
  return true;
}
```

#### 2.2 Comprehensive Input Validation

- Validate node object exists and has ID
- Check driver availability
- Verify API key existence
- Validate embedding text is not empty
- Skip gracefully on validation failures

#### 2.3 API Call Error Handling

- Wrap OpenAI embeddings.create() in try-catch
- Validate response structure
- Validate embedding array
- Non-retriable errors on invalid API data
- Descriptive error messages for debugging

#### 2.4 Neo4j Write Error Handling

- Separate try-catch for database write
- Check for node existence before writing
- Handle write failures with proper logging
- Ensure session closes even on write error
- Return gracefully if node doesn't exist

#### 2.5 Exponential Backoff with Retry Logic

- 1s, 2s, 4s exponential backoff
- Distinguish retriable vs permanent failures
- Log retry attempts with backoff timing
- Give up gracefully on non-retriable errors
- Log all-retries-exhausted message

**Expected Behavior:**

- Temporary failures (rate limits, timeouts) automatically retry
- Permanent failures (auth, not found) fail immediately
- All attempts logged with timing information
- System continues without embedding if all retries fail
- No silent failures or cascading errors

### Implementation Benefits for EmbeddingManager

1. **API Failures** - Retries temporary failures, gives up on permanent ones
2. **Network Timeouts** - Exponential backoff prevents overwhelming service
3. **Database Issues** - Handles missing nodes and write failures gracefully
4. **Resource Cleanup** - Session always closes even on error
5. **Observability** - Detailed logging of all failures and retries

---

## 3. Personalizer Error Handling Enhancements

### File: `servers/odei-neo4j/src/services/personalizer.ts`

### Changes Required

#### 3.1 RankResults with Defensive Programming

- Validate results array and context object
- Return empty array for invalid inputs
- Wrap individual result processing in try-catch
- Handle each boost calculation failure independently
- Filter out null results from failures
- Safe sorting that handles sort errors

**Error Handling per Result:**

```typescript
try {
  // Validate individual result
  if (!result || !result.node || !result.node.id) {
    return null; // Will be filtered out
  }

  // Process result...
} catch (error) {
  // Log and continue with next result
  console.warn('Error processing result:', error);
  return null;
}
```

#### 3.2 Individual Boost Functions with Try-Catch

Each boost function should:

- Validate input data exists and has correct type
- Return neutral boost (1.0) on any error
- Never throw exceptions
- Include null checks for optional properties
- Log warnings but continue

**Pattern for Each Boost:**

```typescript
private applyXxxBoost(node: GraphNodeRecord, context: UserContext): number {
  try {
    // Validate inputs
    if (!context?.property || !node?.field) {
      return 1.0;
    }

    // Apply boost logic
    const boost = calculateBoost(...);

    return Math.max(1.0, Math.min(maxBoost, boost)); // Ensure range
  } catch (error) {
    console.warn("Error in boost function:", error);
    return 1.0; // Neutral boost on error
  }
}
```

#### 3.3 Graceful Sorting with Error Handling

```typescript
.sort((a, b) => {
  try {
    return (b.personalizedScore ?? 0) - (a.personalizedScore ?? 0);
  } catch {
    return 0; // If sort fails, maintain order
  }
})
```

### Implementation Benefits for Personalizer

1. **Invalid Inputs** - Returns unpersonalized results rather than crashing
2. **Missing Properties** - Uses defaults and continues
3. **Boost Calculation Failures** - Skips failed boost, applies others
4. **Sorting Failures** - Maintains original order if sort errors
5. **Partial Personalization** - Returns results even if some boosts fail

---

## Error Handling Patterns Applied

### 1. Input Validation

```typescript
if (!input || typeof input !== 'expected') {
  logger?.warn?.({ input }, 'Invalid input');
  return defaultValue;
}
```

### 2. Graceful Degradation

```typescript
try {
  return executeOptimalPath();
} catch (error) {
  logger?.warn?.({ error }, 'Optimal path failed, using fallback');
  return executeFallbackPath();
}
```

### 3. Partial Failure Handling

```typescript
const results = items
  .map((item) => {
    try {
      return process(item);
    } catch (error) {
      logger?.warn?.({ error, item }, 'Failed to process item');
      return null; // Mark for filtering
    }
  })
  .filter((r) => r !== null);
```

### 4. Resource Cleanup

```typescript
let session;
try {
  session = createResource();
  // Use resource
} catch (error) {
  // Handle error
} finally {
  if (session) {
    try {
      session.close();
    } catch (closeError) {
      logger?.debug?.({ error: closeError }, 'Cleanup failed');
    }
  }
}
```

### 5. Error Classification

```typescript
function isRetriable(error: unknown): boolean {
  const msg = error instanceof Error ? error.message : String(error);
  // Classify error type
  if (isTemporary(msg)) return true;
  if (isPermanent(msg)) return false;
  return true; // Default: retry
}
```

---

## Benefits Summary

### For Users

- Never see blank results or error screens
- System degrades gracefully under failures
- Partial results better than no results
- Faster response times with fallbacks

### For Operators

- Detailed error logging for debugging
- Clear distinction between temporary and permanent failures
- Connection issues identified and logged
- Observability into system behavior

### For System Reliability

- No silent failures that crash components
- Cascade failures prevented by fallbacks
- Automatic retries for transient issues
- Resource cleanup ensures no leaks

---

## Testing Recommendations

### Unit Tests

1. Test each error case in isolation
2. Verify fallback paths execute correctly
3. Test error logging contains proper context
4. Verify resource cleanup always occurs

### Integration Tests

1. Test with API failures (mock OpenAI timeouts)
2. Test with Neo4j connection failures
3. Test with invalid/corrupt data
4. Test partial data scenarios

### Load Tests

1. Verify exponential backoff prevents thundering herd
2. Test connection pool doesn't exhaust under failures
3. Verify logging doesn't impact performance

---

## Migration Path

1. Review each function's error handling
2. Add try-catch blocks around external calls
3. Implement validation at function boundaries
4. Add graceful fallback logic
5. Test error scenarios
6. Deploy with monitoring

---

## Monitoring Metrics

Track these metrics to monitor error handling effectiveness:

1. **API Failures by Type** - Track rate limits vs timeouts vs network errors
2. **Fallback Usage** - Monitor how often each fallback strategy is used
3. **Retry Counts** - Track retry effectiveness and exponential backoff usage
4. **Error Coverage** - Ensure all errors are logged and handled
5. **Resource Cleanup** - Verify no resource leaks under error conditions

---

## References

- [Neo4j Driver Error Handling](https://neo4j.com/docs/drivers/javascript/)
- [OpenAI API Error Handling](https://platform.openai.com/docs/guides/error-handling)
- [Node.js Best Practices](https://nodejs.org/en/docs/guides/nodejs-performance-best-practices/)
