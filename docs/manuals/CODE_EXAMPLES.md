# Error Handling Code Examples

This document provides specific code examples for implementing comprehensive error handling in ODEI services.

---

## 1. Connection Error Detection (hybridSearch.ts)

### Add this utility function after imports:

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

---

## 2. Enhanced Session Opening (hybridSearch.ts)

### Replace the existing `openSession` function:

```typescript
/**
 * Opens a Neo4j session with enhanced error handling
 * Throws descriptive error on failure
 */
function openSession(driver: Driver) {
  try {
    return driver.session({ database: DEFAULT_DB });
  } catch (error) {
    const err = error instanceof Error ? error.message : String(error);
    throw new Error(`Failed to open Neo4j session: ${err}`);
  }
}
```

---

## 3. Embedding Generation with Graceful Degradation (hybridSearch.ts)

### Replace the entire `generateEmbedding` function:

```typescript
/**
 * Generate embedding for query text using Neo4j GenAI integration
 * Gracefully degrades on API failures - returns null to allow fallback to structural search
 */
async function generateEmbedding(driver: Driver, query: string, ctx: ToolContext): Promise<number[] | null> {
  const apiKey = ctx.env.OPENAI_API_KEY || process.env.OPENAI_API_KEY;
  if (!apiKey) {
    ctx.logger?.debug?.('Skipping embedding generation (missing OPENAI_API_KEY)');
    return null;
  }

  // Input validation
  if (!query || query.trim().length === 0) {
    ctx.logger?.warn?.({ query }, 'Invalid query text provided for embedding generation');
    return null;
  }

  let session;
  try {
    session = openSession(driver);

    const result = await session.run(
      `
      CALL genai.vector.encodeBatch([$text], $provider, {
        token: $token,
        model: $model
      }) YIELD vector
      RETURN vector
      `,
      {
        text: query,
        provider: ctx.env.GENAI_PROVIDER || process.env.GENAI_PROVIDER || 'OpenAI',
        token: apiKey,
        model: ctx.env.EMBEDDING_MODEL || process.env.EMBEDDING_MODEL || 'text-embedding-3-large',
      }
    );

    const vector = result.records[0]?.get('vector');
    if (!vector) {
      ctx.logger?.warn?.({ query, recordCount: result.records.length }, 'OpenAI API returned empty embedding response');
      return null;
    }

    return serializeNeo4jValue(vector);
  } catch (error) {
    const errorMessage = error instanceof Error ? error.message : String(error);
    const isConnErr = isConnectionError(error);

    ctx.logger?.warn?.(
      {
        error: errorMessage,
        isConnectionError: isConnErr,
        query: query.substring(0, 100),
      },
      'Failed to generate embedding (graceful degradation enabled)'
    );

    return null; // Return null to allow structural search to continue
  } finally {
    if (session) {
      try {
        await session.close();
      } catch (closeError) {
        ctx.logger?.debug?.(
          { error: closeError instanceof Error ? closeError.message : String(closeError) },
          'Error closing session after embedding generation'
        );
      }
    }
  }
}
```

---

## 4. Semantic Search with Robust Record Handling (hybridSearch.ts)

### Replace the record mapping section in `semanticSearch`:

```typescript
const results = result.records
  .map((record, index) => {
    try {
      return {
        node: toGraphNodeRecord(record.get('n')),
        score: record.get('similarity'),
        source: 'semantic' as const,
        rank: index,
      };
    } catch (recordError) {
      ctx.logger?.warn?.(
        { error: recordError instanceof Error ? recordError.message : String(recordError), index },
        'Failed to convert semantic search record'
      );
      return null;
    }
  })
  .filter((r) => r !== null) as SearchResult[];
```

---

## 5. Structural Search with 3-Level Fallback (hybridSearch.ts)

### Key sections for `structuralSearch`:

```typescript
// Input validation at start
if (!query || query.trim().length === 0) {
  ctx.logger?.warn?.({ query }, 'Invalid query text provided for structural search');
  return [];
}

let session;
let searchStrategy = 'fulltext'; // Track which strategy is used
const layerFilter = params.layers && params.layers.length > 0 ? params.layers : Layers;

try {
  session = openSession(driver);

  let result;

  // Level 1: Try full-text index
  try {
    result = await session.run(/* fulltext query */);
  } catch (fulltextError) {
    // Level 2: Fall back to CONTAINS search
    const errorMsg = fulltextError instanceof Error ? fulltextError.message : String(fulltextError);
    ctx.logger?.debug?.({ error: errorMsg }, 'Full-text index not available, falling back to CONTAINS search');
    searchStrategy = 'contains';

    try {
      // CONTAINS search implementation
      result = await session.run(/* contains query */);
    } catch (containsError) {
      // Level 3: Fall back to basic node matching
      const containsMsg = containsError instanceof Error ? containsError.message : String(containsError);
      ctx.logger?.debug?.({ error: containsMsg }, 'CONTAINS search failed, falling back to basic node matching');
      searchStrategy = 'basic';

      result = await session.run(/* basic query */);
    }
  }

  // Process results with error handling
  const results = result.records
    .map((record, index) => {
      try {
        return {
          node: toGraphNodeRecord(record.get('node')),
          score: record.get('score'),
          source: 'structural' as const,
          rank: index,
        };
      } catch (recordError) {
        ctx.logger?.warn?.(
          { error: recordError instanceof Error ? recordError.message : String(recordError), index },
          'Failed to convert structural search record'
        );
        return null;
      }
    })
    .filter((r) => r !== null) as SearchResult[];

  ctx.logger?.debug?.({ resultCount: results.length, searchStrategy }, 'Structural search completed');

  return results;
} catch (error) {
  const errorMessage = error instanceof Error ? error.message : String(error);
  const isConnErr = isConnectionError(error);

  ctx.logger?.error?.(
    {
      error: errorMessage,
      isConnectionError: isConnErr,
      query: query.substring(0, 100),
    },
    'Structural search failed completely, returning empty results'
  );
  return [];
} finally {
  if (session) {
    try {
      await session.close();
    } catch (closeError) {
      ctx.logger?.debug?.(
        { error: closeError instanceof Error ? closeError.message : String(closeError) },
        'Error closing session after structural search'
      );
    }
  }
}
```

---

## 6. Enhanced Retriable Error Detection (embeddingManager.ts)

### Replace the `isRetriableError` function:

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

---

## 7. Comprehensive Embedding Generation (embeddingManager.ts)

### Replace the entire `ensureEmbeddingForNode` function:

```typescript
/**
 * Ensures a node has an embedding with comprehensive error handling
 * Gracefully degrades if embedding generation fails - continues without embedding
 * Uses exponential backoff and distinguishes retriable vs permanent failures
 */
export async function ensureEmbeddingForNode(
  node: GraphNodeRecord,
  ctx: ToolContext,
  options: { force?: boolean; updatedFields?: string[]; maxRetries?: number } = {}
): Promise<void> {
  // Input validation
  if (!node || !node.id) {
    ctx.logger?.warn?.({ node }, 'Invalid node provided to ensureEmbeddingForNode');
    return;
  }

  const driver = getDriver(ctx);
  if (!driver) {
    ctx.logger?.debug?.({ nodeId: node.id }, 'Neo4j driver not available, skipping embedding generation');
    return;
  }

  const apiKey = ctx.env.OPENAI_API_KEY || process.env.OPENAI_API_KEY;
  if (!apiKey) {
    ctx.logger?.debug?.({ nodeId: node.id }, 'Skipping embedding generation (missing OPENAI_API_KEY)');
    return;
  }

  if (!options.force && shouldSkipEmbeddingUpdate(node, options.updatedFields)) {
    ctx.logger?.debug?.({ nodeId: node.id }, 'Skipping embedding generation (no meaningful fields changed)');
    return;
  }

  const text = buildEmbeddingText(node);
  if (!text.trim()) {
    ctx.logger?.debug?.({ nodeId: node.id }, 'Skipping embedding generation (empty text)');
    return;
  }

  const maxRetries = options.maxRetries ?? 3;
  let lastError: unknown;

  for (let attempt = 0; attempt < maxRetries; attempt++) {
    let session;
    try {
      const openai = new OpenAI({
        apiKey,
      });

      // API Call with error handling
      let response;
      try {
        response = await openai.embeddings.create({
          model: ctx.env.EMBEDDING_MODEL || DEFAULT_MODEL,
          input: text,
        });
      } catch (apiError) {
        const apiMsg = apiError instanceof Error ? apiError.message : String(apiError);
        throw new Error(`OpenAI API call failed: ${apiMsg}`);
      }

      // Validate API response
      if (!response || !response.data || response.data.length === 0) {
        throw new Error('OpenAI API returned empty response');
      }

      const embedding = response.data[0]?.embedding;
      if (!embedding || !Array.isArray(embedding) || embedding.length === 0) {
        ctx.logger?.warn?.(
          { nodeId: node.id, responseLength: response.data.length },
          'Invalid embedding data returned from OpenAI API'
        );
        return; // Don't retry - API returned invalid data
      }

      // Store the embedding in Neo4j with error handling
      session = driver.session({ database: process.env.NEO4J_DATABASE || ctx.env.NEO4J_DATABASE || undefined });
      try {
        const result = await session.run(
          `
            MATCH (n {id: $nodeId})
            SET n.embedding = $embedding,
                n.embeddingUpdatedAt = datetime()
            RETURN size($embedding) AS dimensions
          `,
          {
            nodeId: node.id,
            embedding,
          }
        );

        if (!result.records || result.records.length === 0) {
          ctx.logger?.warn?.({ nodeId: node.id }, 'Node not found in database, embedding not stored');
          return; // Node doesn't exist, don't retry
        }

        const dimensions = result.records[0]?.get('dimensions');
        ctx.logger?.debug?.(
          { nodeId: node.id, dimensions, attempt: attempt + 1 },
          'Embedding generated and stored successfully'
        );
        return; // Success!
      } catch (dbError) {
        const dbMsg = dbError instanceof Error ? dbError.message : String(dbError);
        throw new Error(`Neo4j write failed: ${dbMsg}`);
      }
    } catch (error) {
      lastError = error;

      const isLast = attempt === maxRetries - 1;
      const shouldRetry = isRetriableError(error);
      const errorMsg = error instanceof Error ? error.message : String(error);

      if (isLast || !shouldRetry) {
        ctx.logger?.warn?.(
          {
            nodeId: node.id,
            error: errorMsg,
            attempt: attempt + 1,
            retriable: shouldRetry,
            maxRetries,
          },
          'Failed to generate embedding (giving up)'
        );
        return; // Give up gracefully
      }

      // Exponential backoff: 1s, 2s, 4s
      const backoffMs = 1000 * Math.pow(2, attempt);
      ctx.logger?.debug?.(
        { nodeId: node.id, attempt: attempt + 1, backoffMs, error: errorMsg },
        'Retrying embedding generation after error'
      );
      await sleep(backoffMs);
    } finally {
      // Ensure session is closed even if an error occurs
      if (session) {
        try {
          await session.close();
        } catch (closeError) {
          ctx.logger?.debug?.(
            { nodeId: node.id, error: closeError instanceof Error ? closeError.message : String(closeError) },
            'Error closing Neo4j session'
          );
        }
      }
    }
  }

  // All retries exhausted
  if (lastError) {
    ctx.logger?.warn?.(
      {
        nodeId: node.id,
        error: lastError instanceof Error ? lastError.message : String(lastError),
        totalAttempts: maxRetries,
      },
      'Embedding generation failed after all retries'
    );
  }
}
```

---

## 8. Defensive RankResults (personalizer.ts)

### Key sections to add to `rankResults`:

```typescript
// Input validation with graceful degradation
if (!results || !Array.isArray(results) || results.length === 0) {
  return [];
}

if (!context) {
  // Return results without personalization if context is invalid
  return results.map((result, index) => ({
    nodeId: result.node?.id || `unknown-${index}`,
    baseScore: result.score ?? 0,
    personalizedScore: result.score ?? 0,
    boosts: {},
    metadata: {},
  }));
}

return results
  .map((result) => {
    try {
      if (!result || !result.node || !result.node.id) {
        return null;
      }

      let personalizedScore = result.score ?? 0;
      const boosts: PersonalizedSearchResult["boosts"] = {};

      // Wrap each boost in try-catch
      try {
        const recencyBoost = this.applyRecencyBoost(result.node.id, context);
        if (recencyBoost > 1) {
          personalizedScore *= recencyBoost;
          boosts.recencyBoost = recencyBoost;
        }
      } catch (recencyError) {
        console.warn("Error applying recency boost:", recencyError instanceof Error ? recencyError.message : String(recencyError));
      }

      // ... repeat for other boosts ...

      return {
        nodeId: result.node.id,
        baseScore: result.score ?? 0,
        personalizedScore: Math.max(0, personalizedScore),
        boosts,
        metadata: {
          lastAccessedAt: this.behaviorStore.getLastAccessTime(result.node.id),
          accessCount: this.behaviorStore.getAccessCount(result.node.id),
        },
      };
    } catch (error) {
      console.warn("Error processing result:", error instanceof Error ? error.message : String(error));
      return null;
    }
  })
  .filter((r) => r !== null) as PersonalizedSearchResult[]
  .sort((a, b) => {
    try {
      return (b.personalizedScore ?? 0) - (a.personalizedScore ?? 0);
    } catch {
      return 0;
    }
  });
```

---

## 9. Safe Boost Functions (personalizer.ts)

### Template for each boost function:

```typescript
/**
 * Boost based on [DESCRIPTION]
 * Gracefully returns neutral boost on error
 */
private applyXxxBoost(input: Type, context: UserContext): number {
  try {
    // Validate inputs
    if (!context?.property || !input?.field) {
      return 1.0;
    }

    // Apply boost logic
    const boost = calculateBoost(...);

    // Ensure boost is in reasonable range
    return Math.max(1.0, Math.min(maxBoost, boost));
  } catch (error) {
    return 1.0; // Return neutral boost on any error
  }
}
```

---

## Key Takeaways

1. **Always wrap external API calls** in try-catch
2. **Validate inputs** at function entry points
3. **Prefer partial results** over no results
4. **Log errors with context** for debugging
5. **Clean up resources** in finally blocks
6. **Implement fallbacks** at multiple levels
7. **Distinguish temporary from permanent errors** for retry logic
8. **Return neutral/safe values** from helper functions on error
9. **Never let errors propagate silently**
10. **Test error scenarios** before deployment
