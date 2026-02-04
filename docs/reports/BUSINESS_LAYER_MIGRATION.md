# Business Layer Migration Plan

The Vision layer now distinguishes between **aspirational visions** and the **business entities** that operationalize them. Business nodes capture revenue models, stages, markets, integrations, and can be linked to both constitutional values and measurable goals.

This document explains how to migrate existing data that previously stored business context inside `Vision` nodes.

## 1. Data Model Changes

| Concept                                              | Old Location                            | New Location                                                   |
| ---------------------------------------------------- | --------------------------------------- | -------------------------------------------------------------- |
| Aspirational outcomes (e.g., "Generational Freedom") | Vision node                             | Vision node                                                    |
| Operating companies / business lines                 | Vision node                             | **Business node** (new type)                                   |
| Goals that belong to a specific business             | Direct `Goal → Vision` links            | `Business HAS_GOAL Goal` (still optional `Goal SERVES Vision`) |
| Constitutional alignment                             | `Goal ALIGNS_WITH Value/Principle` only | `Business ALIGNS_WITH Value/Principle` + prior Goal links      |

New relationships:

- `Vision ENABLES Business`
- `Business HAS_GOAL Goal`
- `Business ALIGNS_WITH Value/Principle`

## 2. Migration Steps

1. **List candidate nodes**

   ```javascript
   // Identify legacy Vision nodes that describe businesses
   callTool('odei.neo4j.vision.list.v2', {
     filters: { status: 'active', types: ['Vision'] },
     includeHierarchy: true,
   });
   ```

   Look for entries such as "Automated Exchange" or "Tipz CBS" whose summaries describe an operating company.

2. **Create Business nodes**

   ```javascript
   callTool('odei.neo4j.node.create.v1', {
     layer: 'vision',
     type: 'Business',
     data: {
       title: 'Tipz CBS',
       summary: 'Sovereign banking infrastructure for CBS partnerships',
       description: '3-year plan…',
       businessModel: 'Equity-backed co-founder build',
       stage: 'building',
       markets: ['UAE', 'KSA'],
       competitors: ['Tribe', 'Aspire'],
       timeline: {
         start: '2025-11-01T00:00:00Z',
       },
     },
     provenance: { module: 'decisions', actor: 'human', source: 'migration', confidence: 0.9 },
   });
   ```

   Repeat for each legacy Vision node that truly represents a business.

3. **Wire Vision → Business**

   ```javascript
   callTool('odei.neo4j.relationship.create.v1', {
     type: 'ENABLES',
     fromId: '<vision-id>',
     toId: '<business-id>',
     provenance: { module: 'decisions', actor: 'human', source: 'migration', confidence: 0.9 },
   });
   ```

   Use the life vision ("Generational Freedom & Legacy") as the parent for recurring businesses.

4. **Attach Goals to Business**
   For each goal that previously pointed directly to the business vision:

   ```javascript
   callTool('odei.neo4j.relationship.create.v1', {
     type: 'HAS_GOAL',
     fromId: '<business-id>',
     toId: '<goal-id>',
     provenance: { module: 'decisions', actor: 'human', source: 'migration', confidence: 0.9 },
   });
   ```

   Keep existing `Goal SERVES Vision` edges so long-term traceability remains intact.

5. **Restore constitutional alignment**

   ```javascript
   callTool('odei.neo4j.relationship.create.v1', {
     type: 'ALIGNS_WITH',
     fromId: '<business-id>',
     toId: '<value-or-principle-id>',
     weight: 0.9,
     provenance: { module: 'decisions', actor: 'human', source: 'migration', confidence: 0.9 },
   });
   ```

   Typical targets: `Value:Reality_Based_Decisions`, `Principle:Generational_Trauma_Stops_Here`, etc.

6. **Archive redundant Vision nodes**
   After a Business node mirrors the legacy Vision entry, set the old Vision node to `status: 'archived'` or repurpose it as a high-level Vision document.

## 3. Verification Checklist

1. `callTool('odei.neo4j.vision.list.v2', { filters: { types: ['Business'] }})` returns two active nodes (Tipz CBS, Automated Exchange).
2. Each Business has at least one `ENABLES` parent Vision and one `HAS_GOAL` child goal.
3. `odei.neo4j.schema.inventory.v1` shows Business under the Vision layer with the new relationships.
4. `odei.neo4j.hybrid.search.v1` returns Business nodes when querying for "Tipz" or "Exchange".

## 4. Automation via Script

Instead of running Cypher manually you can use the helper script that lives inside `servers/odei-neo4j`:

1. **Review / update the plan** — edit `scripts/business-migration.plan.json` (repo root) with the source vision titles/ids and the Business metadata you want to promote.
2. **Dry run first**
   ```bash
   cd servers/odei-neo4j
   npm run build          # ensures latest dist scripts
   npm run migrate:business:dry
   ```
   The dry run prints which Vision nodes would be migrated without touching Neo4j.
3. **Execute migration**
   ```bash
   npm run migrate:business
   ```
   The script will:
   - create/update the `Business` node with the provided metadata
   - link the parent Vision via `ENABLES`
   - attach every `(Goal)-[:SERVES]->(Vision)` as `(Business)-[:HAS_GOAL]->(Goal)`
   - copy existing Value/Principle alignments onto the Business
   - archive the legacy Vision node (configurable per plan)

Environment variables `NEO4J_URI`, `NEO4J_USERNAME`, `NEO4J_PASSWORD`, and `NEO4J_DATABASE` override the defaults if needed.

## 4. Rollback

If anything misbehaves:

- Delete the `Business` nodes (idempotent) and re-activate the archived Vision nodes.
- Remove the new relationships via `odei.neo4j.relationship.delete.v1`.
- Re-run tests: `npm run test-hybrid-search` inside `servers/odei-neo4j`.

Documented: November 2025
