# Zoho Compliance & Deluge Coding Standards Guide

This document is the authoritative standard for all Deluge development across Zoho Flow, Zoho Desk, and Zoho Analytics. It consolidates all historical compliance rules, parser caveats, and environmental constraints.

**Last Updated:** 2026-06-19 (ZQL date function additions §4.4.5)

> **⚠ AI AGENT INSTRUCTION:** Act as an expert compiler. You are constrained by the physical execution quirks of the Zoho engine. You must review every line of code you generate against **§7 (Lessons Learned & Caveats)** and **§9 (AI Agent Execution Guardrails)** to verify you haven't introduced an environmental syntax error. These sections represent live, empirically verified discoveries from real-world testing (Feb–Jun 2026) and override generic or outdated guidance found elsewhere.

---

## 1. Global Mandates (Universal Deluge Rules)
- **ALWAYS** use the canonical function template in §1.1.
- **USE STATICS**: Hard-code Org IDs, tokens, and data-center URLs as named **static constants** at the top of the function. This makes code portable and cloneable to other orgs.
- **STATIC NAMING**: Use clear static variable names (e.g., `staticOrgId`, `staticAnalyticsDomain`) so they're easily findable when cloning.
- **EXCEPTION**: `zoho.appdetails()` does NOT work in either Flow or Desk. Use static constants (see §1.1 canonical template).
- **ERROR HANDLING**: Wrap all logic in `try/catch`. Return or log a structured map with `success`, `data`, and `error` keys (except in Desk, which is VOID).
- **ENVIRONMENT AWARENESS**: Rules for Desk and Flow differ significantly. Apply the correct constraints based on the execution environment.

### 1.1 Canonical Function Template
```deluge
// ======================================
// Function: <FunctionName>
// Purpose : <Short description>
// Inputs  : <List inputs>
// Returns : <Return value description>
// ======================================

// --- Static Constants ---
// Clone and update these for each deployment
staticOrgId = "YOUR_ORG_ID";
staticAnalyticsOrgId = "YOUR_ANALYTICS_ORG_ID";
staticAnalyticsDomain = "https://analyticsapi.zoho.com"; // or .eu
staticWsId = "YOUR_WORKSPACE_ID";
staticViewId = "YOUR_VIEW_ID";

// --- Main Logic ---
try
{
	// --- Input Validation ---
	if (input == null || input.isEmpty())
	{
		return {"success": false, "error": "Invalid input"};
	}
	// <Your code here>
	return {"success": true, "data": result};
}
catch (e)
{
	// --- Standardized Error Logging ---
	log = Map();
	log.put("Function", "<FunctionName>");
	log.put("Error", e.toString());
	log.put("Timestamp", zoho.currenttime.toString());
	// In Flow, ensure catch blocks assign a value (e.g., dummy = "caught";)
	return {"success": false, "error": e.toString()};
}
```

### 1.2 Standard API Call Pattern (generic)
```deluge
headers = {"Authorization": "Zoho-oauthtoken " + zoho.authtoken};
url = "https://www.zohoapis.com/crm/v5/<module>?organization_id=" + orgId;
resp = invokeurl
[
	url: url
	type: GET
	headers: headers
];
if (resp.get("code") == "SUCCESS")
{
	// process
}
if (resp.get("code") != "SUCCESS")
{
	// handle error
}
```

### 1.2.1 Advanced API Call Patterns (invokeurl)

**POST/PUT with JSON body:**
```deluge
headers = Map();
headers.put("Authorization", "Zoho-oauthtoken " + zoho.authtoken);
headers.put("Content-Type", "application/json");

body = Map();
body.put("field1", value1);
body.put("field2", value2);

resp = invokeurl
[
	url: url
	type: POST       // or PUT
	headers: headers
	body: body       // Map auto-serialized to JSON
];
```

**Body as raw JSON string (use when Map body causes errors):**
```deluge
// Pre-serialized string bypasses Map→KEY-VALUE issues in Flow
bodyStr = "{\"field1\": \"" + value1 + "\", \"field2\": \"" + value2 + "\"}";
resp = invokeurl
[
	url: url
	type: POST
	headers: headers
	body: bodyStr
];
```

**Non-2xx response handling — ALWAYS wrap in try/catch:**
`invokeurl` throws a script error on HTTP 4xx/5xx; it does NOT return a response map.
```deluge
try
{
	resp = invokeurl [ url: url type: GET headers: headers ];
	// Only reached on 2xx
}
catch (e)
{
	return {"success": false, "error": "API call failed: " + e.toString()};
}
```

**Flow vs Desk differences:**
- **Flow**: `body` as Map is auto-serialized to JSON. String body also works.
- **Desk**: `body` as Map may fail; prefer pre-serialized JSON string.
- **Connection-based calls**: Use `connection:"conn_name"` in invokeurl for OAuth-managed tokens. Non-2xx behavior is the same — still throws.
- **Multi-part forms**: Not supported natively in invokeurl. Use raw string body with `Content-Type: multipart/form-data` and manual boundary encoding, or avoid multi-part in Deluge.

### 1.3 Best-Practice Rules
**Required**
- Wrap logic in **try/catch**.
- Validate inputs first; fail fast with a clear error message.
- Document function header (name, purpose, inputs, returns).
- Define `staticOrgId` and other static constants at top of function (see §1.1 template).
- Return structured JSON: `{ success: bool, data: map/list, error: string }`.

**Avoid**
- Passing IDs as Flow variables (makes code non-portable).
- Silent failures—log or return structured errors.
- Creating records inside loops unless unavoidable.

### 1.4 Pre-Deployment Checklist (Functional & Security)
- [ ] Header present
- [ ] Inputs validated
- [ ] `try/catch` implemented
- [ ] `orgId` loaded dynamically
- [ ] No hard-coded secrets
- [ ] All failures return `{success:false}`
- [ ] Errors logged consistently
- [ ] Structured map output only (no raw text)

### 1.5 Centralized Logger (optional helper)
```deluge
function logError(moduleName, functionName, message)
{
	logMap = Map();
	logMap.put("Module", moduleName);
	logMap.put("Function", functionName);
	logMap.put("Message", message);
	logMap.put("Timestamp", zoho.currenttime.toString());
	// Optional: insert into a Logs table or Desk comment
}
```

### 1.6 Map/List Auto-Casting Trap (CRITICAL)

**Problem:** Deluge silently casts between Map and List in specific scenarios, causing `.get()` to fail with "Not a Map" errors:

| Scenario | Input | What Deluge Does | The Trap |
|----------|-------|------------------|----------|
| Single-element JSON array | `[{"id": 1}]` | Returns a **Map** `{"id": 1}`, not a List | Calling `.get(0)` on a Map throws |
| Map with numeric keys | `{"0": "a", "1": "b"}` | Returns a **List** `["a", "b"]`, not a Map | Calling `.get("key")` on a List throws |

**Always check structure dynamically when the response format is uncertain:**

```deluge
result = invokeurl [ url: url type: GET headers: headers ];

// Check: is it a List? (supports .get(index))
isList = false;
try { dummy = result.get(0); isList = true; }
catch (e) { isList = false; }

// Check: is it a Map? (supports .get("key"))
isMap = false;
try { dummy = result.get("data"); isMap = true; }
catch (e) { isMap = false; }

// Check for nested "data" key (common response wrapper)
items = List();
if (isMap && result.containsKey("data"))
{
    dataVal = result.get("data");
    // dataVal itself may be auto-cast — check again
    try { dummy = dataVal.get(0); items = dataVal; }
    catch (e) { items = List(); items.add(dataVal); }
}
else if (isList)
{
    items = result;
}
```

**Key rule:** Never assume the type of an API response. Always probe with try/catch before accessing.

---

## 2. Zoho Flow Standards

### 2.1 Parser Caveats (Forbidden/Unreliable Constructs)
Zoho Flow's Deluge parser is stricter than other environments. **Avoid these constructs:**
- `instanceof` — not supported.
- `continue` — not supported; replace with nested `if` guards.
- `else` — often misparsed; prefer separate `if` blocks or early returns.
- **Custom helper function declarations** (e.g., `void logDebug(...)`) — not allowed inside Flow custom functions.
- `info` statements — rejected by Flow; use **comment-only debug lines** instead.
- **Chained boolean expressions** (e.g., `if(a && b)`) — expand to separate guarded booleans if parsing fails.
- **Empty `catch` blocks** — Always add a harmless assignment (e.g., `dummy = "caught";`).
- **"Loose expressions"** (assignments that Flow thinks sit outside a block) — ensure braces are balanced and avoid risky constructs above.
- **`sortBy` function** — **NOT available in Deluge**. To sort a list of maps by a field, use extract-sort-rebuild. Explicit pattern:
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

### 2.2 Expression/Mapping Caveats (Flow UI)
- Dragged variables from the picker are rendered lower-case inside expressions (e.g., `Status` → `${trigger.status}`).
- In **Test** overlays, early evaluation can yield empty strings for casts like `toString(...)`; live runs resolve correctly.
- If any mapped input (trigger field or Flow variable) resolves to a **KEY-VALUE** object, Flow may inject an internal validator like `<KEYVALUE> == "TEXT"` and fail with _"left expression is KEY-VALUE, right expression is TEXT"_. Avoid by ensuring text coercion and/or string-only storage.

### 2.3 Variable Initialization & State (critical)
- Newly created Flow variables are **untyped** until first set; empty Map/Number vars behave like KEY-VALUE wrappers in validation.
- Test/Debug does **not** persist variable values; only live runs persist state.
- **Remedies:**
  1) Run a one-time initializer path (two **Set Variable** steps) before the function; or
  2) **Recommended:** store caches as **strings** in Flow and parse inside Deluge (§2.5). This permanently avoids KEY-VALUE validation issues.

### 2.4 Function Signature & First-Line Rules (Flow)
- First non-whitespace line **must** be the signature, and `{` must be on the same line:
  ```deluge
  map MyFunc(string a, string b)
  {
      // body
  }
  ```
- Explicit primitive types are required for all parameters (e.g., `string`, `int`).
- **ONLY include Flow trigger inputs** in the signature. Static constants (orgId, domain, workspace IDs) should be defined as constants inside the function body, not passed from Flow. This makes the function portable and cloneable.

### 2.5 String-Based Cache Pattern (Flow-Safe State Across Runs)
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

### 2.6 Comparison & Control-Flow Rules (Flow-safe)
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

### 2.6.1 DateTime Comparison in Deluge (Mar 2026)
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

**Note:** If Desk API returns timestamps in a different format, adjust the parsing accordingly.

### 2.7 Flow Cache Persistence – Final Notes & Verification
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
1) *Custom Function → Input* (2nd event in a pair) shows non-null `presenceCacheStr` and `onlineCountStr`.
2) *Custom Function → Output* shows `presenceCacheStrUpdated` and `onlineCountStrUpdated` with expected values.
3) *Set Variable* results store the exact strings (no `null`).
4) (Optional) Analytics rows reflect 1/0 toggles that match the runs.

**Good practice**
- Rename corrected Set Variable steps (e.g., "(Option‑4)") to avoid mixing with older steps.
- Add a temporary **echo** step to log `${flow.agentPresenceCache}` and `${flow.agentsOnlineCountCache}` before the function during stabilization.

### 2.6 CRITICAL: Flow Variables Do NOT Persist Across Trigger Events
**Important Discovery (Feb 2026):** Flow variables are **transaction-scoped**. Each trigger event (e.g., "Agent availability changed") creates a **new transaction**, and Flow variables reset to empty/undefined for each new transaction.

**The Problem:**
- Set Variable steps correctly store values within a transaction
- BUT those values are NOT available to the next separate trigger event
- This makes Flow variables unsuitable for cache/persistence across runs

**The Solution - Analytics-Based Cache:**
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

### 2.7 Flow Execution Latency (Mar 2026)
**Important Discovery:** Zoho Flow has execution latency between trigger and custom function execution.

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

### 2.8 Counting Pattern for Presence Tracking (Feb 2026)
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

---

## 3. Zoho Desk Standards

### 3.1 Capabilities & Constraints
- **VOID Return**: Desk custom functions are `VOID`. Do not rely on returned values. Persist state via Analytics or Desk API.
- **Analytics v1 Only**: Desk Deluge *only* supports **Analytics v1 (Reports tasks)** (e.g., `zoho.reports.createRow()`).
- **REST Restrictions**: Direct Analytics v2 REST calls from Desk Deluge will fail with `URL_RULE_NOT_CONFIGURED`. Use Flow as an intermediary if v2 features are required.
- Always use **Connections** rather than hard-coded tokens; keep environment (sandbox vs production) configurable.

### 3.2 Desk Patterns & Snippets
- Use static constants: `staticOrgId = "YOUR_ORG_ID"` at top of function.
- For Analytics writes/reads from Desk, use **v1 Reports tasks**. If you need Analytics v2 features, **invoke Flow** or an intermediary service.
- Log errors compactly and avoid verbose debug logs in production.

**Analytics v1 example (write):**
```deluge
// Example: create a row via Reports (v1)
row = Map();
row.put("field1", value1);
row.put("field2", value2);
// zoho.reports.createRow("workspace", "table", row);
```

**Desk-safe function skeleton (VOID return):**
```deluge
// Static constants
staticOrgId = "YOUR_ORG_ID";
try
{
	// ... do work, persist state via Reports v1 or Desk APIs
}
catch (e)
{
	// log only; Desk will not consume returned maps
	logError("DeskModule", "FunctionName", e.toString()); // Using optional centralized logger
}
```

### 3.3 Zoho Desk API Gotchas & Known Issues (Mar 2026)

**Two API layers — know which one you're using:**

| Layer | How to Call | Parameter Rules |
|-------|-------------|-----------------|
| **Native REST** | `invokeurl` to `https://desk.zoho.com/api/v1/...` | Full HTTP — query params, headers, pagination via URL params |
| **Deluge Integration Task** | `zoho.desk.searchRecords()`, `zoho.desk.getTicketById()`, etc. | Wrapper — uses internal Desk API but **parameter names and limits differ** from the raw REST layer |

Key differences:
- `searchRecords` uses a **criteria Map**, not URL query strings
- `searchRecords` limit max is **100**, not 200 as the REST API accepts. Raw REST `invokeurl` calls can use `limit=200` per page — the 100 cap applies only to the `zoho.desk.*` wrapper.
- `searchRecords` does NOT support `sortField`/`sortOrder` — the raw REST endpoint also rejects these params with `UNPROCESSABLE_ENTITY`
- `searchRecords` does NOT support `statusType` — use `status` instead

- **tickets endpoint**: `GET /api/v1/tickets` does NOT support `sortField` or `sortOrder` query parameters — will return `UNPROCESSABLE_ENTITY` error.
- **status in searchRecords**: The `status` parameter WORKS in searchRecords. Use: `crit.put("status", "Open")`. Do NOT use `statusType` — it is unreliable across all environments (see §7.10).
- **tickets endpoint**: Avoid `isStatusChanged` parameter — not valid for this endpoint.
- **Response handling**: API may return tickets in `data` key, or `tickets` key — always check both paths with `ifnull()`.
- **searchRecords returns List or Map**: `zoho.desk.searchRecords()` can return either a List directly OR a Map with a "data" key. Always handle both formats (see §7.5).
- **statusType field**: The ticket object contains `statusType` field (e.g., "Open", "Closed") but it does NOT work as a search filter in any environment. Always use `status=Open` instead — see §7.10.
- **searchRecords filter param**: Do NOT pass `null` or `{}` for empty criteria. Use `crit = Map()` (assign to variable first), then pass the variable.
- **No while loops**: Desk Deluge does not support `while` loops or `range()` function. Use explicit sequential fetches for pagination.
- **zoho.appdetails()**: Does NOT work in Flow or Desk. Use static constants (orgId, analytics_org_id, etc.) defined at top of function.
- **Pagination**: Use explicit sequential fetches (e.g., `searchRecords(orgId,"tickets",crit,0,100)`, then `searchRecords(orgId,"tickets",crit,100,100)`). No loops allowed.
- **searchRecords limit max**: The `limit` parameter **max value is 100**, not 200. Values >100 return `UNPROCESSABLE_ENTITY` error: "The value passed for field 'limit' exceeds the range of '1-100'."
- **Channel field case sensitivity**: The ticket `channel` field returns `"Email"` (capital E), NOT `"EMAIL"`. Use case-insensitive comparison: `channelVal.toString().toUpperCase().equals("EMAIL")`.
- **Channel values (Analytics import, Jun 2026):** Beyond Email/Chat/Phone, SalesIQ integration creates per-platform tags: `FB Messenger`, `IG Messenger`, `Instagram`, `Facebook`. The Analytics import preserves these verbatim. When filtering in ZQL or dashboard widgets, expect the full set.
- **Channel filter in searchRecords**: The `channel` parameter WORKS in Desk searchRecords. Tested values: `"Email"`, `"Chat"`, `"Phone"`. Use: `crit.put("channel", "Chat")`.
- **Empty criteria Map required**: Must use `critEmpty = Map()` then pass variable. Inline `{}` causes "COLLECTION" type error.

---

## 4. Zoho Analytics & ZQL (SQL) Standards

### 4.1 Analytics v2 REST (Flow Only)
- **Headers**: Always include `"ZANALYTICS-ORGID": <org_id>`.
- **Base URL**: Must be data-center correct (e.g., `analyticsapi.zoho.com` vs `.eu`).
- **Upserts**: Use **Update Row** with `addIfNotExist: true` instead of Bulk Import for state management.
- **Response**: Export responses store records in the `data` key, not `rows`.

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

### 4.2 Analytics v1 Rules (Desk Only)
- Only option inside Desk Deluge.
- Use `zoho.reports.updateData`, `createRow`, etc.

### 4.2.1 Deleting Rows (Empirically Verified 2026-03-02)

**Function:** `zoho.reports.deleteRow`

**Syntax:**
```deluge
zoho.reports.deleteRow(<database_name>, <table_name>, <criteria>, <connection>);
```

**Example - Delete all rows:**
```deluge
// Delete all rows where departmentId is not null
result = zoho.reports.deleteRow("your_workspace_name", "your_table_name", "departmentId is not null", "your_oauth_connection");
// Returns: {"deletedrows":"10","message":"Deleted Rows"}
```

**Example - Delete specific rows:**
```deluge
// Delete rows where departmentId = 123
result = zoho.reports.deleteRow("Workspace", "Table", "departmentId = 123", "connection");
```

**Criteria Operators:** `=`, `!=`, `<`, `>`, `<=`, `>=`, `LIKE`, `NOT LIKE`, `IS NULL`, `IS NOT NULL`

**Note:** This function is available in Desk Deluge - tested and verified.

### 4.3 ZQL (Zoho Query Language) — Basics & Templates

**What is ZQL?** ZQL (Zoho Query Language) is Zoho's SQL dialect used in Analytics Query Tables. It supports standard SQL SELECT statements with some variations.

**Key ZQL Rules:**
- **Comments:** Use `/* */` syntax (not `//`)
- **Column names:** Use double quotes: `"columnName"`
- **String values:** Use single quotes: `'value'`
- **Row limiting:** Use `FETCH FIRST X ROWS ONLY`
- **Type conversion:** Use `to_number(column)`

#### ZQL Templates (Project-Tested)

**Template 1: Count with Filter**
```sql
SELECT COUNT(*) FROM tableName WHERE columnName = 'value'
```

**Template 2: Latest Row by Timestamp**
```sql
SELECT columns...
FROM tableName 
ORDER BY timestamp DESC 
FETCH FIRST 1 ROWS ONLY
```

**Template 3: Latest with Type Conversion**
```sql
SELECT to_number(value) AS counterName
FROM tableName 
WHERE "key" = 'specific_key'
ORDER BY updated_at DESC 
FETCH FIRST 1 ROWS ONLY
```

**Template 4: Latest Per Group (Join Pattern)**
```sql
SELECT a.*
FROM tableName a
JOIN(
    SELECT id, MAX(timestamp) AS last_change
    FROM tableName 
    GROUP BY id
) j ON a.id = j.id AND a.timestamp = j.last_change
```

**Template 5: Group By Count**
```sql
SELECT columnName, COUNT(*) AS count
FROM tableName
GROUP BY columnName
```

---

### 4.4 ZQL (Zoho Analytics SQL) — Practical Constraints & Patterns

#### 4.4.1 General Observations
- **No SQL in KPI widgets** → KPIs must bind to a dataset (table/query table) and aggregate a **single row**; use Query Tables for any SQL logic and then point the KPI to that Query Table.
- **Type conversion**: Prefer `to_number(<expr>)` over `CAST(... AS ...)` for numeric coercion.
- **Row limiting**: Some engines reject `LIMIT`; use `FETCH FIRST 1 ROWS ONLY` or a **MAX(timestamp)** subquery.
- **Identifiers**: Quote reserved/mixed-case names with **double quotes** (e.g., `"key"`, `"timestamp"`).
- **Time math**: If `NOW() - INTERVAL '24' HOUR` is unsupported, use `dateadd('hour', -24, current_timestamp())`.
- **Report cache vs table types**: After changing a column to number in a Query Table, the **report** can still cache it as *Actual(text)*. Re-add the field as a **measure (Sum/Avg)** or **Reset All** to refresh bindings.

#### 4.4.2 Reliable Patterns Adopted
- **Latest value KPIs** via Query Tables that return **one row** (e.g., latest OnlineCount or RouterState), so KPI aggregation becomes a no-op.
- **Heatmaps/Trends** via Query Tables that pre-aggregate (`DATE()`, `HOUR()`, `AVG(...)`).
- **24h windows** via `ADDDATE(LOCALTIMESTAMP(), -1)` for cross-workspace compatibility (day-level only; see §4.4.5 for hour-level caveats).
- **Numeric enforcement** via `to_number(...)` inside the SELECT.

#### 4.4.3 Minimal ZQL Snippets (project-tested)

**Note:** Comments use `/* */` syntax (not `//` or `--`). Comments can appear anywhere in the query.

- **Latest by timestamp (no LIMIT):**
  ```sql
  SELECT cols... FROM T
  WHERE ts = (SELECT MAX(ts) FROM T)
  ```
- **Latest with FETCH (when supported):**
  ```sql
  SELECT cols... FROM T
  ORDER BY ts DESC
  FETCH FIRST 1 ROWS ONLY;
  ```
- **Type conversion:**
  ```sql
  SELECT to_number(value) AS val FROM T
  ```
- **Day window (cross-workspace):**
  ```sql
  WHERE ts >= ADDDATE(LOCALTIMESTAMP(), -1)
  ```
  ⚠ **Avoid `dateadd()`:** Not available in `YOUR_WORKSPACE_NAME` workspace. `ADDDATE()` is preferred and works in all tested workspaces (see §4.4.5). Note: `ADDDATE()` supports day-level only; for hour-level windows, see §4.4.5 for workspace-specific options.

#### 4.4.4 Imported Desk Tables - Column Names (Mar 2026)

**IMPORTANT:** When importing Zoho Desk data into Analytics, the column names shown in the UI are **display names**, not the ZQL column names.

**Empirical Findings (YOUR_CLIENT_NAME - 2026-03-16, updated 2026-06-19):**

| Display Name | ZQL Column Name | Notes |
|--------------|-----------------|-------|
| ID | `"ID"` | Must use "ID", not "id" |
| Subject | `"Subject"` | Must use "Subject", not "subject" |
| Status | `"Status"` | Must use "Status", not "status" |
| Channel | `"Channel"` | Must use "Channel", not "channel" |
| Department | `"Department"` | Department ID |
| Ticket Owner | `"Ticket Owner"` | Agent ID — JOIN key to `agents.ID` |
| Created Time | `"Created Time"` | Datetime string |
| Ticket Closed Time | `"Ticket Closed Time"` | Close datetime (NOT `"Closed Time"`) |
| Modified Time | `"Modified Time"` | Last modified datetime |
| statusType | N/A | Does NOT exist in imported table |

**Verified Working ZQL:**
```sql
SELECT "ID", "Subject", "Status", "Channel"
FROM "tickets"
GROUP BY "Channel"
```

**Agents table** also imported to same workspace:
| Column | ZQL Name | Notes |
|--------|----------|-------|
| Agent ID | `"ID"` | JOIN key to `tickets."Ticket Owner"` |
| Display Name | `"Name"` | e.g. "John Doe" |
| Email | `"Email"` | |
| Status | `"Status"` | ACTIVE / DELETED / SYSTEM_ACTIVE |

**JOIN pattern (tested):**
```sql
SELECT agents.Name, tickets.Channel, COUNT(*)
FROM "tickets" tickets
JOIN "agents" agents ON tickets."Ticket Owner" = agents.ID
WHERE tickets.Status = 'Closed' AND agents.Status != 'DELETED'
GROUP BY agents.Name, tickets.Channel
```

**Query Tables vs Live Data:**
- Query Tables read from **Analytics (synced data)**, NOT live Desk API
- If Desk sync is broken/paused, Query Table data is stale
- For live data testing, use **Deluge smoke tests** (`zoho.desk.searchRecords`)
- No inherent row limit in ZQL - limit issues were due to sync problems

### 4.4.5 Empirically Verified ZQL Date Functions (Jun 2026)

Tested in `YOUR_WORKSPACE_NAME` workspace. ZQL dialect varies by workspace.

| Need | Working Syntax | Example Output |
|------|---------------|---------------|
| Current timestamp | `LOCALTIMESTAMP()` | `19 Jun 2026 08:19:43` |
| Date arithmetic (n days ago) | `ADDDATE(LOCALTIMESTAMP(), -30)` | `20 May 2026 08:22:45` |
| Extract date from datetime | `DATE(column)` | `22 Jan 2026` |
| Extract hour (0-23) | `EXTRACT(HOUR FROM column)` | `23` |

**Time difference functions:**
- `TIMESTAMPDIFF(MINUTE, start, end)` — returns minutes between two timestamps ✅
- `DATEDIFF(end, start)` — returns days only (too coarse for sub-day)
- `TIMEDIFF(end, start)` — returns time string but overflows on multi-day spans
- `TO_SECONDS(col)` — NOT supported

**Common pitfalls:**
- `dateadd()` does NOT work regardless of datepart quoting (`'dd'`, `dd`, `'day'` all fail)
- `HOUR(column)` — unsupported function (use `EXTRACT(HOUR FROM column)` instead)
- `CLOSED_DATE()` not a valid function (use `DATE()` instead)
- `current_date` not supported as standalone keyword (use `LOCALTIMESTAMP()`)

#### 4.4.6 Duration String Parsing — REPLACE Auto-Converts to Seconds (Jun 2026)

**Discovery:** Zoho's duration format `"X days YY:ZZ hrs"` (e.g., `"0 days 00:02 hrs"`) is stored as text, but passing it through `REPLACE(col, 'days ', '')` **auto-converts to total seconds**.

**Working pattern:**
```sql
-- Returns total seconds (numeric) — divide by 60 for minutes
REPLACE(tickets."Total Response Time in Business Hours", 'days ', '') / 60 AS response_minutes

-- Can be aggregated with AVG
AVG(REPLACE(tickets."Total Response Time in Business Hours", 'days ', '') / 60) AS avg_response_minutes
```

**Tested examples (YOUR_WORKSPACE_NAME):**
| Input | REPLACE Result (seconds) | / 60 (minutes) |
|-------|--------------------------|-----------------|
| `0 days 00:01 hrs` | 60 | 1 |
| `0 days 02:30 hrs` | 9,000 | 150 |
| `0 days 11:22 hrs` | 40,920 | 682 |
| `4 days 00:01 hrs` | 345,660 | 5,761 |

**Why this works:** ZQL's `REPLACE` on a duration-formatted column implicitly evaluates the duration expression rather than doing string replacement. The `'days '` and `' hrs'` arguments appear to trigger numeric extraction, returning seconds.

**Known limitations:**
- `REPLACE` must be called with the duration column as the first argument
- The `'days '` or `' hrs'` text must be present in the value for the conversion to trigger
- Tested only with `"Total Response Time in Business Hours"` field in Desk→Analytics import
- Does NOT work with plain `to_number()` or `CAST()` — only `REPLACE` triggers the conversion

---

## 5. Specific Design Patterns

The following are generalized architectural best practices derived from common Zoho Desk + Analytics patterns. They are not client-specific code; adapt the table/column names to your own workspace.

### 5.1 Presence Delta Pattern (Analytics)
- **Logic**:
  - `ONLINE` from not-online → +1
  - `OFFLINE` from online → -1
  - Always clamp values: `if(count < 0) { count = 0; }`
- **Order**: Update individual `agent_status` before updating the aggregate `router_state`.

### 5.2 Debug Comment Mentor & Samples
Since Flow blocks `info`, use comment breadcrumbs for stable tracing in the script editor. Desk allows `info` but it should be minimal.

**Flow sample (from UpdateAgentPresence):**
```deluge
// DEBUG: [A] curCount=<curCount> delta=<delta> newCount=<newCount>
// DEBUG: [A5] router_state upsert done
// DEBUG: [A6] agent_status upsert done for agentId=<agentIdStr>
```

**Desk sample (TicketRouter pattern):**
```deluge
// DEBUG: computing open counts by department
// DEBUG: zoho.reports.updateData() response captured
// DEBUG: createRow fallback executed when updateData matched 0 rows
```

---

## 6. Consolidated Pre-Deployment Checklist
- [ ] **Header**: Function name, purpose, and inputs documented.
- [ ] **Validation**: Input null/empty checks implemented.
- [ ] **Safety**: All logic wrapped in `try/catch`.
- [ ] **Environment**:
  - If **Flow**: No `else`, `continue`, `instanceof`, custom helper functions, or `info` statements. Function signature on the first non-whitespace line with `{` on the same line.
  - If **Desk**: No return values; Analytics v1 tasks only (`zoho.reports.createRow`, `updateData`).
- [ ] **Analytics**: `ZANALYTICS-ORGID` header included for v2 calls (Flow only).
- [ ] **Hard-coding**: No Org IDs, hard-coded URLs, or tokens.
- [ ] **Flow Cache Persistence**: If using cache, ensure string-based pattern is correctly implemented and Flow variable mappings are updated per §2.7.
- [ ] **ZQL**: If using Analytics SQL, ensure compliance with ZQL constraints (no SQL in KPIs, proper type conversion, row limiting).

---

## 7. Lessons Learned & Caveats (2026-02-26)

This section documents real-world discoveries from development sessions that override or clarify earlier guidance.

### 7.0 zoho.appdetails() NOT Available in Flow (2026-03-05)

**Discovery:** `zoho.appdetails()` does NOT work in Flow custom functions - it causes a parser error "Not able to find 'appdetails' function".

**Impact:** Cannot use dynamic orgId discovery in Flow. Must use static constants.

**Solution:** Use hard-coded static values (see §1.1 canonical template):
```deluge
staticOrgId = "YOUR_ORG_ID";
```

### 7.1 searchRecords status Filter - ENVIRONMENT SPECIFIC (2026-03-05)

**Discovery (Empirical Smoke Tests):**

| Environment | `status=Open` | `statusType=Open` |
|-------------|---------------|-------------------|
| **Zoho Desk** | ✅ Works | ❌ Fails — see §7.10 |
| **Zoho Flow** | ✅ Works | ❌ FAILS (index error) |

**Smoke Test Results - Flow (05-Mar-2026):**
```
status=Open filter: 3 tickets (correctly filtered)
statusType=Open filter: ERROR - index invalid
No filter: 10 tickets
```

**Smoke Test Results - Desk (26-Feb-2026 — LATER CONTRADICTED):**
```
searchRecords + statusType=Open: 2 tickets (early result — see §7.10 for corrected findings)
```

**Working Code Snippet (Flow):**
```deluge
// Flow: Use status=Open
critOpen = Map();
critOpen.put("status", "Open");
searchPage1 = zoho.desk.searchRecords(orgId, "tickets", critOpen, 0, 100);
```

**Working Code Snippet (Desk):**
```deluge
// Desk: Always use status=Open — statusType is unreliable
critOpen = Map();
critOpen.put("status", "Open");
searchPage1 = zoho.desk.searchRecords(orgId, "tickets", critOpen, 0, 100);
```

**Rule:** Always use `status=Open` for maximum compatibility across both environments.

### 7.2 Comparison Operators Caveat

**Discovery:** Deluge does not support direct comparison of certain types (e.g., comparing ticket IDs, timestamps) with `!=` or `<` operators. This causes "Comparison of non numeric expression is not supported" errors.

**Workaround:** Convert to strings or numbers before comparing:
```deluge
// PROBLEM: Comparing IDs directly
if(idU2 != oldestId)  // May fail

// SOLUTION: Convert to strings first
if(idU2.toString() != oldestId.toString())

// For timestamps, convert to Long
tsNum = tsU.toLong();
oldestNum = oldestTs.toLong();
if(tsNum < oldestNum)
```

### 7.3 Boolean AND (`&&`) in If Statements

**Discovery:** Chained `&&` in if conditions can cause parsing errors in Desk Deluge.

**Workaround:** Split into nested if statements:
```deluge
// PROBLEM:
if(ps == "ONLINE" && aId != null)

// SOLUTION:
if(ps == "ONLINE")
{
    if(aId != null)
    {
        // do something
    }
}
```

### 7.4 Empty Catch Blocks

**Discovery:** Empty catch blocks cause syntax errors in Desk Deluge.

**Solution:** Add a dummy assignment:
```deluge
catch (ex)
{
    dummy = "caught";
}
```

### 7.5 searchRecords Response Format (Empirically Verified 2026-03-02)

**Discovery:** `searchRecords` can return a response that is BOTH a List AND a Map. The response has numeric keys (like a List) AND contains a "data" key.

**Test Result:** `isList=true isMap=true`

**Solution - Check both:**
```deluge
pageTickets = List();
firstItem = null;
try { firstItem = searchResult.get(0); }
catch (e) { firstItem = null; }

if(firstItem != null) {
    // Has numeric keys - use as List
    pageTickets = searchResult;
}
// ALSO check for data key
if(searchResult.containsKey("data")) {
    dataList = ifnull(searchResult.get("data"), List());
    if(dataList.size() > 0) {
        pageTickets = dataList;
    }
}

### 7.6 searchRecords Limit Parameter

**Important:** The `limit` parameter in `searchRecords` has a maximum of 100, not 200 or 300 as sometimes assumed. Values >100 return `UNPROCESSABLE_ENTITY` error.

### 7.7 Desk Parser: try-catch Block Format (Empirically Verified 2026-03-02)

**Discovery (UPDATED):** Multi-line try-catch blocks WORK in Desk Deluge. Both formats tested and verified:

**Format A - Multi-line (WORKS):**
```deluge
try 
{
    // code
}
catch (e)
{
    // handler
}
```

**Format B - Inline (also WORKS):**
```deluge
try { code; }
catch (e) { handler; }
```

**Format C - try on same line as { (WORKS):**
```deluge
try {
    code;
}
catch (e) {
    handler;
}
```

**Conclusion:** All try-catch formats work. Use whichever style you prefer.

### 7.8 Blank Lines Before try Block (Flow-safe, Desk-unsafe)

**Flow:** Blank lines before try blocks are safe.

**Desk:** A blank line between variable declarations and the `try` keyword can cause "Improper Statement" errors. See §7.9.1 for the Desk-specific rule.

### 7.9 Desk Parser: Misleading Error Line Numbers

**Discovery:** Error messages in Desk Deluge often report incorrect line numbers. The actual error may be 10+ lines before or after the reported line.

**Debugging strategy:**
1. Start with a minimal working function
2. Add code incrementally, testing after each addition
3. When error occurs, isolate the newly added block
4. Check for extra/missing braces in recent edits

### 7.9.1 Blank Lines Before try Block (Desk-specific)

**Discovery:** A blank line between variable declarations and the `try` keyword can cause "Improper Statement" errors in Desk Deluge.

**Fix:** Ensure no blank lines between last variable declaration and `try`:
```deluge
assigned = 0;
dummy = "init";
try {
    // code
}
```

### 7.10 searchRecords: status vs statusType Filter (2026-02-27)

**Test Results:**
| Filter Criteria | Tickets Returned |
|-----------------|------------------|
| `statusType=Open` | 0 (does NOT work) |
| `status=Open` | 5 (WORKS) |
| `isOpen=true` | 0 (does NOT work) |

**Solution:** Use `status=Open` (NOT `statusType=Open`) in searchRecords criteria:
```deluge
crit = Map();
crit.put("status", "Open");  // WORKS
// NOT: crit.put("statusType", "Open");  // DOES NOT WORK
```

**Note:** Both `status` and `statusType` fields exist on ticket objects and have the same values, but only `status` works as a search criteria.

### 7.11 Flow: READ → UPDATE → READ Pattern (2026-02-27)

**Pattern:** When updating state that affects counts, always READ → UPDATE → READ to get accurate counts.

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

### 7.12 Flow: String Comparison with .equals()

**Important:** In Flow, use `.equals()` for string comparison, not `==`:
```deluge
// WRONG - may not work in Flow
if (presenceStatus == "ONLINE")

// CORRECT - use .equals()
if (presenceStatus.equals("ONLINE"))

// With null safety
if (presenceStatus != null && presenceStatus.equals("ONLINE"))
```

### 7.13 Flow: Encryption URL Encode for Analytics v2

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

### 7.14 Flow: Null-Safe Input Parameters

**Always use ifnull()** for Flow inputs that may be empty:
```deluge
orgId = ifnull(flowOrgId, "");
presenceStatus = ifnull(flowPresenceStatus, "").toString().toUpperCase();
```

### 7.15 Identifying Bot-Failed Transfer Tickets by Subject (2026-03-19)

**Context:** When a chat ends in SalesIQ, a ticket is created in Zoho Desk. The ticket status and subject indicate what happened:

| Chat Outcome | Ticket Status | Subject Pattern |
|--------------|---------------|----------------|
| Chat attended by person | Closed | Email address only |
| Bot resolved (satisfied) | Closed | Email address only |
| **Bot failed to transfer** | **Open** | Contains "Transfer issue" or "Bot escalation" |
| Chat timed out | Closed | Email address only |

**How to identify bot-failed transfers:**
```deluge
// searchRecords with subject filter for bot transfers
crit = Map();
crit.put("status", "Open");
crit.put("departmentId", "123456789001"); // Primary Department dept

tickets = zoho.desk.searchRecords(orgId, "tickets", crit, 0, 100);

// Filter for bot transfer failures by subject
for each ticket in tickets {
    subject = ticket.get("subject");
    if (subject.contains("Transfer issue") || 
        subject.contains("Bot escalation")) {
        // This is a bot-failed transfer - needs attention
        // DEBUG: Bot transfer failed: " + subject
    }
}
```

**Alternative: Filter by subject using ZQL criteria:**
```deluge
// Direct criteria string (escaping quotes)
critStr = "(status:equals:Open) AND (departmentId:equals:123456789001) AND (subject:contains:Bot escalation)";

// Or use subject.contains filter
critStr = "status:equals:Open AND subject:contains:Transfer issue";
```

**Why this matters:**
- Bot-failed transfers should remain **Open** (not auto-closed)
- These require human follow-up to contact the end-user/customer
- All other chat tickets should be **Closed**

### 7.15.1 Auto-Close Chat Tickets Workflow Rule

**Location:** Zoho Desk → Automation → Workflow → Rules

| Property | Value |
|----------|-------|
| **Trigger** | On Update of Ticket Owner |
| **Criteria** | Channel is Chat AND Ticket Owner is not empty AND Status is Open AND Subject does not contain "Bot escalation" |
| **Action** | Mark Ticket as Closed |

**Updated 2026-03-19 11:01:** Added subject exclusion to prevent closing bot-failed transfer tickets.

**Example department IDs:**
| Department | ID |
|------------|-----|
| Primary Department (correct) | `123456789001` |
| SalesIQ/Support (legacy/wrong) | `123456789002` |

### 7.16 Debug Logging via Analytics (Recommended Architecture)

**Problem:** Dry-run debug works but production trigger issues invisible.

**Solution:** Log to an Analytics table to inspect execution after the fact. This is a general pattern — adapt the table schema to your own metrics.

**Example table structure:**
```json
{
  "table_name": "ticket_router_debug_log",
  "columns": [
    {"name": "Run ID", "type": "TEXT"},
    {"name": "Timestamp", "type": "DATETIME"},
    {"name": "Function", "type": "TEXT"},
    {"name": "Version", "type": "TEXT"},
    {"name": "Step", "type": "TEXT"},
    {"name": "Level", "type": "TEXT"},
    {"name": "Message", "type": "TEXT"},
    {"name": "Dept Count", "type": "NUMBER"},
    {"name": "Ticket Count", "type": "NUMBER"},
    {"name": "Assigned Count", "type": "NUMBER"},
    {"name": "Online Agents", "type": "NUMBER"},
    {"name": "Capacity Agents", "type": "NUMBER"},
    {"name": "Duration_ms", "type": "NUMBER"}
  ]
}
```

**Implementation pattern:**
```deluge
DEBUG = false;  // Toggle for production

if(DEBUG) {
    rowData = Map();
    rowData.put("Step", "STEP_NAME");
    rowData.put("Message", "Details...");
    rowData.put("Ticket Count", ticketCount);
    zoho.reports.createRow(workspace, table, rowData, "connection");
}
```

**Key learnings:**
1. Always gate debug writes with `if(DEBUG)` to control API costs
2. Log before/after critical operations to isolate failures
3. Include metrics at each step for performance monitoring
4. Use unique Run ID to group a single execution

### 7.17 Agent Capacity Logic (Recommended Pattern)

This is a generalized capacity-checking pattern for round-robin or load-balanced ticket assignment. Adjust `CAPACITY_THRESHOLD` and the load source to your workflow.

**Logic:** Agent has capacity if `assignedTickets < capacityThreshold`

**Code:**
```deluge
CAPACITY_THRESHOLD = 3;  // Agents can have up to 3 tickets

for each agent in onlineAgents {
    valLoad = agentLoad.get(agentId);
    if(valLoad == null) { valLoad = 0; }
    if(valLoad < CAPACITY_THRESHOLD) {
        capacityAgents.add(agentId);
    }
}
```

**Important:** Only counts tickets matching the department filter. Agent may have tickets in other departments - these don't count toward capacity.

### 7.18 Hook Best Practices for Ticket Assignment

**Issue:** Trigger hook firing incorrectly or not at all.

**Debug approach:**
1. Enable DEBUG flag in function
2. Check Analytics debug table for execution logs
3. Verify hook is firing (check for START step)
4. Trace through each step to find failure point

**Hook recommendations (general):
- Use `channel is not Phone` to capture Chat + Email only
- Avoid complex criteria during initial testing
- Add criteria incrementally after basic functionality works

---

## 8. External Reference: The Workflow Academy (GitHub)

**URL:** https://github.com/TheWorkflowAcademy

The Workflow Academy is a Zoho developer/consultant community with 80+ public repos of Deluge scripts covering CRM, Desk, Flow, Analytics, Books, Creator, and more.


The repos (Books, Creator, Inventory, Campaigns, WorkDrive, etc.) are available directly at https://github.com/TheWorkflowAcademy — browse as needed.

---

## 9. AI Agent Execution Guardrails

This section captures rules specifically for AI agents generating or modifying Deluge code. These guardrails override generic Zoho documentation where they conflict. **§7 (Lessons Learned)** and this section represent live, empirically verified discoveries (Feb–Jun 2026) and take precedence over §§1-6 where they conflict.

### 9.1 Absolute Resolution of Document Contradictions

The following contradictions from earlier sections are hereby resolved. Agents must treat these as final rules:

| Resolved Issue | Final Rule | Source |
|----------------|-----------|--------|
| **Desk whitespace before `try`** (§7.8 vs §7.9) | NEVER leave a blank line between variable initializations and the opening `try` in Desk Deluge. This causes "Improper Statement" parser errors. | §7.9.1 |
| **`statusType` as search filter** (§3.3 vs §7.1 vs §7.10) | Completely disregard `statusType` and `isOpen`. Always use `status` (e.g., `crit.put("status", "Open")`). Does NOT work in any environment. | §7.10 |

### 9.2 Complete Snippet: Sorting a List of Maps (sortBy Workaround)

Since Deluge lacks `sortBy()`, use extract-sort-rebuild. See §2.1 for the canonical snippet. If sorting by a numeric key (e.g., agent load), the `.toLong()` variant also works:

```deluge
// Input: agentList (List of Maps, each containing "agentId" and "load")
sortedAgents = List();
if(agentList.size() > 0)
{
    // Step 1: Extract and sort weights
    weights = List();
    for each agent in agentList
    {
        weights.add(agent.get("load").toLong());
    }
    weights.sort(true); // true = ascending

    // Step 2: Rebuild sorted list matching elements to weights
    for each w in weights
    {
        for each agent in agentList
        {
            if(agent.get("load").toLong() == w)
            {
                if(!sortedAgents.contains(agent))
                {
                    sortedAgents.add(agent);
                }
            }
        }
    }
}
```

### 9.3 ZQL Workspace Selection Hierarchy

When generating ZQL date/time functions, select based on the target workspace profile:

| Need | Fallback Dialect (e.g. `YOUR_WORKSPACE_NAME`) | Standard Dialect |
|------|-------------------------------------------|------------------|
| Current timestamp | `LOCALTIMESTAMP()` | `current_timestamp()` |
| Day-level offset | `ADDDATE(LOCALTIMESTAMP(), -N)` | `dateadd('day', -N, current_timestamp())` |
| Hour extraction | `EXTRACT(HOUR FROM column)` | `HOUR(column)` |

**Rules:**
- Prefer **Fallback Dialect** for all `YOUR_WORKSPACE_NAME` workspace queries — `dateadd()` fails regardless of quoting in that workspace.
- `LIMIT` is unsupported everywhere — use `FETCH FIRST N ROWS ONLY`.
- Column names in imported tables are **display names**, not API keys — verify via ZQL Explorer.

### 9.4 Defensive Type Verification (JSON/Map Auto-Casting)

Deluge implicitly casts payloads: single-element JSON arrays become Maps, Maps with numeric keys become Lists. Always normalize ambiguous API returns to a verified List:

```deluge
// Defensive normalization of search/API returns to a verified List
finalRecords = List();
if(rawResponse != null)
{
    if(rawResponse.containsKey("data"))
    {
        // Wrapped in a data envelope
        dataVal = rawResponse.get("data");
        try { dummy = dataVal.get(0); finalRecords = dataVal; }
        catch (e) { finalRecords = List(); finalRecords.add(dataVal); }
    }
    else
    {
        // Probe for List vs Map
        isList = false;
        try { dummy = rawResponse.get(0); isList = true; }
        catch (e) { isList = false; }

        if(isList) { finalRecords = rawResponse; }
        else { finalRecords = List(); finalRecords.add(rawResponse); }
    }
}
```

See also §1.6 for the complete try/catch probing pattern.

### 9.5 Environment Constraints (Quick Reference)

| Constraint | Flow | Desk |
|------------|------|------|
| Return type | Map | VOID |
| Analytics API | v2 REST (via `invokeurl`) | v1 Reports only |
| `else` | ❌ Forbidden | ✅ Works |
| `continue` | ❌ Forbidden | ✅ Works |
| `instanceof` | ❌ Forbidden | ✅ Works |
| `info` statements | ❌ Forbidden | ✅ Allowed |
| `while` loops | ✅ Works | ❌ Forbidden |
| `range()` | ✅ Works | ❌ Forbidden |
| Helper functions | ❌ Forbidden | ✅ Allowed |
| Signature rule | Line 1, `{` on same line | Standard |
| `zoho.appdetails()` | ❌ Fails | ❌ Fails |

### 9.6 Common AI Hallucination Risks

| Hallucination | Reality | Ref |
|---------------|---------|-----|
| `sortBy()` exists in Deluge | Does not exist — use extract-sort-rebuild | §2.1 |
| `statusType` works as search filter | Fails in all environments — use `status` | §7.10 |
| `dateadd()` is portable | Fails in `YOUR_WORKSPACE_NAME` | §4.4.5 |
| Blank lines before `try` are safe in Desk | Causes "Improper Statement" in Desk | §7.9.1 |
| `invokeurl` returns error map on 4xx/5xx | Throws script error — must wrap in try/catch | §1.2.1 |
| API response types are stable | Deluge auto-casts — always probe type dynamically | §1.6 |

### 9.7 Advanced Logic & Data Mutation Guardrails

#### 9.7.1 The Map.toString() JSON Payload Trap
- **Rule**: Never use `myMap.toString()` to generate a JSON payload for a standard third-party REST API. Deluge serializes maps using equal signs (e.g., `{key=value}`), which will cause HTTP 400 Bad Request errors.
- **Resolution**: For third-party APIs, use an explicitly escaped raw string body (§1.2.1). Only use `myMap.toString()` when passing a `CONFIG` string to Zoho Analytics v2, as Zoho's internal systems natively understand the Deluge map string format.

#### 9.7.2 Analytics v2 REST Batch Constraints (Flow Only)
- **Rule**: When using the preferred `PUT` method on `/rows` with `addIfNotExist: true` for bulk state updates, the payload MUST be capped at a maximum of **100 rows per transaction**.
- **Resolution**: If an agent processes an incoming list larger than 100 items, it must chunk the payload or process it via sequential execution, otherwise Analytics will reject the request with a payload size exception.

#### 9.7.3 Safe Data Cleansing (Avoiding getAlphaNumeric())
- **Rule**: Never use `.getAlphaNumeric()` to clean phone numbers, email addresses, or domain names. It removes structural characters like `+`, `-`, `@`, and `.`.
- **Resolution**: Use explicit regex or character replacement instead:
```deluge
// Correct pattern for phone numbers
cleanPhone = rawPhone.toString().replaceAll(" ", "").replaceAll("-", "");
```

#### 9.7.4 Transaction Scope and Loop Re-initialization (Flow-Specific)
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

---

# End of Zoho Compliance Guide
