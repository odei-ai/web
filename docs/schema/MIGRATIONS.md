# ODEI Schema Migrations & Evolution

**Version:** 1.0
**Generated:** 2025-12-25

---

## Table of Contents

1. [Overview](#overview)
2. [Adding New Node Types](#adding-new-node-types)
3. [Adding New Relationships](#adding-new-relationships)
4. [Modifying Existing Schema](#modifying-existing-schema)
5. [Backwards Compatibility](#backwards-compatibility)
6. [Migration Workflow](#migration-workflow)
7. [Testing Strategy](#testing-strategy)

---

## Overview

The ODEI graph schema is **code-defined** and **version-controlled**. Schema changes propagate through:

1. **Source code** — TypeScript type definitions in `servers/odei-neo4j/src/domain/`
2. **Guardian validation** — Zod schemas enforce constraints
3. **Neo4j constraints** — Database-level uniqueness and existence constraints
4. **Schema inventory** — Auto-generated documentation via `schema:export`

**Key Principle:** Schema changes must be **additive** whenever possible. Breaking changes require migration scripts.

---

## Adding New Node Types

### Step 1: Define Zod Schema

Add the new node type schema in `/Users/ai/ODEI/servers/odei-neo4j/src/domain/nodeTypes.ts`:

```typescript
// Example: Adding a "Feature" node to tactics layer
const TacticsFeatureSchema = BaseNodeCommonSchema.extend({
  priority: z.enum(['p0', 'p1', 'p2', 'p3']).optional(),
  estimatedHours: z.number().min(0).optional(),
  completionCriteria: z.string().max(1000).optional(),
  assignee: z.string().optional(),
});

type TacticsFeature = z.infer<typeof TacticsFeatureSchema>;
```

**Required fields:**
- All nodes extend `BaseNodeCommonSchema` (id, title, summary, status, etc.)
- `summary` is **required** (enforced by APOC trigger in Neo4j)

### Step 2: Register Node Type

Add the node type to the registry in the same file:

```typescript
const registry: NodeTypeConfig[] = [
  // ... existing types
  defineNode('Feature', 'tactics', TacticsFeatureSchema),
];
```

**defineNode signature:**
```typescript
function defineNode(
  name: string,           // Type name (PascalCase, e.g., "Feature")
  layer: Layer,           // Layer: "foundation" | "vision" | "strategy" | "tactics" | "execution" | "track" | "mind"
  schema: ZodTypeAny      // Zod schema
): NodeTypeConfig
```

### Step 3: Add Relationships

Define valid relationships in `/Users/ai/ODEI/servers/odei-neo4j/src/domain/relationships.ts`:

```typescript
const relationshipConfigs: RelationshipConfig[] = [
  // ... existing relationships
  {
    type: 'HAS_FEATURE',
    from: { layer: 'tactics', types: ['Project'] },
    to: { layer: 'tactics', types: ['Feature'] },
    directional: true,
    description: 'Project contains features',
  },
  {
    type: 'BLOCKS',
    from: { layer: 'tactics', types: ['Feature'] },
    to: { layer: 'tactics', types: ['Feature'] },
    directional: true,
    description: 'Feature blocks another feature',
    propertiesSchema: z.object({
      reason: z.string().optional(),
    }),
  },
];
```

### Step 4: Rebuild and Export Schema

```bash
cd /Users/ai/ODEI/servers/odei-neo4j
npm run build
npm run schema:export
```

This regenerates:
- `/Users/ai/ODEI/.claude/SCHEMA.md` — Auto-generated schema docs
- Type definitions and validation in `dist/`

### Step 5: Create Tool Handler (Optional)

If you want dedicated create/update tools:

```typescript
// /Users/ai/ODEI/servers/odei-neo4j/src/tools/tacticsFeature.ts
import { NodeCreateParamsSchema, NodeUpdateParamsSchema } from '../domain/constants.js';

export const tacticsFeatureCreate = {
  name: 'odei.neo4j.feature.create.v1',
  description: 'Create a tactics Feature node',
  inputSchema: NodeCreateParamsSchema.extend({
    data: TacticsFeatureSchema.partial({ id: true }),
  }),
  handler: async (context, params) => {
    return createNode(context, 'Feature', params);
  },
};
```

### Step 6: Test

```bash
# Build
npm run build

# Test node creation
node dist/index.js <<EOF
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "odei.neo4j.tactics.create.v1",
  "params": {
    "type": "feature",
    "data": {
      "title": "Dark Mode Toggle",
      "summary": "Add dark mode toggle to settings panel",
      "priority": "p1",
      "estimatedHours": 6
    },
    "provenance": {
      "module": "plan",
      "actor": "human",
      "source": "test",
      "confidence": 1.0
    }
  }
}
EOF
```

---

## Adding New Relationships

### Step 1: Define Relationship Config

In `/Users/ai/ODEI/servers/odei-neo4j/src/domain/relationships.ts`:

```typescript
{
  type: 'DEPENDS_ON',                        // Relationship type (UPPERCASE_SNAKE_CASE)
  from: {
    layer: 'execution',                      // Source layer
    types: ['Task']                          // Allowed source node types
  },
  to: {
    layer: 'execution',                      // Target layer
    types: ['Task']                          // Allowed target node types
  },
  directional: true,                         // Is this a directed relationship?
  cardinality: 'N:M',                        // One-to-many, many-to-many, etc.
  requiredProperties: [],                    // Required properties on the relationship
  optionalProperties: [],                    // Optional properties
  propertiesSchema: z.object({               // Zod schema for relationship properties
    blocking: z.boolean().optional(),
    criticality: z.enum(['low', 'medium', 'high']).optional(),
  }).optional(),
  description: 'Task depends on another task completing',
  semantics: 'Use DEPENDS_ON to create hard dependencies between tasks',
  examples: [
    'Task:implement_auth → Task:design_auth: Auth implementation depends on design'
  ],
}
```

### Step 2: Add Validation Rules (Optional)

In `/Users/ai/ODEI/servers/odei-neo4j/src/guardian/relationshipRules.ts`:

```typescript
const REQUIRED_RELATIONSHIP_RULES: Record<string, RequiredRelationshipRule> = {
  // ... existing rules
  task: {
    typeNames: ['Project'],                  // Task must link to at least one Project
    minimumActiveLinks: 1,
    allowedRelationshipTypes: ['HAS_TASK', 'SERVES'],
  },
};
```

### Step 3: Rebuild and Test

```bash
npm run build
npm run schema:export
```

Test relationship creation:

```bash
node dist/index.js <<EOF
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "odei.neo4j.relationship.create.v1",
  "params": {
    "type": "DEPENDS_ON",
    "fromId": "task_1",
    "toId": "task_2",
    "properties": {
      "blocking": true,
      "criticality": "high"
    },
    "provenance": {
      "module": "execute",
      "actor": "human",
      "source": "test",
      "confidence": 1.0
    }
  }
}
EOF
```

---

## Modifying Existing Schema

### Safe Changes (Non-Breaking)

These changes are backwards-compatible:

✅ **Adding optional fields**
```typescript
// Before
const GoalSchema = BaseNodeCommonSchema.extend({
  horizon: z.enum(['week', 'month', 'quarter', 'year']).optional(),
});

// After (add new optional field)
const GoalSchema = BaseNodeCommonSchema.extend({
  horizon: z.enum(['week', 'month', 'quarter', 'year']).optional(),
  sponsor: z.string().optional(),  // ✅ NEW OPTIONAL FIELD
});
```

✅ **Adding new relationship types**
```typescript
// New relationships don't break existing queries
{
  type: 'NEW_RELATIONSHIP',
  from: { layer: 'vision', types: ['Goal'] },
  to: { layer: 'strategy', types: ['Objective'] },
}
```

✅ **Adding new node types**
```typescript
// New node types don't affect existing nodes
defineNode('NewType', 'tactics', NewTypeSchema);
```

✅ **Expanding enums** (additive only)
```typescript
// Before
status: z.enum(['draft', 'active', 'archived'])

// After (add new option)
status: z.enum(['draft', 'active', 'archived', 'deprecated'])  // ✅ OK
```

### Breaking Changes (Require Migration)

These changes require migration scripts:

❌ **Making optional fields required**
```typescript
// Before
sponsor: z.string().optional()

// After (BREAKING)
sponsor: z.string()  // ❌ Existing nodes without sponsor will fail validation
```

**Migration needed:**
```cypher
// Set default value for missing fields
MATCH (n:Goal)
WHERE n.sponsor IS NULL
SET n.sponsor = 'unspecified'
```

❌ **Removing fields**
```typescript
// Before
completionCriteria: z.string().optional()

// After (BREAKING)
// completionCriteria removed  // ❌ Data loss
```

**Migration needed:**
```cypher
// Archive field value to metadata
MATCH (n:Goal)
WHERE n.completionCriteria IS NOT NULL
SET n.metadata.archivedCompletionCriteria = n.completionCriteria
REMOVE n.completionCriteria
```

❌ **Renaming fields**
```typescript
// Before
estimatedHours: z.number()

// After (BREAKING)
effortHours: z.number()  // ❌ Field rename
```

**Migration needed:**
```cypher
// Copy old field to new field
MATCH (n:Task)
WHERE n.estimatedHours IS NOT NULL
SET n.effortHours = n.estimatedHours
REMOVE n.estimatedHours
```

❌ **Changing field types**
```typescript
// Before
priority: z.string()

// After (BREAKING)
priority: z.enum(['p0', 'p1', 'p2', 'p3'])  // ❌ Type change
```

**Migration needed:**
```cypher
// Normalize priority values
MATCH (n:Task)
WHERE n.priority IS NOT NULL AND n.priority NOT IN ['p0', 'p1', 'p2', 'p3']
SET n.priority = CASE
  WHEN n.priority IN ['urgent', 'critical'] THEN 'p0'
  WHEN n.priority IN ['high'] THEN 'p1'
  WHEN n.priority IN ['medium'] THEN 'p2'
  ELSE 'p3'
END
```

---

## Backwards Compatibility

### Versioning Strategy

**Schema version** is tracked in `schemaInventory.ts`:

```typescript
export const SCHEMA_VERSION = '1.3.0';
```

**Version format:** `MAJOR.MINOR.PATCH`

- **MAJOR** — Breaking changes (requires migration)
- **MINOR** — New features (backwards-compatible)
- **PATCH** — Bug fixes (backwards-compatible)

### Deprecation Process

When removing a field or node type:

1. **Mark as deprecated** (don't remove yet)
```typescript
// Add deprecation notice in schema
const OldFieldSchema = z.object({
  oldField: z.string().optional(),  // DEPRECATED: Use newField instead
  newField: z.string().optional(),
});
```

2. **Add migration script**
```cypher
// Migrate old data to new format
MATCH (n)
WHERE n.oldField IS NOT NULL AND n.newField IS NULL
SET n.newField = n.oldField
```

3. **Update documentation**
```markdown
## Deprecated Fields
- `oldField` (deprecated in v1.3.0, removed in v2.0.0) — Use `newField` instead
```

4. **Remove in next major version**
```typescript
// v2.0.0 — oldField removed
const NewFieldSchema = z.object({
  newField: z.string().optional(),
});
```

---

## Migration Workflow

### 1. Write Migration Script

Create in `/Users/ai/ODEI/servers/odei-neo4j/migrations/`:

```cypher
// migrations/001-add-feature-node-type.cypher

// Step 1: Add any required constraints
CREATE CONSTRAINT feature_id_unique IF NOT EXISTS
FOR (n:Feature) REQUIRE n.id IS UNIQUE;

// Step 2: Add indexes for performance
CREATE INDEX feature_status IF NOT EXISTS
FOR (n:Feature) ON (n.status);

// Step 3: Migrate existing data (if needed)
MATCH (n)
WHERE n.type = 'old_feature'
SET n.type = 'feature',
    n:Feature
REMOVE n:OldFeature;

// Step 4: Validate migration
MATCH (n:Feature)
WHERE n.summary IS NULL
RETURN count(n) AS invalid_nodes;  // Should be 0
```

### 2. Test Migration

```bash
# Apply migration to test database
cat migrations/001-add-feature-node-type.cypher | \
  cypher-shell -u neo4j -p password --database test
```

### 3. Backup Production Database

```bash
cd /Users/ai/ODEI/servers/odei-neo4j
npm run backup:neo4j -- --dir backups/pre-migration-$(date +%Y%m%d)
```

### 4. Apply Migration to Production

```bash
# Apply migration
cat migrations/001-add-feature-node-type.cypher | \
  cypher-shell -u neo4j -p password --database memory

# Verify
cypher-shell -u neo4j -p password --database memory \
  "MATCH (n:Feature) RETURN count(n) AS feature_count"
```

### 5. Update Schema Version

```typescript
// servers/odei-neo4j/src/domain/schemaInventory.ts
export const SCHEMA_VERSION = '1.4.0';  // Increment version
```

### 6. Regenerate Documentation

```bash
npm run build
npm run schema:export
```

---

## Testing Strategy

### Unit Tests

Test schema validation:

```typescript
// test/schema/nodeTypes.test.ts
import { TacticsFeatureSchema } from '../src/domain/nodeTypes.js';

test('Feature schema validates correctly', () => {
  const validFeature = {
    id: 'feat_123',
    title: 'Dark Mode Toggle',
    summary: 'Add dark mode toggle to settings',
    priority: 'p1',
    estimatedHours: 6,
  };

  expect(() => TacticsFeatureSchema.parse(validFeature)).not.toThrow();
});

test('Feature schema rejects invalid priority', () => {
  const invalidFeature = {
    title: 'Dark Mode Toggle',
    summary: 'Add dark mode toggle',
    priority: 'invalid',  // Not in enum
  };

  expect(() => TacticsFeatureSchema.parse(invalidFeature)).toThrow();
});
```

### Integration Tests

Test end-to-end node creation:

```typescript
// test/integration/featureCreate.test.ts
test('Creates Feature node with valid provenance', async () => {
  const response = await client.call('odei.neo4j.feature.create.v1', {
    data: {
      title: 'Dark Mode Toggle',
      summary: 'Add dark mode toggle to settings panel',
      priority: 'p1',
    },
    provenance: {
      module: 'plan',
      actor: 'human',
      source: 'test',
      confidence: 1.0,
    },
  });

  expect(response.result.id).toBeDefined();
  expect(response.result.type).toBe('feature');
});
```

### Migration Tests

Test migration scripts:

```typescript
// test/migrations/001-add-feature.test.ts
test('Migration adds Feature node type', async () => {
  // Apply migration
  await runMigration('001-add-feature-node-type.cypher');

  // Verify constraint exists
  const constraints = await session.run(
    'SHOW CONSTRAINTS WHERE name = "feature_id_unique"'
  );
  expect(constraints.records.length).toBe(1);

  // Verify index exists
  const indexes = await session.run(
    'SHOW INDEXES WHERE name = "feature_status"'
  );
  expect(indexes.records.length).toBe(1);
});
```

---

## Common Migration Patterns

### Pattern 1: Add Optional Field to Existing Type

```cypher
// No migration needed (field is optional)
// Just update schema and rebuild
```

```typescript
// Before
const GoalSchema = BaseNodeCommonSchema.extend({
  horizon: z.enum(['week', 'month', 'quarter', 'year']).optional(),
});

// After
const GoalSchema = BaseNodeCommonSchema.extend({
  horizon: z.enum(['week', 'month', 'quarter', 'year']).optional(),
  sponsor: z.string().optional(),  // NEW FIELD
});
```

### Pattern 2: Rename Field

```cypher
// Migration: Copy old field to new field
MATCH (n:Task)
WHERE n.estimatedHours IS NOT NULL
SET n.effortHours = n.estimatedHours;

// Remove old field after validation
MATCH (n:Task)
WHERE n.effortHours IS NOT NULL
REMOVE n.estimatedHours;
```

### Pattern 3: Split Node Type

```cypher
// Migration: Create new node types from existing
MATCH (n:OldType)
WHERE n.category = 'feature'
CREATE (f:Feature)
SET f = properties(n),
    f.type = 'feature'
CREATE (n)-[:MIGRATED_TO]->(f);

// After validation, remove old nodes
MATCH (n:OldType)-[:MIGRATED_TO]->(f:Feature)
DETACH DELETE n;
```

### Pattern 4: Add Required Relationship

```cypher
// Migration: Create missing relationships
MATCH (t:Task)
WHERE NOT (t)-[:HAS_TASK]-(:Project)
  AND NOT (t)-[:SERVES]->(:Goal)
MATCH (p:Project {title: 'Unorganized Tasks'})
MERGE (p)-[:HAS_TASK]->(t);
```

### Pattern 5: Normalize Enum Values

```cypher
// Migration: Normalize priority values
MATCH (n:Task)
WHERE n.priority IS NOT NULL
SET n.priority = CASE
  WHEN toLower(n.priority) IN ['urgent', 'critical', 'p0'] THEN 'p0'
  WHEN toLower(n.priority) IN ['high', 'p1'] THEN 'p1'
  WHEN toLower(n.priority) IN ['medium', 'normal', 'p2'] THEN 'p2'
  WHEN toLower(n.priority) IN ['low', 'p3'] THEN 'p3'
  ELSE 'p3'
END;
```

---

## Rollback Strategy

### Before Migration

```bash
# Create timestamped backup
npm run backup:neo4j -- --dir backups/pre-migration-$(date +%Y%m%d_%H%M%S)
```

### If Migration Fails

```bash
# Restore from backup
npm run restore:neo4j -- --file backups/pre-migration-20251225_143022/backup.json
```

### Partial Rollback

```cypher
// Rollback specific changes (example: undo field rename)
MATCH (n:Task)
WHERE n.effortHours IS NOT NULL AND n.estimatedHours IS NULL
SET n.estimatedHours = n.effortHours
REMOVE n.effortHours;
```

---

## Best Practices

### ✅ DO

1. **Test migrations on copy of production data**
2. **Backup before every migration**
3. **Use transactions for atomic migrations**
4. **Write idempotent migration scripts** (can run multiple times)
5. **Version all migrations** (use sequential numbering)
6. **Document breaking changes** in CHANGELOG.md

### ❌ DON'T

1. **Don't skip backups** — Always backup before migration
2. **Don't modify production directly** — Test in staging first
3. **Don't remove fields without migration** — Data loss
4. **Don't change field types without mapping** — Type errors
5. **Don't run partial migrations** — Use transactions for atomicity

---

**Next:** See [RELATIONSHIPS.md](./RELATIONSHIPS.md) for complete relationship reference.
