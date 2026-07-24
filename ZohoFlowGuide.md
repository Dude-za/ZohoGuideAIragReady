# Zoho Flow Guide — Event Hook Custom Workflows

**Scope:** Zoho Flow — event hook custom workflows for events like logon and logoff.

> This file is environment-specific. Load `ZohoGuideRoot.md` first for pervasive rules (canonical template, API patterns, ownership model, pre-deployment checklist).

**Last Updated:** 2026-07-24

---

## §F.1 — Parser Caveats (Forbidden/Unreliable Constructs)

Zoho Flow's Deluge parser is stricter than other environments. **Avoid these constructs:**

| Construct | Status | Alternative |
|-----------|--------|-------------|
| `instanceof` | ❌ Not supported | — |
| `continue` | ❌ Not supported | Replace with nested `if` guards |
| `else` | ⚠️ Often misparsed | Separate `if` blocks or early returns |
| Custom helper function declarations (e.g., `void logDebug(...)`, `myFunc = function(...)`) | ❌ Not allowed inside Flow custom functions | Inline all logic at every call site |
| `ifelse()` | ❌ NOT available in Flow | Use explicit `if/else` blocks instead |
| `info` statements | ✅ Work in Flow | Viewable in execution log output |

**Chained boolean expressions** (e.g., `if(a && b)`) — expand to separate guarded booleans if parsing fails.

**Empty `catch` blocks** — Always add a harmless assignment (e.g., `dummy = "caught";`).

**"Loose expressions"** (assignments that Flow thinks sit outside a block) — ensure braces are balanced and avoid risky constructs.

**`sortBy` function** — **NOT available in Deluge**. To sort a list of maps by a field, use extract-sort-rebuild:
```deluge
// Sort list of maps by "priority" ascending
items = List();
items.add({"name": "a", "priority": 3});
items.add({"name": "b", "priority": 1});
items.add({"name": "c", "priority": 2});

// Step 1: extract field values into separate list
vals = List();
for each item in items
{
    vals.add(item.get("priority"));
}

// Step 2: sort values (.sort(true)=ascending, .sort()=descending)
vals.sort(true);

// Step 3: rebuild sorted list (tracks used indices for duplicates)
sorted = List();
used = List();
for each v in vals
{
    i = 0;
    for each item in items
    {
        if (item.get("priority") == v && !used.contains(i))
        {
            sorted.add(item);
            used.add(i);
            break;
        }
        i = i + 1;
    }
}
// sorted now holds items ordered by priority ascending
```
- Alternative: Use a manual loop to find min/max value (simpler for single value)
- See ZohoGuideRoot §3.2 for the `.toLong()` variant when sorting by numeric keys.

### §F.1.1 — `info` Statements vs CSV Debug Logs — Speed Advantage

Flow execution logs are available via two methods:

| Method | Speed | Use Case |
|--------|-------|----------|
| **`info` statements** | **Instant** — visible in execution history immediately after run | Single-run debugging, error detection, status checks |
| **Download CSV** (execution data) | **Slow** — requires export, download, open in spreadsheet | Bulk analysis, multi-run trend analysis |

**Rule:** For **development and debugging**, always use `info` statements to report key status values and API errors. CSV download is for **post-hoc analysis** of production trends, not for real-time debugging.

**Why this matters:** Waiting for CSV exports (especially on mobile or slow connections) adds minutes of latency per iteration. `info` output appears in the execution log tile instantly — click the run, read the result.

**Best practice pattern:**
```deluge
// Entry point — always log inputs
info funcVersion + " ENTERED | agent=" + agentEmail + " status=" + status;

// Critical API results — log success/failure + error detail
info funcVersion + " STEP4 | INSERT success=" + addSuccess + " agent=" + agentId;

// Errors — log error code and message for rapid diagnosis
info funcVersion + " API_ERROR | step=STEP4 code=" + errorCode + " msg=" + errorMsg;

// Summary — one-liner with all key outcomes
info funcVersion + " END | online=" + count + " assigned=" + assigned + " insertOK=" + success;
```

### §F.1.2 — Debug Log Table Write-Then-Info Pattern

**Pattern:** When debugging Flow functions that write to Analytics, write to a debug log table FIRST, then use `info` to report whether the write succeeded. This gives two layers of visibility:
1. **`info`** — instant feedback in execution history (success/failure of the debug write itself)
2. **Debug log table** — persistent audit trail for multi-run analysis

**Why not just `info`?** `info` is ephemeral — it disappears if you need to compare across runs. The debug log table persists and can be queried with ZQL.

**Why write to the table before `info`?** So the `info` statement can report whether the debug write landed. If the debug write fails (e.g., permission error), you see it instantly in `info` without downloading a CSV.

**Implementation (inlined — no helper functions allowed in Flow):**
```deluge
// Write to debug log table FIRST
dbgWriteOk = false;
try
{
    dbgCfg = Map();
    dbgCols = Map();
    dbgCols.put("Run ID", runId);
    dbgCols.put("Timestamp", zoho.currenttime.toString("yyyy-MM-dd'T'HH:mm:ss'Z'"));
    dbgCols.put("Function", "MyFunction");
    dbgCols.put("Version", funcVersion);
    dbgCols.put("Step", "STEP_NAME");
    dbgCols.put("Level", "INFO");
    dbgCols.put("Message", "description of what happened");
    dbgCfg.put("columns", dbgCols);
    dbgCfg.put("addIfNotExist", true);
    dbgCfg.put("criteria", "1=2");
    dbgCfgEnc = zoho.encryption.urlEncode(dbgCfg.toString());
    dbgPutUrl = analytics_domain + "/restapi/v2/workspaces/" + wsId + "/views/" + debugLogViewId + "/rows?CONFIG=" + dbgCfgEnc;
    dbgPutResp = invokeurl
    [
        url: dbgPutUrl
        type: PUT
        headers: {"ZANALYTICS-ORGID": orgId, "Content-Type": "application/json"}
        connection: "your_connection"
    ];
    dbgPutRespStr = dbgPutResp.toString();
    if (dbgPutRespStr.contains("errorCode")) { dbgWriteOk = false; }
    else { dbgWriteOk = true; }
}
catch (e_dbg) { dummy = "caught"; dbgWriteOk = false; }

// THEN report via info — includes dbgLogOK status
info funcVersion + " STEP_NAME | details... dbgLogOK=" + dbgWriteOk.toString();
```

**Key constraints:**
- All logic must be inlined (no `function` declarations in Flow — §F.1)
- Use `zoho.encryption.urlEncode()` for CONFIG (not `parameters: Map()` — unreliable for PUT)
- Each debug write site has ~20 lines of boilerplate — place at critical decision points only (entry, key API calls, summary)

---

## §F.2 — Expression/Mapping, Variable Initialization, Function Signatures

### §F.2.1 — Expression/Mapping Caveats (Flow UI)

- Dragged variables from the picker are rendered lower-case inside expressions (e.g., `Status` → `${trigger.status}`).
- In **Test** overlays, early evaluation can yield empty strings for casts like `toString(...)`; live runs resolve correctly.
- If any mapped input (trigger field or Flow variable) resolves to a **KEY-VALUE** object, Flow may inject an internal validator like `<KEYVALUE> == "TEXT"` and fail with _"left expression is KEY-VALUE, right expression is TEXT"_. Avoid by ensuring text coercion and/or string-only storage.

### §F.2.2 — Variable Initialization & State (critical)

- Newly created Flow variables are **untyped** until first set; empty Map/Number vars behave like KEY-VALUE wrappers in validation.
- Test/Debug does **not** persist variable values; only live runs persist state.
- **Remedies:**
  1. Run a one-time initializer path (two **Set Variable** steps) before the function; or
  2. **Recommended:** store caches as **strings** in Flow and parse inside Deluge (§F.3).

### §F.2.3 — Function Signature & First-Line Rules (Flow)

- First non-whitespace line **must** be the signature, and `{` must be on the same line:
  ```deluge
  map MyFunc(string a, string b)
  {
      // body
  }
  ```
- Explicit primitive types are required for all parameters (e.g., `string`, `int`).
- **ONLY include Flow trigger inputs** in the signature. Static constants (orgId, domain, workspace IDs) should be defined as constants inside the function body, not passed from Flow. This makes the function portable and cloneable.

---

## §F.3 — Cache Patterns & Persistence

### §F.3.1 — String-Based Cache Pattern (Flow-Safe State Across Runs)

**Why**: Flow reliably passes strings; Deluge can parse them safely. This bypasses KEY-VALUE vs TEXT errors and stabilizes persistence.

**Function signature (example):**
```deluge
map UpdateAgentPresence(string flowAgentId, string flowPresenceStatus,
                       string presenceCacheStr, string onlineCountStr)
{
    // Parse presence cache string → Map
    presenceCache = Map();
    if (presenceCacheStr != null && presenceCacheStr != "")
    {
        try { presenceCache = presenceCacheStr.toMap(); }
        catch (e_pc) { presenceCache = Map(); dummy = "caught"; }
    }

    // Parse online count string → number
    curCount = 0;
    if (onlineCountStr != null && onlineCountStr != "")
    {
        try { curCount = onlineCountStr.toLong(); }
        catch (e_cnt) { curCount = 0; dummy = "caught"; }
    }

    // ... logic ...

    // Return updated cache as strings so Flow stores them safely
    ret = Map();
    ret.put("presenceCacheStrUpdated", updatedPresenceCache.toString());
    ret.put("onlineCountStrUpdated", newCount.toString());
    return ret;
}
```

**Flow mappings (Example):**
- Inputs → `presenceCacheStr = ${flow.agentPresenceCache}`; `onlineCountStr = ${flow.agentsOnlineCountCache}`
- After function → Set Variable: `agentPresenceCache = ${UpdateAgentPresence.presenceCacheStrUpdated}`; `agentsOnlineCountCache = ${UpdateAgentPresence.onlineCountStrUpdated}`

### §F.3.2 — Cache Persistence Verification

**What broke (common issues)**
- Set Variable steps mapped to **old output fields** (e.g., `presenceCacheUpdated`, `onlineCountCacheUpdated`) wrote `null` into Flow variables, causing non-persistent caches.
- Some function tiles used **old input names** (e.g., `flowPresenceCache`) instead of the Option‑4 strings.

**Fix**
- Map Set Variable values to the **Option‑4** fields:
  - `agentPresenceCache` = `${<functionOutputAlias>.presenceCacheStrUpdated}`
  - `agentsOnlineCountCache` = `${<functionOutputAlias>.onlineCountStrUpdated}`
- Ensure function parameters are the **Option‑4 signature**:
  `map ... (string flowAgentId, string flowPresenceStatus, string presenceCacheStr, string onlineCountStr)`
  with inputs: `presenceCacheStr = ${flow.agentPresenceCache}` and `onlineCountStr = ${flow.agentsOnlineCountCache}`.

**Expected behavior after fix**
- The **first** run after the change may still show `presenceCacheStr:null` / `onlineCountStr:null` (previous runs didn't persist strings). In that run, Set Variable stores the correct strings; **subsequent** runs receive non-null inputs.

**Verification checklist (Flow history)**
1. *Custom Function → Input* (2nd event in a pair) shows non-null `presenceCacheStr` and `onlineCountStr`.
2. *Custom Function → Output* shows `presenceCacheStrUpdated` and `onlineCountStrUpdated` with expected values.
3. *Set Variable* results store the exact strings (no `null`).
4. (Optional) Analytics rows reflect 1/0 toggles that match the runs.

**Good practice**
- Rename corrected Set Variable steps (e.g., "(Option‑4)") to avoid mixing with older steps.
- Add a temporary **echo** step to log `${flow.agentPresenceCache}` and `${flow.agentsOnlineCountCache}` before the function during stabilization.

### §F.3.3 — CRITICAL: Flow Variables Do NOT Persist Across Trigger Events

**Important Discovery (Feb 2026):** Flow variables are **transaction-scoped**. Each trigger event (e.g., "Agent availability changed") creates a **new transaction**, and Flow variables reset to empty/undefined for each new transaction.

**The Problem:**
- Set Variable steps correctly store values within a transaction
- BUT those values are NOT available to the next separate trigger event
- This makes Flow variables unsuitable for cache/persistence across runs

**The Solution — Analytics-Based Cache:**
For state that must persist across separate Flow executions, use external storage:
1. **Read** previous state from Analytics tables at the start of the function
2. **Write** updated state to Analytics tables at the end
3. Use Analytics v2 REST API from Flow (supported)

**Example Pattern:**
```deluge
// READ from Analytics
cfg = {"responseFormat":"json", "criteria":""key"='some_value'"};
// ... execute GET request ...

// WRITE to Analytics
cfgUpd = {"columns": {...}, "criteria": "...", "addIfNotExist": true};
// ... execute PUT request ...
```

**Quote Escaping in Criteria:**
- Flow parser has issues with `""quoted""` escaping
- Use string concatenation instead: `"\"" + "fieldname" + "\"'='value'"`

---

## §F.4 — Comparison & Control-Flow Rules

### §F.4.1 — Comparison & Control-Flow Rules (Flow-safe)

- **Comparisons:** always coerce and use `.equals()` with null guards:
  ```deluge
  presenceUpper = ifnull(flowPresenceStatus, "").toString().toUpperCase();
  prevUpper = ifnull(prevRaw, "").toString().toUpperCase();
  if (presenceUpper != null && presenceUpper.toString().equals("ONLINE"))
  {
      // ...
  }
  ```
- Replace boolean chaining and `else` with explicit guarded blocks.
- Replace `continue` with nested `if` guards.

### §F.4.2 — DateTime Comparison in Deluge

**CRITICAL: TEXT only supports `==` and `!=` operators. DateTime supports `<`, `>`, `<=`, `>=`.**

**Problem:** ISO 8601 timestamps (e.g., `"2026-03-19T14:30:00Z"`) are TEXT and cannot use `<` or `>` operators directly.

**Solution:** Convert ISO timestamp to Deluge DateTime before comparison:

```deluge
// Input: utRawTime = "2026-03-19T14:30:00Z"
// Step 1: Convert ISO format to Deluge DateTime format
utStr = utRawTime.toString().replaceAll("T", " ").left(19);
// Result: "2026-03-19 14:30:00"

// Step 2: Parse to DateTime object
utDateTime = utStr.toDateTime("yyyy-MM-dd HH:mm:ss");

// Step 3: Compare DateTime objects (supports <, > operators)
if (utDateTime < oldestTime) { isOlder = true; }
```

**DateTime Relational Operators (from Zoho docs):**
- Number, Decimal, Date-time: supports `<`, `>`, `<=`, `>=`, `==`, `!=`
- Text: **only** supports `==`, `!=`

---

## §F.5 — Execution Characteristics & Connections

### §F.5.1 — Execution Latency

**Important Discovery (Mar 2026):** Zoho Flow has execution latency between trigger and custom function execution.

**Observed Behavior:**
- Trigger fires (e.g., "Agent availability updated")
- Flow shows status: "Pending" → "Executing" → "Complete"
- Typical lag: **10-30 seconds** between trigger event and function execution

**Why This Matters:**
- UI shows trigger timestamp, but actual processing happens later
- Apparent "delays" in ticket assignment/recycling are normal
- Execution logs show actual processing timestamps, not trigger timestamps
- Misleading impressions of "stale data" may just be normal latency

**Verification Approach:**
- Check execution log timestamps, not just trigger timestamps
- Allow 15-30 seconds between trigger and result
- "Pending" status is normal, not an error state

### §F.5.2 — Counting Pattern for Presence Tracking

When tracking agent presence and needing accurate counts:

**Problem:** If you READ → COUNT → UPDATE, the count reflects the PREVIOUS state (before update).

**Solution: UPDATE FIRST → READ → COUNT (or READ → UPDATE → READ)**
```deluge
// Step 1: READ current state
allAgents = ... // READ from Analytics

// Step 2: Determine action (recycle/assign) from PREVIOUS state
prevStatus = findAgentStatus(allAgents, currentAgentId);
doRecycle = (newStatus == "OFFLINE" && prevStatus == "ONLINE");

// Step 3: UPDATE Analytics with new status
... // UPDATE agent_status

// Step 4: READ again to get ACCURATE count (post-update)
allAgents2 = ... // READ again
onlineCount = countOnlineAgents(allAgents2); // Now reflects new state
```

This ensures `onlineCount` is accurate in the return value.

### §F.5.3 — Connections Are Environment-Specific

**Each Zoho environment (Flow, Desk, SalesIQ) has its own isolated Connection pool.** A connection created in Desk is NOT visible in Flow, and vice versa.

**Flow Connections** — Created in Flow → Settings → Connections. Flow can use both literal strings and variables for `connection:`.

**Common scopes needed for Flow → Analytics:**
- Zoho Analytics: `ZohoAnalytics.data.all`, `ZohoReports.data.ALL`

**Example (example deployment):**
- `flow2analytics_agents_online` — Zoho Analytics (for Analytics v2 REST: GET /data, PUT /rows, DELETE /rows)
- `desk_agents_online` — Zoho Desk (for `zoho.desk.update` to recycle/assign tickets)

These must be created in **Flow's own connection settings**, not imported from Desk or SalesIQ.

→ See ZohoGuideRoot §1 for Workspace Ownership rules — connection owner must be Workspace Owner for writes.

---

## §F.6 — Analytics v2 REST from Flow

### §F.6.1 — Analytics v2 REST (Flow)

- **Headers**: Always include `"ZANALYTICS-ORGID": <org_id>`.
- **Base URL**: Must be data-center correct (e.g., `analyticsapi.zoho.com` vs `.eu`).
- **Upserts**: Use **Update Row** with `addIfNotExist: true` instead of Bulk Import for state management.
- **Response**: Export responses store records in the `data` key, not `rows`.
- **⚠️ WRITE PERMISSIONS:** PUT /rows will return error 7301 `SECURITY_NOT_PERMITTED` if the connection owner is not the **Workspace Owner** in the target workspace. READ works fine for regular users and Administrators — only writes require Owner role. → See ZohoGuideRoot §1 (Workspace Ownership & Permission Model).
- **⚠️ CONFIG ENCODING:** Always use `zoho.encryption.urlEncode(cfg.toString())` and append to the URL string. Do NOT use `parameters: Map()` for PUT/DELETE — the CONFIG parameter may not be passed correctly, causing silent failures.

**Analytics v2 — Update Row (preferred for state upserts):**
```deluge
cfg = Map();
cols = Map();
cols.put("value", newCount.toString());
cols.put("updated_at", zoho.currenttime.toString("yyyy-MM-dd'T'HH:mm:ss'Z'"));
cfg.put("columns", cols);
cfg.put("criteria", ""router_state"."key"='agents_online_count'");
cfg.put("addIfNotExist", true);
cfgEnc = zoho.encryption.urlEncode(cfg.toString());
url = analytics_domain + "/restapi/v2/workspaces/" + ws_id +
      "/views/" + view_id + "/rows?CONFIG=" + cfgEnc;
headers = {"ZANALYTICS-ORGID": analytics_org_id, "Content-Type":"application/json"};
resp = invokeurl
[
	url: url
	type: PUT
	headers: headers
	connection:"your_oauth_connection"
];
```

**Analytics v2 — Export (read rows):**
```deluge
cfg = {"responseFormat":"json",
       "criteria":""agentId"='1183...'",
       "selectedColumns":{"agentId","presenceStatus","changedAt"}};
cfgEnc = zoho.encryption.urlEncode(cfg.toString());
url = analytics_domain + "/restapi/v2/workspaces/" + ws_id +
      "/views/" + agent_status_view_id + "/data?CONFIG=" + cfgEnc;
resp = invokeurl
[
	url: url
	type: GET
	headers: {"ZANALYTICS-ORGID": analytics_org_id}
	connection:"your_oauth_connection"
];
rows = ifnull(resp.get("data"), List()); // note: 'data', not 'rows'
```

### §F.6.2 — Analytics v1 Rules (Context for Flow)

- **⚠️ `zoho.reports.*` functions are silent no-ops in Desk** — they return success but do NOT write data.
- `zoho.reports.*` only works reliably in **Flow** (see §F.6.1 for v2 REST pattern).
- **⚠️ WRITE PERMISSIONS:** Even with correct API calls, writes will fail with error 7301 if the connection owner is not the **Workspace Owner** (Administrator role is NOT sufficient). → See ZohoGuideRoot §1 (Workspace Ownership & Permission Model).

### §F.6.3 — Write Permissions Warning (Error 7301)

**The #1 trap:** A connection can have `ZohoAnalytics.data.all` scope and show "Connected" status, yet STILL fail on writes. Even having **Workspace Administrator** role is not sufficient. The connection owner must be the **Workspace Owner**.

**Error 7301:** `"You (Your_Name) do not have the permission to do this operation. Only Workspace Admin has the permission."`

**Misleading error message:** The API says "Workspace Admin" but in practice, **only the Workspace Owner** can perform INSERT/UPDATE/DELETE via API.

**Flow vs Desk — different rules:**

| Environment | Can Workspace Administrator Write? | Can Workspace Owner Write? |
|-------------|-----------------------------------|---------------------------|
| **Zoho Flow** | ❌ **Read-only** — Admins can read via `invokeUrl` GET, but PUT/POST/DELETE fail with 7301 | ✅ Full read/write |
| **Zoho Desk** | ✅ **Can write** — Desk does NOT enforce the Owner-only rule | ✅ Full read/write |

**Symptom in Flow:** READ operations (GET /data) succeed. WRITE operations (PUT /rows, POST /rows) fail with `SECURITY_NOT_PERMITTED`.

→ See ZohoGuideRoot §1.7 for propagation delay (30+ minutes after ownership fix).
→ See ZohoGuideRoot §1.8 for creating new connections as permission reset.

### §F.6.4 — CONFIG Encoding Warning

**Always use `zoho.encryption.urlEncode()`** for CONFIG parameters — it handles all special characters correctly. Do NOT use manual `replaceAll` encoding (fragile, misses characters) or `parameters: Map()` (unreliable for PUT/DELETE in Flow — CONFIG may not be passed correctly).

**Availability by environment:**

| Environment | Method | Availability |
|-------------|--------|-------------|
| Flow | `zoho.encryption.urlEncode()` | ✅ Works |
| Desk | `zoho.encryption.urlEncode()` | ❌ Not available |
| SalesIQ | `zoho.encryption.urlEncode()` | ❌ Parse error — `zoho` not a namespace |

---

## §F-A — Flow Lessons Learned

### §F-A.0 — zoho.appdetails() NOT Available in Flow

**Discovery (2026-03-05):** `zoho.appdetails()` does NOT work in Flow custom functions — it causes a parser error "Not able to find 'appdetails' function".

**Impact:** Cannot use dynamic orgId discovery in Flow. Must use static constants.

**Solution:** Use hard-coded static values (see ZohoGuideRoot §1.1 canonical template):
```deluge
staticOrgId = "YOUR_ORG_ID";
```

### §F-A.1 — Blank Lines Before try Block (Flow-safe, Desk-unsafe)

**Flow:** Blank lines before try blocks are safe.

**Desk:** A blank line between variable declarations and the `try` keyword can cause "Improper Statement" errors. (Desk-specific rule documented in ZohoGuideRoot §7.9.1.)

### §F-A.2 — READ → UPDATE → READ Pattern

**Pattern (2026-02-27):** When updating state that affects counts, always READ → UPDATE → READ to get accurate counts.

**Example from UpdateAgentPresence:**
```deluge
// 1. READ all agents from Analytics
allAgents = ... // READ

// 2. UPDATE this agent's status
zoho.desk.update(...); // UPDATE

// 3. READ again to get ACCURATE count (post-update)
allAgents2 = ... // READ
onlineCount = countOnlineAgents(allAgents2); // Now reflects new state
```

**Why:** If you READ → COUNT → UPDATE, the count reflects the PREVIOUS state, not the new state.

### §F-A.3 — String Comparison with .equals()

**Important:** In Flow, use `.equals()` for string comparison, not `==`:
```deluge
// WRONG - may not work in Flow
if (presenceStatus == "ONLINE")

// CORRECT - use .equals()
if (presenceStatus.equals("ONLINE"))

// With null safety
if (presenceStatus != null && presenceStatus.equals("ONLINE"))
```

### §F-A.4 — Encryption URL Encode for Analytics v2

**Required** when passing CONFIG to Analytics v2 REST API:
```deluge
cfg = Map();
cfg.put("columns", cols);
cfg.put("criteria", criteria);
cfg.put("addIfNotExist", true);

cfgEnc = "";
try { cfgEnc = zoho.encryption.urlEncode(cfg.toString()); }
catch (e) { dummy = "caught"; cfgEnc = ""; }

url = analytics_domain + "/restapi/v2/workspaces/" + ws_id + 
      "/views/" + view_id + "/rows?CONFIG=" + cfgEnc;
```

### §F-A.5 — Null-Safe Input Parameters

**Always use `ifnull()`** for Flow inputs that may be empty:
```deluge
orgId = ifnull(flowOrgId, "");
presenceStatus = ifnull(flowPresenceStatus, "").toString().toUpperCase();
```

---

## §F-G — Flow AI Guardrails

### §F-G.1 — Analytics v2 REST Batch Constraints

- **Rule**: When using the preferred `PUT` method on `/rows` with `addIfNotExist: true` for bulk state updates, the payload MUST be capped at a maximum of **100 rows per transaction**.
- **Resolution**: If an agent processes an incoming list larger than 100 items, it must chunk the payload or process it via sequential execution, otherwise Analytics will reject the request with a payload size exception.

### §F-G.2 — Transaction Scope and Loop Re-initialization

- **Rule**: When running a `for each` loop inside Zoho Flow that contains a `try/catch` block, state variables must be re-initialized at the immediate top of the loop body to prevent transaction-scope freezing.
- **Pattern**:
```deluge
for each item in itemsList
{
    // Explicit loop-scope reset
    currentStatus = "";
    itemValid = true;

    try {
        currentStatus = item.get("status");
        // process
    } catch(e) {
        itemValid = false;
        dummy = "caught";
    }
}
```

### §F-G.3 — Environment Write Paths (Flow-Relevant)

**Key lesson from root §3.5:** Always use `zoho.encryption.urlEncode()` for CONFIG parameters — it handles all special characters correctly. Do NOT use manual `replaceAll` encoding (fragile, misses characters) or `parameters: Map()` (unreliable for PUT/DELETE in Flow — CONFIG may not be passed correctly).

**Flow → Analytics write pattern:**
```deluge
cfg = Map();
cols = Map();
cols.put("column1", value1);
cfg.put("columns", cols);
cfg.put("addIfNotExist", true);
cfg.put("criteria", "1=2");
cfgStr = cfg.toString();
encStr = zoho.encryption.urlEncode(cfgStr);
putUrl = analyticsDomain + "/restapi/v2/workspaces/" + wsId + "/views/" + viewId + "/rows?CONFIG=" + encStr;
resp = invokeurl
[
    url: putUrl
    type: PUT
    headers: {"ZANALYTICS-ORGID": orgId, "Content-Type": "application/json"}
    connection: "analyticsconnection"
];
```

**Common AI hallucination for Flow:**
| Hallucination | Reality |
|---------------|---------|
| `parameters: Map()` works for PUT/DELETE CONFIG in Flow | ❌ Unreliable — CONFIG may not be passed correctly. Always use `zoho.encryption.urlEncode()` + URL string for PUT/DELETE. `parameters:` works for GET query params. |
| `ifelse(condition, trueVal, falseVal)` works in Flow | ❌ NOT available — causes parser error. Use explicit `if/else` blocks. |

→ See ZohoGuideRoot §3.5 for the full hallucination risk table.
