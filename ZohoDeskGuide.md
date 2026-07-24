# Zoho Desk Guide — The Ticket System

**Scope:** Zoho Desk — the ticket system.

> This file is environment-specific. Load `ZohoGuideRoot.md` first for pervasive rules (canonical template, API patterns, ownership model, pre-deployment checklist).

**Last Updated:** 2026-07-24

---

## §D.1 — Capabilities & Constraints

- **VOID Return**: Desk custom functions are `VOID`. Do not rely on returned values. Persist state via Analytics or Desk API.
- **Analytics Writes**: `zoho.reports.createRow()`/`updateData()`/`deleteRow()` are **silent no-ops** in Desk — return success without writing data. Use `invokeUrl` + PUT /rows + CONFIG instead (→ See ZohoDeskGuide §D.4).
- **Analytics Reads**: `zoho.reports.getData()` does NOT exist in Desk. For Analytics reads from Desk, use a Zoho Flow intermediary.
- **No Function Declarations**: `myFunc = function(args) { ... }` is **rejected** by the Desk parser with `"Expecting ';' at the end of statement"`. All helper logic must be inlined at every call site. See §D-A.9.2 for details.
- Always use **Connections** rather than hard-coded tokens; keep environment (sandbox vs production) configurable.
- **`zoho.encryption.urlEncode()`**: Not available in Desk. Use manual `replaceAll` for URL-encoding CONFIG strings (see §D.4).
- **`zoho.appdetails()`**: Does NOT work in Desk. Use static constants (→ See ZohoGuideRoot §1.1 canonical template).
- **`while` loops**: Desk Deluge does not support `while` loops or `range()` function. Use explicit sequential fetches for pagination.
- **`info` statements**: Allowed and viewable in execution logs in Desk.
- **`else`/`continue`/`instanceof`**: All supported in Desk (unlike Flow).

### Environment Constraints Quick Reference (Desk Column)

| Constraint | Desk |
|------------|------|
| Return type | VOID |
| Event object | None (hook → function params) |
| Analytics API | v2 REST via `invokeUrl` + PUT /rows + CONFIG (`zoho.reports.*` are silent no-ops — see §D.4) |
| `else` | ✅ Works |
| `continue` | ✅ Works |
| `instanceof` | ✅ Works |
| `info` statements | ✅ Allowed |
| `while` loops | ❌ Forbidden |
| `range()` | ❌ Forbidden |
| Helper functions (inline) | ✅ Allowed (but `= function(...)` variable declarations rejected — see §D-A.9.2) |
| `zoho.appdetails()` | ❌ Fails |
| `now()` | ✅ Works |
| `keySet()` | ✅ Works |
| `getType()` | ✅ Works |
| `containsKey()` | ✅ Works |
| `invokeUrl` | ✅ Works |
| `zoho.reports.getData` | ❌ **Does NOT exist** — Desk cannot read Analytics tables. Use Flow intermediary. |
| `zoho.encryption.urlEncode` | ❌ Not available |
| `for each` loops | ✅ Works |
| External non-Zoho APIs | ✅ Works |
| Connections | OAuth + STRING variable |

---

## §D.2 — Desk Patterns & Snippets

- Use static constants: `staticOrgId = "YOUR_ORG_ID"` at top of function.
- For Analytics writes from Desk, use **`invokeUrl` + PUT /rows + CONFIG** (see §D.4 for code example). `zoho.reports.*` functions are silent no-ops in Desk.
- For Analytics reads from Desk, use a **Zoho Flow intermediary** (`zoho.reports.getData()` does not exist in Desk).
- Log errors compactly and avoid verbose debug logs in production.

### Desk → Analytics Write Pattern (PUT /rows + CONFIG)

```deluge
cfg = Map();
cols = Map();
cols.put("column1", value1);
cfg.put("columns", cols);
cfg.put("addIfNotExist", true);
cfg.put("criteria", "1=2");
cfgStr = cfg.toString();
encStr = cfgStr.replaceAll("\"", "%22").replaceAll(":", "%3A").replaceAll(",", "%2C").replaceAll(" ", "%20").replaceAll("'", "%27");
putUrl = analyticsDomain + "/restapi/v2/workspaces/" + wsId + "/views/" + viewId + "/rows?CONFIG=" + encStr;
resp = invokeurl
[
    url: putUrl
    type: PUT
    headers: {"ZANALYTICS-ORGID": orgId, "Content-Type": "application/json"}
    connection: "analyticsconnection"
];
```

**Note:** Desk does NOT have `zoho.encryption.urlEncode()`. Use manual `replaceAll` encoding as shown above. The characters `\"`, `:`, `,`, ` `, and `'` are the safe set to encode. Leave `{` and `}` raw — Zoho APIs accept them.

### Desk-Safe Function Skeleton (VOID Return)

```deluge
// Static constants
staticOrgId = "YOUR_ORG_ID";
try
{
	// ... do work, persist state via invokeUrl + REST APIs
}
catch (e)
{
	// log only; Desk will not consume returned maps
	logError("DeskModule", "FunctionName", e.toString());
}
```

### Desk API Call Pattern (invokeurl)

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

**Desk-specific body behavior:** `body` as Map may fail in Desk; prefer pre-serialized JSON string when Map body causes errors:

```deluge
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

### Debug Comment Pattern (Desk)

Since Desk allows `info` but it should be minimal, use comment breadcrumbs for stable tracing:

```deluge
// DEBUG: computing open counts by department
// DEBUG: invokeUrl PUT /rows response received
// DEBUG: write to Analytics via CONFIG completed
```

---

## §D.3 — Desk API Gotchas & Known Issues

**Two API layers — know which one you're using:**

| Layer | How to Call | Parameter Rules |
|-------|-------------|-----------------|
| **Native REST** | `invokeurl` to `https://desk.zoho.com/api/v1/...` | Full HTTP — query params, headers, pagination via URL params |
| **Deluge Integration Task** | `zoho.desk.searchRecords()`, `zoho.desk.getTicketById()`, etc. | Wrapper — uses internal Desk API but **parameter names and limits differ** from the raw REST layer |

Key differences:
- `searchRecords` uses a **criteria Map**, not URL query strings
- `searchRecords` limit max is **100**, not 200 as the REST API accepts. Raw REST `invokeurl` calls can use `limit=200` per page — the 100 cap applies only to the `zoho.desk.*` wrapper.
- `searchRecords` does NOT support `sortField`/`sortOrder` — the raw REST endpoint also rejects these params with `UNPROCESSABLE_ENTITY`
- `searchRecords` does NOT support `statusType` — use `status` instead (see §D-A.10)

- **tickets endpoint**: `GET /api/v1/tickets` does NOT support `sortField` or `sortOrder` query parameters — will return `UNPROCESSABLE_ENTITY` error.
- **status in searchRecords**: The `status` parameter WORKS in searchRecords. Use: `crit.put("status", "Open")`. Do NOT use `statusType` — it is unreliable across all environments (see §D-A.10).
- **tickets endpoint**: Avoid `isStatusChanged` parameter — not valid for this endpoint.
- **Response handling**: API may return tickets in `data` key, or `tickets` key — always check both paths with `ifnull()`.
- **searchRecords returns List or Map**: `zoho.desk.searchRecords()` can return either a List directly OR a Map with a "data" key. Always handle both formats (see §D-A.5).
- **statusType field**: The ticket object contains `statusType` field (e.g., "Open", "Closed") but it does NOT work as a search filter in any environment. Always use `status=Open` instead — see §D-A.10.
- **searchRecords filter param**: Do NOT pass `null` or `{}` for empty criteria. Use `crit = Map()` (assign to variable first), then pass the variable.
- **No while loops**: Desk Deluge does not support `while` loops or `range()` function. Use explicit sequential fetches for pagination.
- **zoho.appdetails()**: Does NOT work in Flow or Desk. Use static constants (orgId, analytics_org_id, etc.) defined at top of function.
- **Pagination**: Use explicit sequential fetches (e.g., `searchRecords(orgId,"tickets",crit,0,100)`, then `searchRecords(orgId,"tickets",crit,100,100)`). No loops allowed.
- **searchRecords limit max**: The `limit` parameter **max value is 100**, not 200. Values >100 return `UNPROCESSABLE_ENTITY` error: "The value passed for field 'limit' exceeds the range of '1-100'."
- **Channel field case sensitivity**: The ticket `channel` field returns `"Email"` (capital E), NOT `"EMAIL"`. Use case-insensitive comparison: `channelVal.toString().toUpperCase().equals("EMAIL")`.
- **Channel values (Analytics import, Jul 2026):** Beyond Email/Chat/Phone, SalesIQ integration creates per-platform tags: `FB Messenger`, `IG Messenger`, `Instagram`, `Facebook`. The Analytics import preserves these verbatim. When filtering in ZQL or dashboard widgets, expect the full set.
- **CRITICAL — SalesIQ channel scope:** All workflow rules, Flow functions, and Deluge scripts that filter on `Channel = 'Chat'` will **miss** `FB Messenger`, `IG Messenger`, `Instagram`, and `Facebook` tickets. These are all SalesIQ channels and must be included explicitly in any rule that targets chat conversations. Full channel tag list: `Chat`, `FB Messenger`, `IG Messenger`, `Instagram`, `Facebook`, plus `Web` (legacy/Zendesk import, inactive since May 2026).
- **Channel filter in searchRecords**: The `channel` parameter WORKS in Desk searchRecords. Tested values: `"Email"`, `"Chat"`, `"Phone"`, `"FB Messenger"`, `"IG Messenger"`, `"Instagram"`, `"Facebook"`. For multi-channel matching, use OR criteria or loop through a channel list.
- **SalesIQ Conversations table:** Imported to Analytics as `Conversations`. Links to tickets via `Zoho Desk Ticket ID`. Key columns: `Initiated Time`, `End Time`, `Chat Duration in Minutes`, `Status` (Attended/Closed/Bot Transfer Missed), `Ended By`, `Missed Reason`. Useful for cross-referencing chat outcomes against ticket data.
- **Empty criteria Map required**: Must use `critEmpty = Map()` then pass variable. Inline `{}` causes "COLLECTION" type error.

---

## §D.4 — Analytics Write Path from Desk

### Why `zoho.reports.*` Fails in Desk

**Final finding (2026-07-14):** `zoho.reports.createRow()` in Desk is a **silent no-op** — it returns success but does NOT write data to Analytics. The same applies to `updateData()` and `deleteRow()`.

| Function | Desk (`analyticsconnection`) |
|------|------|
| `zoho.reports.createRow()` | ⚠️ **Silent no-op** — returns success, no data written |
| `zoho.reports.updateData()` | ⚠️ Likely same silent no-op behavior |
| `zoho.reports.deleteRow()` | ⬜ Likely same silent no-op behavior |
| `zoho.reports.getData()` | ❌ Does NOT exist in Desk Deluge |
| `invokeUrl` + GET | ✅ Works |
| `invokeUrl` + PUT /rows + CONFIG | ✅ **Correct write path** |
| `invokeUrl` + POST /import | ❌ Response unparseable |
| `invokeUrl` + POST/PUT with Map body | ❌ `"index values were invalid"` crash |

→ See ZohoGuideRoot §1 for Workspace Ownership rules — error 7301 can silently block writes.

### The Correct Pattern: `invokeUrl` + PUT /rows + CONFIG

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

### URL Encoding in Desk

**`zoho.encryption.urlEncode()` is NOT available in Desk.** Use manual `replaceAll` encoding:

```deluge
cfgStr = cfg.toString();
encStr = cfgStr.replaceAll("\"", "%22").replaceAll(":", "%3A").replaceAll(",", "%2C").replaceAll(" ", "%20").replaceAll("'", "%27");
```

Safe characters to encode: `\"`, `:`, `,`, ` `, `'`. Leave `{` and `}` raw — Zoho APIs accept them.

→ See ZohoSalesIQGuide §S.6 for the full environment comparison table (`zoho.encryption.urlEncode` availability).

### `zoho.reports.*` Silent No-Op Warning

> **⚠ CRITICAL:** `zoho.reports.createRow()`, `updateData()`, and `deleteRow()` are **silent no-ops** in Desk — they return a success response but do NOT write data. This was empirically confirmed in TicketRouter v18→v19 (Jul 14, 2026). Always use `invokeUrl` + PUT /rows + CONFIG for Desk → Analytics writes.

### Desk Write Behavior vs Flow

Desk can write to Analytics even with Administrator or shared-user roles. The Workspace Owner-only restriction that blocks Flow (→ See ZohoGuideRoot §1.2) does NOT apply to Desk. However, the connection must still exist in Desk's connection pool (→ See ZohoGuideRoot §2.11).

---

## §D.5 — Desk Function Trigger Patterns

When building Desk custom functions triggered by workflow rules, the parameter mapping differs from CRM/Creator:

| Platform | Parameter UI | How Data Arrives |
|----------|-------------|-------------------|
| CRM / Creator | Modal popup with explicit data types (Map, List, String) | Parameters declared in the modal are available as typed variables |
| **Desk** | Inline "Edit Arguments" — field-by-field mapping | Each mapped field arrives as a separate parameter. **No Map type** — you must map individual fields or pass the ticket ID and fetch via API |

**Best practice for Desk:** Pass only the Ticket ID as a string parameter, then use `zoho.desk.getRecordById()` to fetch the full ticket map inside the function. This avoids the field-by-field mapping burden and ensures access to ALL ticket fields.

```deluge
// Function parameter: id (string) — mapped from ${ticket.ID} in workflow rule
ticket = zoho.desk.getRecordById(orgId, "tickets", id.toString(), "connection");
// Now access any field: ticket.get("Subject"), ticket.get("Ticket Owner"), etc.
```

---

## §D-A — Desk Lessons Learned

This section documents real-world discoveries from Desk development sessions that override or clarify earlier guidance. All cross-references have been updated to point within this file.

---

### §D-A.1 — searchRecords Status Filter (Environment-Specific, Desk Results)

**Discovery (Empirical Smoke Tests):**

| Environment | `status=Open` | `statusType=Open` |
|-------------|---------------|-------------------|
| **Zoho Desk** | ✅ Works | ❌ Fails — see §D-A.10 |
| **Zoho Flow** | ✅ Works | ❌ FAILS (index error) |

**Smoke Test Results - Desk (26-Feb-2026 — LATER CONTRADICTED):**
```
searchRecords + statusType=Open: 2 tickets (early result — see §D-A.10 for corrected findings)
```

**Working Code Snippet (Desk):**
```deluge
// Desk: Always use status=Open — statusType is unreliable
critOpen = Map();
critOpen.put("status", "Open");
searchPage1 = zoho.desk.searchRecords(orgId, "tickets", critOpen, 0, 100);
```

**Rule:** Always use `status=Open` for maximum compatibility across both environments.

---

### §D-A.2 — Comparison Operators Caveat

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

---

### §D-A.3 — Boolean AND (`&&`) in If Statements

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

---

### §D-A.4 — Empty Catch Blocks

**Discovery:** Empty catch blocks cause syntax errors in Desk Deluge.

**Solution:** Add a dummy assignment:
```deluge
catch (ex)
{
    dummy = "caught";
}
```

---

### §D-A.5 — searchRecords Response Format

**Discovery (Empirically Verified 2026-03-02):** `searchRecords` can return a response that is BOTH a List AND a Map. The response has numeric keys (like a List) AND contains a "data" key.

**Test Result:** `isList=true isMap=true`

**Solution — Check both:**
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
```

**See also:** → See ZohoGuideRoot §3.4 for the defensive type verification pattern (JSON/Map auto-casting).

---

### §D-A.6 — searchRecords Limit Parameter

**Important:** The `limit` parameter in `searchRecords` has a maximum of 100, not 200 or 300 as sometimes assumed. Values >100 return `UNPROCESSABLE_ENTITY` error: "The value passed for field 'limit' exceeds the range of '1-100'."

---

### §D-A.7 — Desk Parser: try-catch Block Format

**Discovery (Empirically Verified 2026-03-02):** Multi-line try-catch blocks WORK in Desk Deluge. All formats tested and verified:

**Format A — Multi-line (WORKS):**
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

**Format B — Inline (also WORKS):**
```deluge
try { code; }
catch (e) { handler; }
```

**Format C — try on same line as { (WORKS):**
```deluge
try {
    code;
}
catch (e) {
    handler;
}
```

**Conclusion:** All try-catch formats work. Use whichever style you prefer.

---

### §D-A.8 — Desk Parser: Misleading Error Line Numbers

**Discovery:** Error messages in Desk Deluge often report incorrect line numbers. The actual error may be 10+ lines before or after the reported line.

**Debugging strategy:**
1. Start with a minimal working function
2. Add code incrementally, testing after each addition
3. When error occurs, isolate the newly added block
4. Check for extra/missing braces in recent edits

---

### §D-A.9.1 — Blank Lines Before try Block (Desk-Specific Rule)

**Discovery:** A blank line between variable declarations and the `try` keyword can cause "Improper Statement" errors in Desk Deluge. (Flow is safe — blank lines before try blocks are fine in Flow.)

**Fix:** Ensure no blank lines between last variable declaration and `try`:
```deluge
assigned = 0;
dummy = "init";
try {
    // code
}
```

---

### §D-A.9.2 — Custom Function Declarations Rejected (2026-07-14)

**Discovery:** Desk Deluge rejects `function(...)` variable declarations — the parser throws `"Expecting ';' at the end of statement"` on the line with `= function(...)`. This applies to both `void logDebug(...)` style and `myFunc = function(args) { ... };` style.

**Tested in:** TicketRouter v18 → v19 (empirically verified, Jul 14 2026)

**Impact:** Any code using helper functions declared as variables will fail to deploy:
```deluge
// ❌ REJECTED by Desk parser — line with = function(...) causes parse error
dbgWrite = function(step, level, msg)
{
    // ... code ...
};

// ❌ Also rejected
void logError(string msg) { ... }
```

**Solution:** Inline all helper logic at every call site. Wrap each inlined block in its own `try/catch` to isolate failures:
```deluge
// ✅ INLINED — works in Desk
if(DEBUG)
{
    try
    {
        c = Map();
        c.put("Step", "START");
        // ... all debug write logic inlined ...
    }
    catch (e_dbg) { dummy = "caught"; }
}
```

**Why this is necessary:** Desk Deluge does not support first-class functions or closures. Unlike Flow (which at least allows inline functions in some contexts), Desk's parser treats `= function(...)` as a statement expecting a semicolon. The only workaround is to duplicate the logic at each call site.

**Trade-off:** Code becomes longer but is functionally correct. When DEBUG=false, the `if(DEBUG)` guard ensures zero overhead — the inlined blocks never execute.

**Precedent:** This constraint was previously discovered for SalesIQ (SalesIQ_Ticket_Probe_v4, Jul 2026) and Flow (→ See ZohoGuideRoot §2.1). The Jul 14 TicketRouter v18→v19 fix confirms it also applies to Desk.

---

### §D-A.10 — searchRecords: status vs statusType Filter

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

---

### §D-A.15 — Identifying Bot-Failed Transfer Tickets by Subject

**Context:** When a chat ends in SalesIQ, a ticket is created in Zoho Desk. The ticket status and subject indicate what happened.

#### 3-Type Bot Ticket Classification (Empirically Verified Jul 2026)

| Type | Subject Contains | Owner? | Status | Meaning |
|------|-----------------|--------|--------|---------|
| 1 - Bot resolved | `Bot resolved chat query ...` | No | Closed | Bot handled the query successfully, closed chat, created closed ticket. No agent needed. |
| 2 - Human resolved | `Bot transferred chat query ...` | Yes | Closed | Bot transferred to agent, they handled it, workflow auto-closed. **Normal flow.** |
| 3 - Missed transfer | `Bot transferred chat query ...` | No | Closed | Bot tried to transfer, no agent available, bulk-closed ticket **without ever assigning** anyone. Customer query was never serviced. |

**Subject is the only differentiator** between Type 2 (assigned) and Type 3 (unassigned) since both say "Bot transferred". Type 3 requires owner presence check.

#### SalesIQ Channel Scope (CRITICAL)

SalesIQ integration creates tickets under **5 distinct channel names**, not just `Chat`:

| Channel | Source |
|---------|--------|
| `Chat` | Standard SalesIQ web chat |
| `FB Messenger` | Facebook Messenger |
| `IG Messenger` | Instagram Messenger |
| `Instagram` | Instagram comments |
| `Facebook` | Facebook page posts |

**Any workflow rule, Flow function, or Deluge script filtering on `Channel = 'Chat'` will silently miss the other 4 channels.** Always use an explicit list.

#### Original Subject Patterns (Historical, pre-Jul 2026)

| Chat Outcome | Ticket Status | Subject Pattern |
|--------------|---------------|----------------|
| Bot resolved | Closed | `Bot resolved chat query email (Name)` |
| Human resolved | Closed | `Bot transferred chat query email (Name)` |
| **Missed transfer** | **Closed** | `Bot transferred chat query email (Name)` (no owner) |
| Chat timed out | Closed | Email address only |

---

### §D-A.15.1 — Auto-Close Chat Tickets Workflow Rule

**Location:** Zoho Desk → Automation → Workflow → Rules

| Property | Value |
|----------|-------|
| **Trigger** | On Update of Ticket Owner |
| **Criteria** | Channel is Chat AND Ticket Owner is not empty AND Status is Open AND Subject does not contain "Bot transfer chat query" |
| **Action** | Mark Ticket as Closed |

**⚠ SalesIQ channel scope bug:** Rule only covers `Channel = Chat`. Tickets from `FB Messenger`, `IG Messenger`, `Instagram`, `Facebook` are never auto-closed. Fix: expand criteria to include all 5 SalesIQ channels.

**Updated 2026-03-19 11:01:** Added subject exclusion to prevent closing bot-failed transfer tickets.

**Example department IDs:**
| Department | ID |
|------------|-----|
| Primary Department (correct) | `123456789001` |
| SalesIQ/Support (legacy/wrong) | `123456789002` |

---

### §D-A.16 — Debug Logging via Analytics (Recommended Architecture)

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

**Implementation pattern (Desk — using invokeUrl + PUT /rows):**
```deluge
DEBUG = false;  // Toggle for production

if(DEBUG) {
    cfgDbg = Map();
    colsDbg = Map();
    colsDbg.put("Step", "STEP_NAME");
    colsDbg.put("Message", "Details...");
    colsDbg.put("Ticket Count", ticketCount);
    cfgDbg.put("columns", colsDbg);
    cfgDbg.put("addIfNotExist", true);
    cfgDbg.put("criteria", "1=2");
    cfgDbgStr = cfgDbg.toString();
    encDbg = cfgDbgStr.replaceAll("\"", "%22").replaceAll(":", "%3A").replaceAll(",", "%2C").replaceAll(" ", "%20");
    putUrlDbg = analyticsDomain + "/restapi/v2/workspaces/" + wsId + "/views/" + viewId + "/rows?CONFIG=" + encDbg;
    respDbg = invokeurl
    [
        url: putUrlDbg
        type: PUT
        headers: {"ZANALYTICS-ORGID": orgId, "Content-Type": "application/json"}
        connection: "analyticsconnection"
    ];
}
```

**Key learnings:**
1. Always gate debug writes with `if(DEBUG)` to control API costs
2. Log before/after critical operations to isolate failures
3. Include metrics at each step for performance monitoring
4. Use unique Run ID to group a single execution

---

### §D-A.17 — Agent Capacity Logic (Recommended Pattern)

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

---

### §D-A.19 — Hook Best Practices for Ticket Assignment

**Issue:** Trigger hook firing incorrectly or not at all.

**Debug approach:**
1. Enable DEBUG flag in function
2. Check Analytics debug table for execution logs
3. Verify hook is firing (check for START step)
4. Trace through each step to find failure point

**Hook recommendations (general):**
- Use `channel is not Phone` to capture Chat + Email only
- Avoid complex criteria during initial testing
- Add criteria incrementally after basic functionality works

---

### §D-A.20 — Workspace Ownership Transfer Breaks Analytics Writes

**Issue:** `invokeUrl` PUT /rows to Analytics returns `SECURITY_NOT_PERMITTED` (error 7301) despite connection having correct API scopes and showing "Connected" status.

**Root cause:** The connection owner is not the **Workspace Owner** in the target Analytics workspace. API scopes (`ZohoAnalytics.data.all`) control endpoint access; only the Workspace Owner can perform writes via API in Flow. Even Workspace Administrator role is not sufficient in Flow.

**Misleading error:** The API error says "Only Workspace Admin has the permission" but in practice, only the **Owner** role works for API writes in Flow.

**Flow vs Desk distinction:**
- **Flow:** Workspace Administrators are **read-only**. Only the Owner can write. This is an environment-specific permission boundary.
- **Desk:** Does NOT enforce the same rule. Desk can write to Analytics even with Administrator or shared-user roles. Desk was able to work around ownership issues by restructuring API calls.

**Why it happened:** Workspace ownership was transferred to a different Zoho account. The original developer lost ownership. The OAuth connection still works (API scopes unchanged), but the workspace permission layer rejects writes in Flow.

**Fix:** Zoho Analytics → Workspace → Settings → Manage Users → **transfer workspace ownership** back to the developer account. Adding as Administrator is NOT enough for Flow.

**Prevention:** The developer account should own ALL components: code, triggers, connections, AND workspaces. Never transfer workspace ownership without re-establishing Owner for the developer account.

**Full documentation:** → See ZohoGuideRoot §1 (Workspace Ownership & Permission Model).

---

## §D-G — Desk AI Guardrails

These are rules specifically for AI agents generating or modifying Desk Deluge code. They override generic Zoho documentation where they conflict.

### §D-G.1 — Environment Write Paths: Desk

**Final finding (2026-07-14):** `zoho.reports.createRow()` in Desk is a **silent no-op** — it returns success but does NOT write data to Analytics.

| Function | Desk (`analyticsconnection`) |
|------|------|
| `zoho.reports.createRow()` | ⚠️ **Silent no-op** — returns success, no data written |
| `zoho.reports.updateData()` | ⚠️ Likely same silent no-op behavior |
| `zoho.reports.deleteRow()` | ⬜ Likely same silent no-op behavior |
| `zoho.reports.getData()` | ❌ Does NOT exist in Desk Deluge |
| `invokeUrl` + GET | ✅ Works |
| `invokeUrl` + PUT /rows + CONFIG | ✅ **Correct write path** |

**Desk → Analytics writes — use `invokeUrl` + PUT /rows + CONFIG:**
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

**Key lesson:** `zoho.reports.*` functions silently no-op in Desk — they return success without writing. Always use `invokeUrl` + REST for Desk → Analytics writes. **Always use `zoho.encryption.urlEncode()`** for CONFIG parameters — it handles all special characters correctly. Do NOT use manual `replaceAll` encoding (fragile, misses characters) or `parameters: Map()` (unreliable for PUT/DELETE in Flow — CONFIG may not be passed correctly).

> **⚠ Desk Note:** `zoho.encryption.urlEncode()` is NOT available in Desk. For Desk, use manual `replaceAll` encoding as shown in §D.2 and §D.4.

### §D-G.2 — Common AI Hallucination Risks (Desk-Specific)

| Hallucination | Reality | Ref |
|---------------|---------|-----|
| `sortBy()` exists in Deluge | Does not exist — use extract-sort-rebuild | → See ZohoGuideRoot §2.1 |
| `statusType` works as search filter | Fails in all environments — use `status` | §D-A.10 |
| Blank lines before `try` are safe in Desk | Causes "Improper Statement" in Desk | §D-A.9.1 |
| `invokeurl` returns error map on 4xx/5xx | Throws script error — must wrap in try/catch | → See ZohoGuideRoot §1.2.1 |
| API response types are stable | Deluge auto-casts — always probe type dynamically | → See ZohoGuideRoot §1.6 |
| Desk can read Analytics tables (`zoho.reports.getData`) | ❌ `getData` does NOT exist in Desk Deluge. `zoho.reports.createRow()`/`updateData()`/`deleteRow()` are **silent no-ops** in Desk. Use `invokeUrl` + PUT /rows + CONFIG for writes. For reads, use a Zoho Flow intermediary. | §D.1 |
| `zoho.reports.*` functions work in Desk Deluge | ⚠️ **Silent no-op** — returns success but does NOT write data in Desk. Use `invokeUrl` + PUT /rows + CONFIG instead. | §D.4, §D-G.1 |
| `zoho.encryption.urlEncode` works in Desk | ❌ Not available in Desk. Use manual `replaceAll`. | §D.1, §D.4 |
| Desk Deluge supports `while` loops | ❌ Forbidden — use explicit sequential fetches | §D.1 |
| Desk Ticket ID available in `conversation.completed` entity | ❌ NOT in the payload — ticket may not exist yet when event fires | → See ZohoGuideRoot §7.18.19 |
| SalesIQ and Desk Deluge are interchangeable | ❌ Fundamentally different engines. Desk: full `zoho.desk.*`, `keySet()` works, no regex issues. SalesIQ: no `zoho.desk.*`, no `keySet()`, regex pre-validator. | → See ZohoGuideRoot §7.18.16 |
| Desk Ticket ID exists in `conversation.completed` payload | ❌ NOT present — cannot directly update ticket from SalesIQ workflow | → See ZohoGuideRoot §7.18.1 |

### §D-G.3 — ZQL Workspace Selection (Desk Context)

When generating ZQL for Analytics tables imported from Desk, select the correct date function syntax based on the target workspace:

| Need | Fallback Dialect (e.g. `YOUR_WORKSPACE_NAME`) | Standard Dialect |
|------|-------------------------------------------|------------------|
| Current timestamp | `LOCALTIMESTAMP()` | `current_timestamp()` |
| Day-level offset | `ADDDATE(LOCALTIMESTAMP(), -N)` | `dateadd('day', -N, current_timestamp())` |
| Hour extraction | `EXTRACT(HOUR FROM column)` | `HOUR(column)` |

**Rules:**
- Prefer **Fallback Dialect** for all `YOUR_WORKSPACE_NAME` workspace queries — `dateadd()` fails regardless of quoting in that workspace.
- `LIMIT` is unsupported everywhere — use `FETCH FIRST N ROWS ONLY`.
- Column names in imported Desk tables are **display names**, not API keys — verify via ZQL Explorer.

→ See ZohoGuideRoot §4.4.4 for the complete Desk imported table column name mapping.
→ See ZohoGuideRoot §4.4.5 for empirically verified ZQL date functions.
