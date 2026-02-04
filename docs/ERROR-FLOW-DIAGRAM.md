# ODEI Error Handling Flow Diagram

Visual guide to understanding how errors flow through the system.

## Error Flow Overview

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         ERROR OCCURS                                     │
│                              ↓                                           │
│  ┌───────────────┬──────────────────┬──────────────────┬──────────────┐ │
│  │  Renderer     │  IPC Channel     │  Main Process    │  MCP Server  │ │
│  │  (UI code)    │                  │  (Node.js)       │  (External)  │ │
└──┴───────────────┴──────────────────┴──────────────────┴──────────────┴─┘
```

## 1. Renderer Process Error Flow

```
User Action / Code Execution
         ↓
    Error Occurs
         ↓
  ┌─────────────────────┐
  │  Try/Catch Block?   │
  └─────────────────────┘
         ↓
    ┌────┴────┐
    │   YES   │                           NO
    └────┬────┘                           ↓
         ↓                    ┌──────────────────────────┐
  ┌──────────────────┐        │  Global Error Handlers   │
  │  Error Caught    │        │  (error-init.js)         │
  └──────────────────┘        └──────────────────────────┘
         ↓                                ↓
  ┌──────────────────┐        ┌──────────────────────────┐
  │  errorHandler    │←───────│  Uncaught Error /        │
  │  .logError()     │        │  Unhandled Rejection     │
  └──────────────────┘        └──────────────────────────┘
         ↓
  ┌──────────────────────────────────────┐
  │  Error Processing                     │
  │  • Add timestamp                      │
  │  • Store in error log                 │
  │  • Emit to subscribers                │
  │  • Send critical to main process      │
  └──────────────────────────────────────┘
         ↓
  ┌─────────────────────────────┐
  │  User Notification          │
  │  ┌───────────────────────┐  │
  │  │  Error Toast          │  │  (Quick, auto-dismiss)
  │  │  or                   │  │
  │  │  Error Panel          │  │  (Full, with retry)
  │  └───────────────────────┘  │
  └─────────────────────────────┘
         ↓
  ┌─────────────────────────────┐
  │  Recovery Options           │
  │  • Retry operation          │
  │  • Dismiss notification     │
  │  • Reload application       │
  │  • Export error log         │
  └─────────────────────────────┘
```

## 2. IPC Communication Error Flow

```
Renderer                           Main Process
   │                                    │
   │  odei.data.get(server, tool, params)
   │────────────────────────────────────▶│
   │                                    │
   │                                    ├─ wrapIPCHandler()
   │                                    │    ↓
   │                                    ├─ validateParams()
   │                                    │    ↓
   │                                    │  Try/Catch
   │                                    │    ↓
   │                                    ├─ ERROR?
   │                                    │    ↓
   │                                    │  ┌───YES────┐      NO
   │                                    │  ↓          │       ↓
   │                                    │ Map to     │   SUCCESS
   │                                    │ Error Code │       ↓
   │                                    │  ↓          │  createSuccess
   │                                    │ createError│  Response()
   │                                    │ Response() │       ↓
   │                                    │  ↓          │       ↓
   │◀───────────────────────────────────┴─────────────┴───────┤
   │  { success: false, error: {...} }
   │  or
   │  { success: true, data: {...} }
   │
   ├─ Check response.success
   │    ↓
   │  ┌────NO─────┐       YES
   │  ↓           │        ↓
   │ Handle Error │   Process Data
   │  ↓           │        ↓
   │ showError   │   Update UI
   │ Toast       │
   │             │
   └─────────────┘
```

## 3. MCP Server Error Flow

```
Main Process                     MCP Server
   │                                  │
   │  healthProbes.callTool()         │
   │──────────────────────────────────▶│
   │                                  │
   │                                  ├─ Tool Handler
   │                                  │    ↓
   │                                  ├─ validateParams()
   │                                  │    ↓
   │                                  │  Try/Catch
   │                                  │    ↓
   │                                  ├─ Database/Service Call
   │                                  │    ↓
   │                                  │  ERROR?
   │                                  │    ↓
   │                                  │  ┌───YES────┐      NO
   │                                  │  ↓          │       ↓
   │                                  │ Create     │   Format
   │                                  │ MCPError   │   Result
   │                                  │  ↓          │       ↓
   │                                  │ Return     │   Return
   │                                  │ Error      │   Success
   │                                  │ Response   │   Response
   │                                  │  ↓          │       ↓
   │◀─────────────────────────────────┴─────────────┴───────┤
   │  { content: [{ type: "text", text: "{error: ...}" }],
   │    isError: true }
   │  or
   │  { content: [{ type: "text", text: "{...data}" }],
   │    isError: false }
   │
   ├─ Check result.isError
   │    ↓
   │  ┌────YES────┐       NO
   │  ↓           │        ↓
   │ Parse Error │   Parse Data
   │  ↓           │        ↓
   │ Return to   │   Return to
   │ Renderer    │   Renderer
   │             │
   └─────────────┘
```

## 4. Component-Level Error Flow

```
Component Initialization
         ↓
  ┌──────────────────┐
  │  ErrorBoundary   │
  │  .render()       │
  └──────────────────┘
         ↓
    Try/Catch
         ↓
  ┌──────────────────┐
  │  Component Code  │
  │  • Load data     │
  │  • Render UI     │
  │  • Setup events  │
  └──────────────────┘
         ↓
      ERROR?
         ↓
   ┌─────┴─────┐
   │    YES    │                    NO
   └─────┬─────┘                    ↓
         ↓                     SUCCESS
  ┌──────────────────┐              ↓
  │  Error Caught    │        Component Rendered
  └──────────────────┘
         ↓
  ┌──────────────────────────┐
  │  boundary.handleError()  │
  │  • Log error             │
  │  • Call onError callback │
  │  • Show error UI         │
  └──────────────────────────┘
         ↓
  ┌──────────────────────────┐
  │  Error Panel in          │
  │  Component Container     │
  │  ┌────────────────────┐  │
  │  │  [X] Module Error  │  │
  │  │  Message here      │  │
  │  │  [Retry] [Reload]  │  │
  │  └────────────────────┘  │
  └──────────────────────────┘
         ↓
    User Action
         ↓
   ┌─────┴─────┐
   │   RETRY   │               RELOAD
   └─────┬─────┘                 ↓
         ↓                   location.reload()
  boundary.reset()
         ↓
  Re-run Component Init
```

## 5. Error Severity Flow

```
Error Occurs
     ↓
  Severity?
     ↓
  ┌──────┴──────────┬──────────┬──────────┐
  │                 │          │          │
 INFO            WARNING     ERROR    CRITICAL
  │                 │          │          │
  ↓                 ↓          ↓          ↓
Toast           Toast       Toast      Panel
Info color    Warning     Error      + Log
              color       color      + Main
  ↓                 ↓          ↓      Process
Auto-          Auto-      Auto-         ↓
dismiss        dismiss    dismiss   Requires
3 sec          5 sec      5 sec     Action
  ↓                 ↓          ↓          ↓
 Done             Done       Done    User
                                    Resolves
```

## 6. Error Recovery Flow

```
Error Displayed to User
         ↓
  ┌──────────────────┐
  │  Recovery Options│
  └──────────────────┘
         ↓
  ┌──────┴────────────────────┬─────────────────┐
  │                           │                 │
RETRY                      DISMISS           RELOAD
  │                           │                 │
  ↓                           ↓                 ↓
Try Same                   Hide Error      location.reload()
Operation                  Continue              ↓
  ↓                        Working          Fresh Start
SUCCESS?                      ↓
  ↓                          Done
  ├─YES─▶ Show Success Toast
  │          ↓
  │         Done
  │
  ├─NO──▶ Show Error Again
            ↓
         Suggest Reload
```

## 7. Error Context Flow

```
Error Occurs
     ↓
  ┌─────────────────────────────┐
  │  Context Collection         │
  │  • Module name              │
  │  • Operation details        │
  │  • User action              │
  │  • Input parameters         │
  │  • Current state            │
  │  • Timestamp                │
  └─────────────────────────────┘
     ↓
  ┌─────────────────────────────┐
  │  errorHandler.logError()    │
  │  Combines:                  │
  │  • Error object             │
  │  • Context data             │
  │  • Stack trace              │
  │  • User agent               │
  │  • URL                      │
  └─────────────────────────────┘
     ↓
  ┌─────────────────────────────┐
  │  Storage                    │
  │  • In-memory log (100 max)  │
  │  • localStorage (crashes)   │
  │  • Console output           │
  │  • Main process (critical)  │
  └─────────────────────────────┘
     ↓
  ┌─────────────────────────────┐
  │  Retrieval                  │
  │  • getErrorLog()            │
  │  • exportErrorLog()         │
  │  • Crash recovery           │
  └─────────────────────────────┘
```

## 8. Error Propagation

```
Lowest Level                                      Highest Level
    │                                                  │
    ▼                                                  ▼
┌─────────┐   ┌─────────┐   ┌─────────┐   ┌─────────────────┐
│Database │──▶│ Service │──▶│  Module │──▶│  Application    │
│         │   │         │   │         │   │  Global Handler │
└─────────┘   └─────────┘   └─────────┘   └─────────────────┘
    │             │             │                  │
    ↓             ↓             ↓                  ↓
Neo4j Error   MCPError    ODEIError         User Notification
+ context     + user msg  + code            + recovery options
    │             │             │                  │
    └─────────────┴─────────────┴──────────────────┘
                        │
                        ▼
              ┌─────────────────┐
              │  Error Log      │
              │  with full      │
              │  context chain  │
              └─────────────────┘
```

## 9. Critical vs Optional Errors

```
Error Occurs
     ↓
  Module Type?
     ↓
  ┌──────┴──────┐
  │             │
CRITICAL     OPTIONAL
  │             │
  ↓             ↓
Must        Can Continue
Succeed     Without
  │             │
  ↓             ↓
Error       Warning Toast
Panel       "Feature unavailable"
  ↓             ↓
Block       Graceful
Further     Degradation
Init            ↓
  ↓         App Continues
User Must    Working
Resolve
  ↓
Retry or
Reload

Example:
CRITICAL          OPTIONAL
- Database        - Finance Module
- UI Manager      - 3D Graph
- Terminal Mgr    - Apple Health
```

## 10. Full System Error Flow

```
                    ┌─────────────────────────┐
                    │  User Action            │
                    └───────────┬─────────────┘
                                ↓
                    ┌─────────────────────────┐
                    │  UI Component           │
                    └───────────┬─────────────┘
                                ↓
                          Try/Catch
                                ↓
                           ┌────┴────┐
                           │  Error? │
                           └────┬────┘
                                ↓
                          ┌─────┴──────┐
                          │    YES     │           NO
                          └─────┬──────┘            ↓
                                ↓               SUCCESS
                    ┌─────────────────────────┐     ↓
                    │  ErrorBoundary/Handler  │  Continue
                    └───────────┬─────────────┘
                                ↓
                    ┌─────────────────────────┐
                    │  errorHandler.logError()│
                    │  • Store in log         │
                    │  • Emit event           │
                    │  • Format message       │
                    └───────────┬─────────────┘
                                ↓
                          ┌─────┴─────┐
                          │ Severity? │
                          └─────┬─────┘
                                ↓
                    ┌───────────┴───────────┐
                    │                       │
              INFO/WARNING             ERROR/CRITICAL
                    │                       │
                    ↓                       ↓
            ┌───────────────┐      ┌──────────────────┐
            │  Error Toast  │      │  Error Panel     │
            │  Auto-dismiss │      │  Requires Action │
            └───────────────┘      └────────┬─────────┘
                    │                       │
                    ↓                       ↓
                   Done            ┌────────────────┐
                                   │ User Action    │
                                   │ • Retry        │
                                   │ • Dismiss      │
                                   │ • Reload       │
                                   └────────┬───────┘
                                            ↓
                                          Done
```

---

**Visual Guide Complete**
All error flows documented with clear paths and decision points.
