---
name: explore
description: Structured, detailed exploration of any codebase component — screens, APIs, modules, data flows, or systems. Outputs visual layout, user actions, full request/response cycles, concurrent operations, and persistence patterns. Use when the user says "explore", "walk me through", "explain how X works end to end", "show me the flow of", or wants a detailed structured walkthrough of any part of the codebase.
allowed-tools: ["Grep", "Glob", "Read", "Agent"]
---

# Structured Codebase Explorer

Perform a detailed, structured exploration of any codebase component and produce a comprehensive walkthrough. Works with any language or framework. Follows a systematic process to trace every user action through the full stack.

## Usage

When the user invokes `/explore`:

**Mode selection:**
- `/explore <target>` — **full** mode (default). Complete structured walkthrough with all sections.
- `/explore quick <target>` — **quick** mode. Summary only — key files, main flow, no ASCII diagrams.
- `/explore screen <screen_name>` — **screen** mode. Frontend screen walkthrough with layout, actions, data flows.
- `/explore api <endpoint>` — **api** mode. Backend API deep dive — handler, middleware, DB queries, response shape.
- `/explore flow <process>` — **flow** mode. End-to-end data flow tracing — from trigger to persistence.

The target can be:
- A screen/page: `/explore screen login`, `/explore agent list page`
- An API endpoint: `/explore api POST /api/agents`, `/explore GET /api/usage`
- A process/flow: `/explore flow message sending`, `/explore token usage tracking`
- A module/feature: `/explore chart system`, `/explore file explorer`
- A general area: `/explore frontend architecture`, `/explore auth system`

## Process

Follow these steps strictly. Each step uses real code — never guess or fabricate.

### Step 1: Identify scope and locate entry points

- If the target is a **screen/page**: Use Grep to find the route definition, then locate the page component
- If the target is an **API endpoint**: Use Grep to find the route handler in backend routing files
- If the target is a **process/flow**: Use Grep to find the trigger point (user action, WS event, scheduled job)
- If the target is a **module/feature**: Use Glob to find all files in the module directory, Grep for key exports

Record every file path found — these will be read in subsequent steps.

### Step 2: Read the entry point(s)

- Read the main handler/component source code using the Read tool
- Note the exact file path and line numbers
- Identify all child components, imported services, and called functions

### Step 3: Trace the full call chain

For each action/interaction found in Step 2:

1. **Trace forward** — for every function call, use Grep to find its definition, then Read it
2. **Trace the data** — follow the data from input to output: what transformations happen, what gets persisted
3. **Trace the response** — follow the return path back to the user
4. **Maximum depth: 4 levels** — beyond that, summarize the sub-function in one line
5. **Use parallel Agent calls** when tracing multiple independent paths (e.g., frontend + backend simultaneously)

### Step 4: Identify concurrent operations

- Look for simultaneous event streams (WebSocket events, parallel API calls, background jobs)
- Note what updates happen independently of the main flow
- Identify any polling, intervals, or real-time subscriptions

### Step 5: Map persistence

- What gets written to disk/DB? Where? In what format?
- What gets read? From where?
- Are there caches? How are they invalidated?

### Step 6: Produce structured output

Produce the exploration result using this template:

---

#### Section 1: Visual Layout (for screens/frontend only)

```
┌──────────────────────────────────────┐
│  Header / Title bar                  │
├──────────┬───────────────────────────┤
│          │                           │
│  Sidebar │  Main content area        │
│          │                           │
│          │  ┌───────────────────┐    │
│          │  │  Component        │    │
│          │  └───────────────────┘    │
│          │                           │
├──────────┴───────────────────────────┤
│  Footer / Input area                 │
└──────────────────────────────────────┘
```

Describe what the user sees — layout, buttons, cards, input fields, status indicators. Be specific about text labels, colors, and arrangement.

#### Section 2: User Actions and Data Flows

For every user action, produce a numbered trace:

```
Frontend                    Backend                     DB/Disk
────────                    ──────                     ────────

1. User clicks X
2. Component handler
   calls API
   POST /api/thing       ──► 3. Route handler
                              4. Validation
                              5. Service call
                              6. DB write             ──► INSERT INTO table
   ◄── { response }      ◄── 7. Return result

8. UI updates with response
```

Each trace must include:
- **What triggers it** (user click, timer, WS event)
- **What API/WS call is made** (exact endpoint, method, payload shape)
- **What the backend does** (validation, business logic, persistence)
- **What gets written** (DB tables, files, cache)
- **What response comes back** (exact response shape)
- **How the UI updates** (state change, re-render)

#### Section 3: Concurrent Operations

List what happens simultaneously during normal operation:

```
Timeline:
─────────────────────────────────────────────────────
  T+0s   Action A triggered
  T+0.1s Action B starts (parallel)
  T+1s   Event stream X updates UI
  T+2s   Background job Y completes
─────────────────────────────────────────────────────
```

#### Section 4: Persistence Map

| What | Where | Format | When |
|------|-------|--------|------|
| Data A | path/to/file.db | SQLite table | On every write |
| Data B | path/to/file.json | JSON | On config change |

#### Section 5: Error Handling

| Scenario | What happens | User sees |
|----------|-------------|-----------|
| Network failure | Retry / reconnect | Error toast |
| Validation fail | 400 response | Inline error |

#### Section 6: File Map

```
path/to/file_a.ts — Main page component (lines X–Y)
path/to/file_b.ts — API route handler (lines X–Y)
path/to/file_c.ts — Service/business logic (lines X–Y)
path/to/file_d.ts — DB/storage layer (lines X–Y)
path/to/file_e.ts — Types and interfaces (lines X–Y)
```

---

## Quick Mode

When invoked with `quick`:
- Skip ASCII diagrams and detailed traces
- Output only: key files, main flow summary (5-10 lines), data persistence summary, and 2-3 key observations
- Maximum 100 lines of output

## Critical Rules

1. **NEVER guess code.** Every file path, line number, function signature, and response shape must come from actually reading the code with the Read/Grep tools.
2. **NEVER fabricate data flows.** If you can't trace a path fully, say "untraceable from here" rather than making up the next step.
3. **Read before describing.** Read the actual component/handler before describing what it does or what the UI looks like.
4. **Use parallel exploration.** When tracing both frontend and backend for the same feature, launch Agent calls in parallel for speed.
5. **Be exhaustive on actions.** Every button, input, dropdown, and interaction on a screen must be covered — don't skip "minor" actions.
6. **Show real payload shapes.** Use actual JSON shapes from the code (request bodies, response objects, event payloads), not approximations.
7. **Include line numbers.** Every file reference should include line numbers from actual reads.
8. **Respect scope.** If the user asked about one screen, don't wander into related screens unless the flow naturally crosses boundaries.
9. **No filler.** Don't add preamble like "Here's the exploration..." or "Based on my analysis...". Start directly with the structured output.
10. **Adapt to language.** The skill works with any language/framework — detect from the code, don't assume TypeScript/React.
