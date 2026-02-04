# Health Dashboard Implementation - ULTRATHINK

## Complete Architecture & Real Data Integration

> **Co-founder Task:** Replace unused Companion panel with live Health Dashboard showing real temporal health, goals, and memory coverage from Neo4j graph.

---

## 🎯 IMPLEMENTATION STRATEGY OVERVIEW

### The Challenge

Dashboard needs **REAL DATA** from Neo4j, not mock data:

- Health score (0-100) calculated from actual graph state
- Active goals with real deadlines from Vision layer
- Memory coverage across 7 actual layers
- Live issues detected from temporal inconsistencies
- Auto-refresh every 5 minutes

### The Architecture Chain

```
Neo4j Graph (31 node types)
    ↓
odei-neo4j MCP Server (new tools)
    ↓
HealthProbes (MCP subprocess manager)
    ↓
Electron IPC (data:get channel)
    ↓
Preload Bridge (window.odei.data.get)
    ↓
Renderer.js (Health Dashboard UI)
    ↓
DOM (#health-dashboard-panel)
```

### Implementation Layers

1. **Data Layer** - New Neo4j queries for health metrics
2. **Service Layer** - New MCP tools exposing health data
3. **Transport Layer** - Existing IPC infrastructure (reuse)
4. **UI Layer** - New React-like component rendering health widgets

---

## 📊 PART 1: DATA LAYER - NEO4J QUERIES

### Query 1: Temporal Health Context

**Purpose:** Calculate health score, find stale data, get current period

**Location:** `servers/odei-neo4j/src/services/temporalContext.ts`

```typescript
interface TemporalHealthContext {
  healthScore: number; // 0-100
  grade: 'A' | 'B' | 'C' | 'D' | 'F';
  status: 'excellent' | 'good' | 'fair' | 'poor' | 'critical';
  currentTime: {
    utc: string; // ISO8601
    dubai: string; // ISO8601 (UTC+3)
    fiscal: {
      quarter: 'Q1' | 'Q2' | 'Q3' | 'Q4';
      year: number;
      week: number; // Week of year
    };
  };
  activeGoals: {
    quarter?: GoalSummary;
    month?: GoalSummary;
    week?: GoalSummary;
  };
  issues: TemporalIssue[];
  recommendations: string[];
}

interface GoalSummary {
  id: string;
  title: string;
  deadline: string; // ISO8601
  daysRemaining: number;
  urgency: 'critical' | 'warning' | 'normal';
  taskCount: number;
  completionPercent: number;
}

interface TemporalIssue {
  type: 'STALE_GOAL' | 'OVERDUE_TASK' | 'MISSING_DEADLINE' | 'FUTURE_COMPLETION' | 'LOW_COVERAGE';
  severity: 'error' | 'warning' | 'info';
  message: string;
  nodeId?: string;
  autoFixable: boolean;
  fixDescription?: string;
}
```

**Neo4j Query - Get Current Active Goals:**

```cypher
// Get current fiscal period
WITH datetime() as now,
     toInteger(((dayOfYear(datetime()) - 1) / 7) + 1) as weekNum,
     toInteger(((month(datetime()) - 1) / 3) + 1) as quarterNum

// Match active goals with their task counts
MATCH (g:Goal)
WHERE g.status = 'active'
  AND g.layer = 'vision'

// Get related tasks
OPTIONAL MATCH (g)<-[:SUPPORTS]-(t:Task)
WHERE t.status IN ['todo', 'in_progress', 'blocked']

// Calculate completion percentage
WITH g, now, weekNum, quarterNum,
     count(DISTINCT t) as taskCount,
     count(DISTINCT CASE WHEN t.status = 'done' THEN t END) as completedCount

// Filter by horizon
WITH g, now, weekNum, quarterNum, taskCount, completedCount,
     CASE
       WHEN g.properties.horizon = 'quarter' THEN 'Q' + toString(quarterNum) + ' ' + toString(year(now))
       WHEN g.properties.horizon = 'month' THEN ['January','February','March','April','May','June','July','August','September','October','November','December'][month(now)-1] + ' ' + toString(year(now))
       WHEN g.properties.horizon = 'week' THEN 'Week ' + toString(weekNum)
       ELSE null
     END as expectedTitle

// Only return goals matching current period
WHERE g.title CONTAINS expectedTitle
  OR (g.properties.deadline IS NOT NULL AND datetime(g.properties.deadline) >= now)

// Calculate days remaining
WITH g, taskCount, completedCount,
     duration.between(now, datetime(g.properties.deadline)).days as daysRemaining

// Map to output structure
RETURN {
  id: g.id,
  title: g.title,
  deadline: g.properties.deadline,
  daysRemaining: daysRemaining,
  urgency: CASE
    WHEN daysRemaining < 7 THEN 'critical'
    WHEN daysRemaining < 30 THEN 'warning'
    ELSE 'normal'
  END,
  taskCount: taskCount,
  completionPercent: CASE
    WHEN taskCount = 0 THEN 0
    ELSE toInteger((toFloat(completedCount) / toFloat(taskCount)) * 100)
  END,
  horizon: g.properties.horizon
} as goal
ORDER BY daysRemaining ASC
LIMIT 10
```

**Neo4j Query - Detect Stale Goals:**

```cypher
WITH datetime() as now

// Find goals that are active but their period has ended
MATCH (g:Goal)
WHERE g.status = 'active'
  AND g.layer = 'vision'
  AND g.properties.deadline IS NOT NULL
  AND datetime(g.properties.deadline) < now
  AND g.properties.completionStatus IS NULL  // Not marked as completed

RETURN {
  type: 'STALE_GOAL',
  severity: 'error',
  message: '"' + g.title + '" is marked active but deadline ' + g.properties.deadline + ' has passed',
  nodeId: g.id,
  autoFixable: true,
  fixDescription: 'Set status to archived or mark as completed'
} as issue

UNION

// Find tasks that are overdue
MATCH (t:Task)
WHERE t.status IN ['todo', 'in_progress']
  AND t.properties.dueDate IS NOT NULL
  AND datetime(t.properties.dueDate) < now

RETURN {
  type: 'OVERDUE_TASK',
  severity: 'warning',
  message: 'Task "' + t.title + '" is overdue by ' + toString(duration.between(datetime(t.properties.dueDate), now).days) + ' days',
  nodeId: t.id,
  autoFixable: false,
  fixDescription: 'Update dueDate or mark as blocked/done'
} as issue

UNION

// Find goals with completedAt in the future
MATCH (g:Goal)
WHERE g.properties.completedAt IS NOT NULL
  AND datetime(g.properties.completedAt) > now

RETURN {
  type: 'FUTURE_COMPLETION',
  severity: 'error',
  message: 'Goal "' + g.title + '" has completedAt in the future: ' + g.properties.completedAt,
  nodeId: g.id,
  autoFixable: true,
  fixDescription: 'Fix completedAt timestamp'
} as issue
```

**Neo4j Query - Calculate Health Score:**

```cypher
WITH datetime() as now

// Count total issues
MATCH (g:Goal)
WHERE g.status = 'active'
  AND g.properties.deadline IS NOT NULL
  AND datetime(g.properties.deadline) < now
  AND g.properties.completionStatus IS NULL
WITH count(g) as staleGoals

MATCH (t:Task)
WHERE t.status IN ['todo', 'in_progress']
  AND t.properties.dueDate IS NOT NULL
  AND datetime(t.properties.dueDate) < now
WITH staleGoals, count(t) as overdueTasks

MATCH (n:Goal)
WHERE n.properties.completedAt IS NOT NULL
  AND datetime(n.properties.completedAt) > datetime()
WITH staleGoals, overdueTasks, count(n) as futureCompletions

// Calculate deductions
WITH (staleGoals * 15) + (overdueTasks * 5) + (futureCompletions * 20) as totalDeductions

// Calculate score (max 100, min 0)
WITH CASE
  WHEN totalDeductions > 100 THEN 0
  ELSE 100 - totalDeductions
END as healthScore

RETURN {
  healthScore: healthScore,
  grade: CASE
    WHEN healthScore >= 90 THEN 'A'
    WHEN healthScore >= 80 THEN 'B'
    WHEN healthScore >= 70 THEN 'C'
    WHEN healthScore >= 60 THEN 'D'
    ELSE 'F'
  END,
  status: CASE
    WHEN healthScore >= 90 THEN 'excellent'
    WHEN healthScore >= 80 THEN 'good'
    WHEN healthScore >= 70 THEN 'fair'
    WHEN healthScore >= 60 THEN 'poor'
    ELSE 'critical'
  END,
  deductions: {
    staleGoals: staleGoals,
    overdueTasks: overdueTasks,
    futureCompletions: futureCompletions
  }
} as healthMetrics
```

### Query 2: Memory Coverage by Layer

**Purpose:** Show coverage percentage for each of 7 knowledge layers

**Location:** `servers/odei-neo4j/src/services/metaMemory.ts`

```typescript
interface MemoryCoverage {
  overall: number; // 0-100
  layers: {
    foundation: LayerCoverage;
    vision: LayerCoverage;
    strategy: LayerCoverage;
    tactics: LayerCoverage;
    execution: LayerCoverage;
    track: LayerCoverage;
    mind: LayerCoverage;
  };
}

interface LayerCoverage {
  percent: number; // 0-100
  nodeCount: number;
  relationshipCount: number;
  lastUpdated: string; // ISO8601
  coverage: 'excellent' | 'good' | 'fair' | 'low';
}
```

**Neo4j Query - Coverage by Layer:**

```cypher
// Define expected baseline for each layer (aspirational targets)
WITH {
  foundation: 15,    // Should have ~15+ foundation nodes
  vision: 10,        // Should have ~10+ active goals/visions
  strategy: 20,      // Should have ~20+ strategic elements
  tactics: 25,       // Should have ~25+ projects/areas
  execution: 50,     // Should have ~50+ tasks/decisions
  track: 30,         // Should have ~30+ metrics/observations
  mind: 25           // Should have ~25+ insights/patterns
} as baseline

// Count nodes by layer
MATCH (n)
WHERE n.status = 'active'
WITH baseline, n.layer as layer, count(n) as nodeCount

// Count relationships for each layer
MATCH (n)-[r]->(m)
WHERE n.layer = layer AND n.status = 'active'
WITH baseline, layer, nodeCount, count(r) as relCount

// Get most recent update
MATCH (n)
WHERE n.layer = layer
WITH baseline, layer, nodeCount, relCount,
     max(n.updatedAt) as lastUpdated

// Calculate coverage percentage (capped at 100%)
WITH layer, nodeCount, relCount, lastUpdated,
     CASE
       WHEN nodeCount >= baseline[layer] THEN 100
       ELSE toInteger((toFloat(nodeCount) / toFloat(baseline[layer])) * 100)
     END as percent

RETURN {
  layer: layer,
  percent: percent,
  nodeCount: nodeCount,
  relationshipCount: relCount,
  lastUpdated: lastUpdated,
  coverage: CASE
    WHEN percent >= 90 THEN 'excellent'
    WHEN percent >= 70 THEN 'good'
    WHEN percent >= 50 THEN 'fair'
    ELSE 'low'
  END
} as layerCoverage
ORDER BY layer
```

**Overall Coverage Calculation:**

```cypher
// Get all layer coverages
MATCH (n)
WHERE n.status = 'active'
WITH n.layer as layer, count(n) as nodeCount

WITH collect({layer: layer, count: nodeCount}) as layers

// Calculate weighted average (execution matters more than foundation)
WITH layers,
     {
       foundation: 0.8,
       vision: 1.2,
       strategy: 1.0,
       tactics: 1.0,
       execution: 1.5,
       track: 1.0,
       mind: 1.2
     } as weights

UNWIND layers as layerData
WITH layerData, weights,
     weights[layerData.layer] as weight,
     layerData.count as count

WITH sum(count * weight) as weightedSum,
     sum(weight) as totalWeight

RETURN toInteger((weightedSum / totalWeight)) as overallCoverage
```

---

## 🔧 PART 2: SERVICE LAYER - NEW MCP TOOLS

### Tool 1: `odei_neo4j_temporal_context_v1`

**File:** `servers/odei-neo4j/src/tools/temporalContext.ts`

```typescript
import { z } from 'zod';
import { ToolDefinition } from '../types';
import { Neo4jGraphService } from '../services/neo4jGraph';

const inputSchema = z.object({
  includeRecommendations: z.boolean().optional().default(true),
  timezone: z.string().optional().default('UTC'),
});

export const temporalContextTool: ToolDefinition = {
  name: 'odei_neo4j_temporal_context_v1',
  description: 'Get temporal health context including health score, active goals, and temporal issues',
  inputSchema: inputSchema,
  handler: async (input, { graph }) => {
    const now = new Date();
    const dubaiTime = new Date(now.getTime() + 3 * 60 * 60 * 1000); // UTC+3

    // Calculate fiscal period
    const quarter = Math.floor(now.getMonth() / 3) + 1;
    const weekOfYear = getWeekNumber(now);

    // Execute health score query
    const healthResult = await graph.runCypher(HEALTH_SCORE_QUERY);
    const healthMetrics = healthResult[0];

    // Execute active goals query
    const goalsResult = await graph.runCypher(ACTIVE_GOALS_QUERY, {
      now: now.toISOString(),
    });

    // Organize goals by horizon
    const activeGoals = {
      quarter: goalsResult.find((g) => g.horizon === 'quarter'),
      month: goalsResult.find((g) => g.horizon === 'month'),
      week: goalsResult.find((g) => g.horizon === 'week'),
    };

    // Execute issues detection query
    const issuesResult = await graph.runCypher(DETECT_ISSUES_QUERY);

    // Build response
    const context: TemporalHealthContext = {
      healthScore: healthMetrics.healthScore,
      grade: healthMetrics.grade,
      status: healthMetrics.status,
      currentTime: {
        utc: now.toISOString(),
        dubai: dubaiTime.toISOString(),
        fiscal: {
          quarter: `Q${quarter}` as any,
          year: now.getFullYear(),
          week: weekOfYear,
        },
      },
      activeGoals,
      issues: issuesResult,
      recommendations: input.includeRecommendations ? generateRecommendations(healthMetrics, issuesResult) : [],
    };

    return {
      content: [
        {
          type: 'text',
          text: JSON.stringify(context, null, 2),
        },
      ],
    };
  },
};

// Helper: Generate recommendations based on issues
function generateRecommendations(health: any, issues: TemporalIssue[]): string[] {
  const recommendations: string[] = [];

  if (health.healthScore < 70) {
    recommendations.push('Health score is below 70. Review and fix temporal issues.');
  }

  const staleGoals = issues.filter((i) => i.type === 'STALE_GOAL');
  if (staleGoals.length > 0) {
    recommendations.push(`Archive ${staleGoals.length} stale goal(s) that have passed their deadline.`);
  }

  const overdueTasks = issues.filter((i) => i.type === 'OVERDUE_TASK');
  if (overdueTasks.length > 3) {
    recommendations.push(`${overdueTasks.length} tasks are overdue. Consider reprioritizing or blocking.`);
  }

  return recommendations;
}

// Helper: Calculate week number
function getWeekNumber(date: Date): number {
  const start = new Date(date.getFullYear(), 0, 1);
  const diff = date.getTime() - start.getTime();
  const oneWeek = 1000 * 60 * 60 * 24 * 7;
  return Math.floor(diff / oneWeek) + 1;
}
```

### Tool 2: `odei_neo4j_memory_coverage_v1`

**File:** `servers/odei-neo4j/src/tools/memoryCoverage.ts`

```typescript
import { z } from 'zod';
import { ToolDefinition } from '../types';

const inputSchema = z.object({
  layers: z.array(z.enum(['foundation', 'vision', 'strategy', 'tactics', 'execution', 'track', 'mind'])).optional(),
});

export const memoryCoverageTool: ToolDefinition = {
  name: 'odei_neo4j_memory_coverage_v1',
  description: 'Get memory coverage statistics across knowledge layers',
  inputSchema: inputSchema,
  handler: async (input, { graph }) => {
    // Execute layer coverage query
    const layerResults = await graph.runCypher(LAYER_COVERAGE_QUERY);

    // Build layers object
    const layers = layerResults.reduce((acc, layer) => {
      acc[layer.layer] = {
        percent: layer.percent,
        nodeCount: layer.nodeCount,
        relationshipCount: layer.relationshipCount,
        lastUpdated: layer.lastUpdated,
        coverage: layer.coverage,
      };
      return acc;
    }, {} as any);

    // Calculate overall coverage
    const overallResult = await graph.runCypher(OVERALL_COVERAGE_QUERY);
    const overall = overallResult[0]?.overallCoverage || 0;

    const coverage: MemoryCoverage = {
      overall,
      layers,
    };

    return {
      content: [
        {
          type: 'text',
          text: JSON.stringify(coverage, null, 2),
        },
      ],
    };
  },
};
```

### Tool 3: `odei_neo4j_temporal_autofix_v1`

**File:** `servers/odei-neo4j/src/tools/temporalAutofix.ts`

```typescript
import { z } from 'zod';
import { ToolDefinition } from '../types';

const inputSchema = z.object({
  issueType: z.enum(['STALE_GOAL', 'FUTURE_COMPLETION', 'ALL']),
  dryRun: z.boolean().optional().default(true),
});

export const temporalAutofixTool: ToolDefinition = {
  name: 'odei_neo4j_temporal_autofix_v1',
  description: 'Automatically fix temporal issues in the graph',
  inputSchema: inputSchema,
  handler: async (input, { graph }) => {
    const fixes: any[] = [];

    if (input.issueType === 'STALE_GOAL' || input.issueType === 'ALL') {
      // Fix stale goals by archiving them
      const staleGoalQuery = `
        MATCH (g:Goal)
        WHERE g.status = 'active'
          AND g.properties.deadline IS NOT NULL
          AND datetime(g.properties.deadline) < datetime()
          AND g.properties.completionStatus IS NULL
        ${input.dryRun ? '' : 'SET g.status = "archived", g.updatedAt = datetime()'}
        RETURN g.id as id, g.title as title
      `;

      const staleGoals = await graph.runCypher(staleGoalQuery);
      fixes.push({
        type: 'STALE_GOAL',
        count: staleGoals.length,
        applied: !input.dryRun,
        items: staleGoals,
      });
    }

    if (input.issueType === 'FUTURE_COMPLETION' || input.issueType === 'ALL') {
      // Fix future completion dates by clearing them
      const futureCompletionQuery = `
        MATCH (g:Goal)
        WHERE g.properties.completedAt IS NOT NULL
          AND datetime(g.properties.completedAt) > datetime()
        ${input.dryRun ? '' : 'SET g.properties.completedAt = null, g.updatedAt = datetime()'}
        RETURN g.id as id, g.title as title
      `;

      const futureGoals = await graph.runCypher(futureCompletionQuery);
      fixes.push({
        type: 'FUTURE_COMPLETION',
        count: futureGoals.length,
        applied: !input.dryRun,
        items: futureGoals,
      });
    }

    return {
      content: [
        {
          type: 'text',
          text: JSON.stringify(
            {
              dryRun: input.dryRun,
              fixes,
              message: input.dryRun
                ? 'Dry run - no changes applied. Set dryRun=false to apply fixes.'
                : 'Fixes applied successfully.',
            },
            null,
            2
          ),
        },
      ],
    };
  },
};
```

### Register New Tools

**File:** `servers/odei-neo4j/src/tools/index.ts`

```typescript
// Add to existing tools export
export { temporalContextTool } from './temporalContext';
export { memoryCoverageTool } from './memoryCoverage';
export { temporalAutofixTool } from './temporalAutofix';

// Add to tools array in server initialization
const tools = [
  // ... existing tools
  temporalContextTool,
  memoryCoverageTool,
  temporalAutofixTool,
];
```

---

## 🌉 PART 3: TRANSPORT LAYER - IPC INTEGRATION

**Good news:** EXISTING IPC infrastructure already supports our needs!

### Existing IPC Channel: `data:get`

**Location:** `/Users/ai/ODEI/electron/main.js` (lines 533-596)

```javascript
// Already exists - NO CHANGES NEEDED
ipcMain.handle('data:get', async (event, request) => {
  const { server, tool, params } = request;

  // Validate with Zod
  const validated = DataGetRequestSchema.parse(request);

  // Call MCP server
  const response = await healthProbes.callTool(validated.server, validated.tool, validated.params);

  return response;
});
```

### Preload Bridge: `window.odei.data.get`

**Location:** `/Users/ai/ODEI/electron/preload.js` (lines 120-135)

```javascript
// Already exists - NO CHANGES NEEDED
contextBridge.exposeInMainWorld('odei', {
  data: {
    get: async (server, tool, params) => {
      return await ipcRenderer.invoke('data:get', {
        server,
        tool,
        params,
      });
    },
  },
  // ... other APIs
});
```

### Usage from Renderer

```javascript
// This pattern ALREADY WORKS - just use it!
const response = await window.odei.data.get(
  'odei-neo4j', // server
  'odei_neo4j_temporal_context_v1', // tool
  { includeRecommendations: true } // params
);

// Response format (standard MCP response)
{
  content: [
    {
      type: 'text',
      text: '{"healthScore": 92, "grade": "A", ...}',
    },
  ];
}

// Parse the response
const data = JSON.parse(response.content[0].text);
```

**No IPC changes needed! Transport layer is ready.**

---

## 🎨 PART 4: UI LAYER - RENDERER IMPLEMENTATION

### Step 1: Create Health Dashboard Module

**File:** `src/health-dashboard.js` (NEW)

```javascript
/**
 * Health Dashboard Module
 * Replaces unused Companion AI Consultation panel
 */

class HealthDashboard {
  constructor(containerId) {
    this.container = document.getElementById(containerId);
    this.data = null;
    this.refreshInterval = null;
  }

  /**
   * Initialize dashboard and start auto-refresh
   */
  async init() {
    console.log('[HealthDashboard] Initializing...');

    // Show loading state
    this.renderLoading();

    // Load initial data
    await this.refresh();

    // Start auto-refresh every 5 minutes
    this.refreshInterval = setInterval(
      () => {
        this.refresh();
      },
      5 * 60 * 1000
    );

    console.log('[HealthDashboard] Initialized successfully');
  }

  /**
   * Fetch fresh data from MCP servers
   */
  async refresh() {
    try {
      console.log('[HealthDashboard] Refreshing data...');

      // Call both MCP tools in parallel
      const [temporalResponse, coverageResponse] = await Promise.all([
        window.odei.data.get('odei-neo4j', 'odei_neo4j_temporal_context_v1', {
          includeRecommendations: true,
          timezone: 'UTC',
        }),
        window.odei.data.get('odei-neo4j', 'odei_neo4j_memory_coverage_v1', {}),
      ]);

      // Parse responses
      const temporal = JSON.parse(temporalResponse.content[0].text);
      const coverage = JSON.parse(coverageResponse.content[0].text);

      // Store data
      this.data = { temporal, coverage };

      // Render UI
      this.render();

      console.log('[HealthDashboard] Data refreshed successfully', {
        healthScore: temporal.healthScore,
        overallCoverage: coverage.overall,
      });
    } catch (error) {
      console.error('[HealthDashboard] Refresh failed:', error);
      this.renderError(error);
    }
  }

  /**
   * Auto-fix temporal issues
   */
  async autoFix() {
    try {
      const response = await window.odei.data.get('odei-neo4j', 'odei_neo4j_temporal_autofix_v1', {
        issueType: 'ALL',
        dryRun: false,
      });

      const result = JSON.parse(response.content[0].text);

      console.log('[HealthDashboard] Auto-fix completed:', result);

      // Show success toast
      this.showToast('Issues fixed successfully', 'success');

      // Refresh dashboard
      await this.refresh();
    } catch (error) {
      console.error('[HealthDashboard] Auto-fix failed:', error);
      this.showToast('Auto-fix failed', 'error');
    }
  }

  /**
   * Render complete dashboard
   */
  render() {
    if (!this.data) return;

    const { temporal, coverage } = this.data;

    this.container.innerHTML = `
      <div class="health-dashboard">
        ${this.renderHealthScore(temporal)}
        ${this.renderActiveGoals(temporal)}
        ${this.renderMemoryCoverage(coverage)}
        ${this.renderIssues(temporal)}
        ${this.renderRecommendations(temporal)}
        ${this.renderActions()}
      </div>
    `;

    // Attach event listeners
    this.attachEventListeners();

    // Trigger animations
    this.triggerAnimations();
  }

  /**
   * Render health score widget
   */
  renderHealthScore(temporal) {
    const { healthScore, grade, status, currentTime } = temporal;

    // Map status to color
    const colorMap = {
      excellent: '#10b981',
      good: '#3b82f6',
      fair: '#f59e0b',
      poor: '#f97316',
      critical: '#ef4444',
    };
    const color = colorMap[status];

    // SVG circle parameters
    const radius = 54;
    const circumference = 2 * Math.PI * radius;
    const offset = circumference - (healthScore / 100) * circumference;

    return `
      <div class="widget health-score-widget">
        <div class="health-score-circle">
          <svg width="120" height="120" viewBox="0 0 120 120">
            <circle
              class="score-ring-bg"
              cx="60"
              cy="60"
              r="${radius}"
              stroke="#374151"
              stroke-width="12"
              fill="none"
              opacity="0.2"
            />
            <circle
              class="score-ring"
              cx="60"
              cy="60"
              r="${radius}"
              stroke="${color}"
              stroke-width="12"
              fill="none"
              stroke-linecap="round"
              stroke-dasharray="${circumference}"
              stroke-dashoffset="${offset}"
              transform="rotate(-90 60 60)"
              style="transition: stroke-dashoffset 1s ease;"
            />
          </svg>
          <div class="score-text">${healthScore}</div>
        </div>
        <div class="health-info">
          <div class="health-grade">Grade: ${grade}</div>
          <div class="health-status" style="color: ${color}">
            ${status.charAt(0).toUpperCase() + status.slice(1)}
          </div>
        </div>
        <div class="current-time">
          ${new Date(currentTime.utc).toLocaleString('en-US', {
            weekday: 'short',
            year: 'numeric',
            month: 'short',
            day: 'numeric',
            hour: '2-digit',
            minute: '2-digit',
            timeZone: 'UTC',
          })} UTC
        </div>
        <div class="current-period">
          ${currentTime.fiscal.quarter} ${currentTime.fiscal.year}, Week ${currentTime.fiscal.week}
        </div>
      </div>
    `;
  }

  /**
   * Render active goals widget
   */
  renderActiveGoals(temporal) {
    const { activeGoals } = temporal;
    const goals = [activeGoals.quarter, activeGoals.month, activeGoals.week].filter(Boolean);

    if (goals.length === 0) {
      return `
        <div class="widget active-goals-widget">
          <div class="widget-header">Active Goals</div>
          <div class="empty-state">No active goals found</div>
        </div>
      `;
    }

    return `
      <div class="widget active-goals-widget">
        <div class="widget-header">Active Goals</div>
        ${goals.map((goal) => this.renderGoalCard(goal)).join('')}
      </div>
    `;
  }

  /**
   * Render single goal card
   */
  renderGoalCard(goal) {
    const urgencyColors = {
      critical: '#ef4444',
      warning: '#f59e0b',
      normal: '#6b7280',
    };
    const color = urgencyColors[goal.urgency];

    const deadlineEmoji = goal.urgency === 'critical' ? '🔴' : goal.urgency === 'warning' ? '⚠️' : '📅';

    return `
      <div class="goal-card" data-goal-id="${goal.id}">
        <div class="goal-title">${goal.title}</div>
        <div class="goal-deadline" style="color: ${color}">
          ${deadlineEmoji} ${new Date(goal.deadline).toLocaleDateString('en-US', {
            month: 'short',
            day: 'numeric',
            year: 'numeric',
          })} (${goal.daysRemaining} days)
        </div>
        <div class="goal-progress">
          <div class="progress-bar">
            <div class="progress-fill" style="width: ${goal.completionPercent}%"></div>
          </div>
          <div class="progress-text">${goal.completionPercent}% complete</div>
        </div>
      </div>
    `;
  }

  /**
   * Render memory coverage widget
   */
  renderMemoryCoverage(coverage) {
    const layers = [
      { key: 'foundation', label: 'Foundation' },
      { key: 'vision', label: 'Vision' },
      { key: 'strategy', label: 'Strategy' },
      { key: 'tactics', label: 'Tactics' },
      { key: 'execution', label: 'Execution' },
      { key: 'track', label: 'Track' },
      { key: 'mind', label: 'Mind' },
    ];

    return `
      <div class="widget memory-coverage-widget">
        <div class="widget-header">Memory Coverage</div>
        ${layers
          .map((layer) => {
            const data = coverage.layers[layer.key];
            if (!data) return '';

            const colorClass =
              data.coverage === 'excellent'
                ? 'excellent'
                : data.coverage === 'good'
                  ? 'good'
                  : data.coverage === 'fair'
                    ? 'fair'
                    : 'low';

            return `
            <div class="coverage-row">
              <div class="coverage-label">${layer.label}</div>
              <div class="coverage-bar-container">
                <div class="coverage-bar-fill ${colorClass}"
                     style="width: ${data.percent}%"
                     data-percent="${data.percent}">
                </div>
              </div>
              <div class="coverage-percent">${data.percent}%</div>
            </div>
          `;
          })
          .join('')}
        <div class="overall-coverage">
          Overall: ${coverage.overall}% coverage
        </div>
      </div>
    `;
  }

  /**
   * Render issues widget
   */
  renderIssues(temporal) {
    const { issues } = temporal;

    if (issues.length === 0) {
      return `
        <div class="widget issues-widget">
          <div class="collapsible-header">
            <span>✅ No Issues Found</span>
          </div>
        </div>
      `;
    }

    return `
      <div class="widget issues-widget">
        <div class="collapsible-header expanded" data-collapsible="issues">
          <span>
            <span class="collapsible-icon">▼</span>
            Issues & Alerts
            <span class="issue-count">${issues.length}</span>
          </span>
        </div>
        <div class="collapsible-content" id="issues-content">
          ${issues.map((issue) => this.renderIssueCard(issue)).join('')}
        </div>
      </div>
    `;
  }

  /**
   * Render single issue card
   */
  renderIssueCard(issue) {
    const severityClass = issue.severity;
    const icon = issue.severity === 'error' ? '🔴' : issue.severity === 'warning' ? '⚠️' : 'ℹ️';

    return `
      <div class="issue-card ${severityClass}">
        <div class="issue-type">
          ${icon} ${issue.type.replace(/_/g, ' ')}
        </div>
        <div class="issue-message">${issue.message}</div>
        <div class="issue-actions">
          ${
            issue.autoFixable
              ? `
            <button class="issue-button primary" data-action="autofix">
              Auto-Fix
            </button>
          `
              : ''
          }
          <button class="issue-button" data-action="dismiss" data-issue-id="${issue.nodeId}">
            Dismiss
          </button>
        </div>
      </div>
    `;
  }

  /**
   * Render recommendations widget
   */
  renderRecommendations(temporal) {
    const { recommendations } = temporal;

    if (recommendations.length === 0) {
      return '';
    }

    return `
      <div class="widget recommendations-widget">
        <div class="collapsible-header" data-collapsible="recommendations">
          <span>
            <span class="collapsible-icon">▶</span>
            Recommendations
            <span class="issue-count">${recommendations.length}</span>
          </span>
        </div>
        <div class="collapsible-content collapsed" id="recommendations-content">
          <ul class="recommendations-list">
            ${recommendations.map((rec) => `<li>${rec}</li>`).join('')}
          </ul>
        </div>
      </div>
    `;
  }

  /**
   * Render actions widget
   */
  renderActions() {
    return `
      <div class="widget actions-widget">
        <div class="actions-container">
          <button class="action-button" id="refresh-button">
            <span class="icon">🔄</span>
            Refresh Now
          </button>
          <button class="action-button" id="settings-button">
            <span class="icon">⚙️</span>
            Settings
          </button>
        </div>
        <div class="status-footer">
          Last updated: <span class="status-time">just now</span><br>
          Next check: <span class="status-time">in 5 minutes</span>
        </div>
      </div>
    `;
  }

  /**
   * Attach event listeners to interactive elements
   */
  attachEventListeners() {
    // Refresh button
    const refreshBtn = document.getElementById('refresh-button');
    if (refreshBtn) {
      refreshBtn.addEventListener('click', async () => {
        refreshBtn.classList.add('loading');
        await this.refresh();
        refreshBtn.classList.remove('loading');
      });
    }

    // Auto-fix buttons
    const autofixBtns = document.querySelectorAll('[data-action="autofix"]');
    autofixBtns.forEach((btn) => {
      btn.addEventListener('click', () => this.autoFix());
    });

    // Collapsible sections
    const collapsibleHeaders = document.querySelectorAll('[data-collapsible]');
    collapsibleHeaders.forEach((header) => {
      header.addEventListener('click', () => {
        const contentId = header.dataset.collapsible + '-content';
        const content = document.getElementById(contentId);

        header.classList.toggle('expanded');
        content.classList.toggle('collapsed');
      });
    });
  }

  /**
   * Trigger CSS animations
   */
  triggerAnimations() {
    // Stagger widget animations
    const widgets = document.querySelectorAll('.widget');
    widgets.forEach((widget, index) => {
      widget.style.animationDelay = `${index * 0.1}s`;
    });
  }

  /**
   * Render loading state
   */
  renderLoading() {
    this.container.innerHTML = `
      <div class="health-dashboard loading">
        <div class="loading-spinner"></div>
        <div class="loading-text">Loading health data...</div>
      </div>
    `;
  }

  /**
   * Render error state
   */
  renderError(error) {
    this.container.innerHTML = `
      <div class="health-dashboard error">
        <div class="error-icon">⚠️</div>
        <div class="error-message">Failed to load health data</div>
        <div class="error-detail">${error.message}</div>
        <button class="retry-button" onclick="healthDashboard.refresh()">
          Retry
        </button>
      </div>
    `;
  }

  /**
   * Show toast notification
   */
  showToast(message, type = 'info') {
    // Simple toast implementation
    const toast = document.createElement('div');
    toast.className = `toast toast-${type}`;
    toast.textContent = message;
    document.body.appendChild(toast);

    setTimeout(() => {
      toast.classList.add('show');
    }, 100);

    setTimeout(() => {
      toast.classList.remove('show');
      setTimeout(() => toast.remove(), 300);
    }, 3000);
  }

  /**
   * Cleanup
   */
  destroy() {
    if (this.refreshInterval) {
      clearInterval(this.refreshInterval);
    }
  }
}

// Export for use in renderer.js
window.HealthDashboard = HealthDashboard;
```

### Step 2: Modify Renderer.js

**File:** `src/renderer.js`

```javascript
// REMOVE: Companion terminal initialization (lines 51-155, 9402-9577)
// DELETE: All companion-related code

// ADD: Health dashboard initialization
let healthDashboard = null;

async function initHealthDashboard() {
  console.log('[Renderer] Initializing Health Dashboard...');

  try {
    healthDashboard = new HealthDashboard('health-dashboard-panel');
    await healthDashboard.init();

    console.log('[Renderer] Health Dashboard initialized');
  } catch (error) {
    console.error('[Renderer] Health Dashboard initialization failed:', error);
  }
}

// Call during app initialization
document.addEventListener('DOMContentLoaded', async () => {
  // ... existing initialization code

  // Initialize health dashboard instead of companion
  await initHealthDashboard();

  // ... rest of initialization
});
```

### Step 3: Modify HTML

**File:** `src/index.html`

```html
<!-- REPLACE companion-panel div (lines 823-879) -->
<!-- OLD:
<div id="companion-panel" class="right-panel">
  <div id="companion-terminal" class="companion-terminal"></div>
  ...
</div>
-->

<!-- NEW: -->
<div id="health-dashboard-panel" class="right-panel">
  <div class="loading">Loading health data...</div>
</div>

<!-- Add health dashboard script -->
<script src="health-dashboard.js"></script>
```

### Step 4: Add CSS

**File:** `src/styles/health-dashboard.css` (NEW)

```css
/* Health Dashboard Styles */
.health-dashboard {
  padding: 20px;
  height: 100vh;
  overflow-y: auto;
  scrollbar-width: thin;
  scrollbar-color: #374151 transparent;
}

.health-dashboard::-webkit-scrollbar {
  width: 6px;
}

.health-dashboard::-webkit-scrollbar-thumb {
  background: #374151;
  border-radius: 3px;
}

/* Widget base styles */
.widget {
  background: #1f2937;
  border: 1px solid #374151;
  border-radius: 12px;
  padding: 16px;
  margin-bottom: 16px;
  animation: slideInRight 0.4s ease;
}

.widget:nth-child(1) {
  animation-delay: 0.1s;
}
.widget:nth-child(2) {
  animation-delay: 0.2s;
}
.widget:nth-child(3) {
  animation-delay: 0.3s;
}

@keyframes slideInRight {
  from {
    opacity: 0;
    transform: translateX(20px);
  }
  to {
    opacity: 1;
    transform: translateX(0);
  }
}

.widget-header {
  font-size: 14px;
  font-weight: 700;
  color: #f9fafb;
  margin-bottom: 16px;
  text-transform: uppercase;
  letter-spacing: 0.5px;
}

/* Health Score Widget */
.health-score-circle {
  width: 120px;
  height: 120px;
  position: relative;
  margin: 0 auto 20px;
}

.score-text {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  font-size: 36px;
  font-weight: 700;
  color: #f9fafb;
}

.health-info {
  text-align: center;
  margin-bottom: 12px;
}

.health-grade {
  font-size: 16px;
  font-weight: 600;
  color: #d1d5db;
}

.health-status {
  font-size: 14px;
  font-weight: 600;
  margin-top: 4px;
}

.current-time,
.current-period {
  font-size: 13px;
  color: #9ca3af;
  text-align: center;
  line-height: 1.5;
}

/* Goal Cards */
.goal-card {
  background: #111827;
  border: 1px solid #374151;
  border-radius: 8px;
  padding: 12px;
  margin-bottom: 8px;
  transition: all 0.2s ease;
  cursor: pointer;
}

.goal-card:hover {
  background: #1f2937;
  border-color: #3b82f6;
  transform: translateX(2px);
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
}

.goal-title {
  font-size: 14px;
  font-weight: 600;
  color: #f9fafb;
  margin-bottom: 6px;
}

.goal-deadline {
  font-size: 12px;
  display: flex;
  align-items: center;
  gap: 6px;
  margin-bottom: 8px;
}

.goal-progress {
  display: flex;
  align-items: center;
  gap: 8px;
}

.progress-bar {
  flex: 1;
  height: 6px;
  background: #374151;
  border-radius: 3px;
  overflow: hidden;
}

.progress-fill {
  height: 100%;
  background: linear-gradient(90deg, #10b981, #059669);
  border-radius: 3px;
  transition: width 0.6s ease;
}

.progress-text {
  font-size: 11px;
  color: #9ca3af;
}

/* Memory Coverage */
.coverage-row {
  display: flex;
  align-items: center;
  margin-bottom: 12px;
  gap: 12px;
}

.coverage-label {
  width: 80px;
  font-size: 12px;
  color: #9ca3af;
  text-align: right;
}

.coverage-bar-container {
  flex: 1;
  height: 20px;
  background: #374151;
  border-radius: 10px;
  overflow: hidden;
  position: relative;
}

.coverage-bar-fill {
  height: 100%;
  border-radius: 10px;
  transition: width 0.6s ease;
  position: relative;
}

.coverage-bar-fill.excellent {
  background: linear-gradient(90deg, #10b981, #059669);
}

.coverage-bar-fill.good {
  background: linear-gradient(90deg, #3b82f6, #2563eb);
}

.coverage-bar-fill.fair {
  background: linear-gradient(90deg, #f59e0b, #d97706);
}

.coverage-bar-fill.low {
  background: linear-gradient(90deg, #ef4444, #dc2626);
}

.coverage-bar-fill::after {
  content: '';
  position: absolute;
  top: 0;
  left: -100%;
  width: 100%;
  height: 100%;
  background: linear-gradient(90deg, transparent, rgba(255, 255, 255, 0.3), transparent);
  animation: shimmer 2s infinite;
}

@keyframes shimmer {
  to {
    left: 100%;
  }
}

.coverage-percent {
  width: 40px;
  font-size: 12px;
  font-weight: 600;
  color: #f9fafb;
  text-align: right;
}

.overall-coverage {
  margin-top: 16px;
  padding-top: 16px;
  border-top: 1px solid #4b5563;
  text-align: center;
  font-size: 14px;
  font-weight: 600;
  color: #f9fafb;
}

/* Issue Cards */
.issue-card {
  border-left: 4px solid;
  border-radius: 6px;
  padding: 12px;
  margin-bottom: 8px;
}

.issue-card.error {
  background: #fee2e2;
  border-color: #ef4444;
}

.issue-card.warning {
  background: #fef3c7;
  border-color: #f59e0b;
}

.issue-card.info {
  background: #dbeafe;
  border-color: #3b82f6;
}

.issue-type {
  font-size: 11px;
  font-weight: 700;
  text-transform: uppercase;
  margin-bottom: 6px;
}

.issue-card.error .issue-type,
.issue-card.error .issue-message {
  color: #7f1d1d;
}

.issue-card.warning .issue-type,
.issue-card.warning .issue-message {
  color: #92400e;
}

.issue-card.info .issue-type,
.issue-card.info .issue-message {
  color: #1e3a8a;
}

.issue-message {
  font-size: 13px;
  line-height: 1.4;
  margin-bottom: 10px;
}

.issue-actions {
  display: flex;
  gap: 8px;
}

.issue-button {
  padding: 4px 12px;
  font-size: 11px;
  font-weight: 600;
  border-radius: 4px;
  border: 1px solid;
  background: white;
  cursor: pointer;
  transition: all 0.2s ease;
}

.issue-card.error .issue-button {
  border-color: #ef4444;
  color: #ef4444;
}

.issue-card.error .issue-button:hover,
.issue-card.error .issue-button.primary {
  background: #ef4444;
  color: white;
}

/* Collapsible sections */
.collapsible-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 12px;
  cursor: pointer;
  user-select: none;
  transition: background 0.2s ease;
}

.collapsible-header:hover {
  background: #374151;
}

.collapsible-icon {
  display: inline-block;
  transition: transform 0.2s ease;
  margin-right: 8px;
}

.collapsible-header.expanded .collapsible-icon {
  transform: rotate(90deg);
}

.collapsible-content.collapsed {
  display: none;
}

.issue-count {
  display: inline-block;
  background: #374151;
  color: #f9fafb;
  padding: 2px 8px;
  border-radius: 12px;
  font-size: 11px;
  font-weight: 700;
  margin-left: 8px;
}

/* Action Buttons */
.actions-container {
  display: flex;
  gap: 8px;
  margin-bottom: 12px;
}

.action-button {
  flex: 1;
  padding: 10px 16px;
  font-size: 13px;
  font-weight: 600;
  border-radius: 6px;
  border: 1px solid #374151;
  background: #1f2937;
  color: #f9fafb;
  cursor: pointer;
  transition: all 0.2s ease;
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 6px;
}

.action-button:hover {
  background: #3b82f6;
  color: white;
  border-color: #3b82f6;
  transform: translateY(-1px);
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
}

.action-button.loading {
  pointer-events: none;
  opacity: 0.6;
}

.action-button.loading .icon {
  animation: spin 1s linear infinite;
}

@keyframes spin {
  to {
    transform: rotate(360deg);
  }
}

.status-footer {
  font-size: 11px;
  color: #6b7280;
  text-align: center;
  line-height: 1.6;
}

.status-time {
  font-weight: 600;
  color: #9ca3af;
}

/* Loading & Error states */
.health-dashboard.loading {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  height: 100%;
}

.loading-spinner {
  width: 48px;
  height: 48px;
  border: 4px solid #374151;
  border-top-color: #3b82f6;
  border-radius: 50%;
  animation: spin 1s linear infinite;
}

.loading-text {
  margin-top: 16px;
  font-size: 14px;
  color: #9ca3af;
}

.health-dashboard.error {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  height: 100%;
}

.error-icon {
  font-size: 48px;
  margin-bottom: 16px;
}

.error-message {
  font-size: 16px;
  font-weight: 600;
  color: #f9fafb;
  margin-bottom: 8px;
}

.error-detail {
  font-size: 13px;
  color: #9ca3af;
  margin-bottom: 16px;
}

.retry-button {
  padding: 8px 16px;
  font-size: 13px;
  font-weight: 600;
  border-radius: 6px;
  border: 1px solid #3b82f6;
  background: #3b82f6;
  color: white;
  cursor: pointer;
  transition: all 0.2s ease;
}

.retry-button:hover {
  background: #2563eb;
  border-color: #2563eb;
}

/* Toast notifications */
.toast {
  position: fixed;
  bottom: 20px;
  right: 20px;
  padding: 12px 20px;
  border-radius: 8px;
  font-size: 13px;
  font-weight: 600;
  color: white;
  opacity: 0;
  transform: translateY(10px);
  transition: all 0.3s ease;
  z-index: 10000;
}

.toast.show {
  opacity: 1;
  transform: translateY(0);
}

.toast-success {
  background: #10b981;
}

.toast-error {
  background: #ef4444;
}

.toast-info {
  background: #3b82f6;
}
```

**Link CSS in index.html:**

```html
<link rel="stylesheet" href="styles/health-dashboard.css" />
```

---

## 🚀 PART 5: IMPLEMENTATION ROADMAP

### Phase 1: Backend - MCP Tools (1.5 days)

**Files to create:**

1. `servers/odei-neo4j/src/services/temporalContext.ts` (~400 lines)
2. `servers/odei-neo4j/src/services/metaMemory.ts` (~350 lines)
3. `servers/odei-neo4j/src/tools/temporalContext.ts` (~200 lines)
4. `servers/odei-neo4j/src/tools/memoryCoverage.ts` (~150 lines)
5. `servers/odei-neo4j/src/tools/temporalAutofix.ts` (~180 lines)

**Files to modify:**

1. `servers/odei-neo4j/src/tools/index.ts` (register new tools)

**Steps:**

1. Create service layer with Neo4j queries
2. Create MCP tool handlers
3. Test queries in Neo4j Browser
4. Test MCP tools via curl/Postman
5. Build and verify: `cd servers/odei-neo4j && npm run build`

### Phase 2: Frontend - UI Components (1 day)

**Files to create:**

1. `src/health-dashboard.js` (~600 lines)
2. `src/styles/health-dashboard.css` (~450 lines)

**Files to modify:**

1. `src/renderer.js` (remove companion code ~500 lines, add dashboard init ~20 lines)
2. `src/index.html` (replace companion panel div)

**Steps:**

1. Create HealthDashboard class
2. Implement all render methods
3. Add CSS styles
4. Test in isolation (mock data first)
5. Connect to real MCP calls

### Phase 3: Integration & Testing (0.5 days)

**Tasks:**

1. Rebuild odei-neo4j: `npm run build`
2. Restart Electron app
3. Test full data flow: Neo4j → MCP → IPC → Renderer
4. Test auto-refresh (wait 5 minutes)
5. Test auto-fix functionality
6. Test error handling (kill Neo4j temporarily)
7. Test with empty graph (no goals)
8. Test with stale data

### Phase 4: Polish & Optimization (0.5 days)

**Tasks:**

1. Add loading skeletons
2. Optimize Neo4j queries (add indexes if needed)
3. Add error boundary
4. Add keyboard shortcuts (R to refresh)
5. Add settings modal (configure refresh interval)
6. Add export functionality (JSON/CSV)

**Total: 3.5 days**

---

## 📝 TESTING CHECKLIST

### Unit Tests

- [ ] Temporal health score calculation
- [ ] Stale goal detection
- [ ] Coverage percentage calculation
- [ ] Auto-fix dry run vs. actual
- [ ] Goal urgency classification

### Integration Tests

- [ ] MCP tool calls return valid JSON
- [ ] IPC channel handles errors gracefully
- [ ] Renderer parses MCP responses correctly
- [ ] Auto-refresh triggers on interval
- [ ] Manual refresh button works

### End-to-End Tests

- [ ] Dashboard loads on app start
- [ ] Health score displays correctly
- [ ] Active goals appear with correct deadlines
- [ ] Coverage bars show accurate percentages
- [ ] Issues are detected and displayed
- [ ] Auto-fix resolves stale goals
- [ ] Dashboard survives Neo4j restart
- [ ] Dashboard handles no data gracefully

### Edge Cases

- [ ] No active goals (empty state)
- [ ] No issues (success state)
- [ ] 100% health score
- [ ] 0% health score (critical state)
- [ ] Very long goal titles (truncation)
- [ ] Goals with no deadline
- [ ] Coverage with missing layers

---

## 🎁 BONUS: SUBAGENT INTEGRATION (Optional)

After core implementation, optionally add Task-based subagents for agents:

### Subagent: `temporal-health-check`

**File:** `.claude/agents/temporal-health-check.md`

````markdown
---
name: temporal-health-check
description: Check temporal health status and detect stale data in the knowledge graph
when_to_use: >
  Use when you need to validate that goals, tasks, and deadlines are current
  and accurate. Essential before making strategic recommendations.
---

# Temporal Health Check Subagent

## Quick Check

```bash
# Task invocation from Claude Code
Task({
  label: 'temporal health sweep',
  files: ['.claude/agents/temporal-health-check.md'],
  instructions: 'Need latest grade + issue list before planning sprint.'
})
```
````

## Interpreting Results

- **Grade A (90-100)**: Excellent - minimal temporal issues
- **Grade B (80-89)**: Good - minor issues, review recommended
- **Grade C (70-79)**: Fair - several issues need attention
- **Grade D (60-69)**: Poor - significant temporal drift
- **Grade F (<60)**: Critical - immediate action required

## Auto-Fix

```bash
# Preview fixes
window.odei.data.get('odei-neo4j', 'odei_neo4j_temporal_autofix_v1', {
  issueType: 'ALL',
  dryRun: true
})

# Apply fixes
window.odei.data.get('odei-neo4j', 'odei_neo4j_temporal_autofix_v1', {
  issueType: 'ALL',
  dryRun: false
})
```

```

This gives agents built-in temporal awareness without cluttering the main UI!

---

## 🎯 SUCCESS CRITERIA

Dashboard successfully implemented when:

1. ✅ **Real Data**: All widgets show live data from Neo4j (not mock data)
2. ✅ **Auto-Refresh**: Dashboard updates every 5 minutes automatically
3. ✅ **Health Score**: Accurate 0-100 score with A-F grade
4. ✅ **Active Goals**: Shows current quarter/month/week goals with real deadlines
5. ✅ **Coverage Bars**: 7 layers with accurate percentages
6. ✅ **Issue Detection**: Finds stale goals, overdue tasks, future completions
7. ✅ **Auto-Fix**: One-click resolution of fixable issues
8. ✅ **Animations**: Smooth transitions, shimmer effects, slide-in
9. ✅ **Error Handling**: Graceful fallback when MCP servers unavailable
10. ✅ **Performance**: Loads in <2 seconds, refresh in <1 second

---

## 🔥 NEXT STEPS

**Start with Phase 1 - Backend MCP Tools:**

1. Create `servers/odei-neo4j/src/services/temporalContext.ts`
2. Test queries in Neo4j Browser first
3. Build MCP tool wrappers
4. Test via IPC manually

**Then Phase 2 - Frontend:**

1. Create `src/health-dashboard.js`
2. Test with mock data first
3. Connect to real MCP calls
4. Add CSS and animations

**Finally Phase 3 - Integration:**

1. Remove companion code
2. Wire up dashboard
3. Test end-to-end
4. Polish and deploy

---

**Партнер, этот план дает нам полную реализацию Health Dashboard с реальными данными из Neo4j!** 🚀

Все слои спроектированы:
- ✅ Data Layer (Neo4j queries)
- ✅ Service Layer (MCP tools)
- ✅ Transport Layer (existing IPC)
- ✅ UI Layer (React-like components)

Готов начать реализацию! С чего начнем?
```
