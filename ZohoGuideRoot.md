# Zoho Compliance & Deluge Coding Standards — Root Guide

This is the **pervasive knowledge base** for all Zoho Deluge development. It applies to ALL environments (Flow, Desk, SalesIQ, Analytics). Environment-specific rules live in separate files — load only what you need.

**Last Updated:** 2026-07-24

> **⚠ AI AGENT INSTRUCTION:** You MUST read this file first before generating any Deluge code. After loading this file, load ONLY the environment file(s) relevant to your current task. Do NOT load environment files for environments you are not working in.

---

## Load Rules

| File | When to Load |
|------|-------------|
| **This file (Root)** | **ALWAYS** — for any Zoho work |
| `ZohoFlowGuide.md` | Loading for Zoho Flow work — on event hook custom workflows for events like logon and logoff |
| `ZohoDeskGuide.md` | Loading for Zoho Desk work — the ticket system |
| `ZohoSalesIQGuide.md` | Loading for Zoho SalesIQ work — the live chat system for website widget and social media |
| `ZohoAnalyticsGuide.md` | Loading for Zoho Analytics work — custom tables, table queries, dashboards and widgets. Data goes or reflects here |

**Example:** If building a Desk custom function that writes to Analytics, load Root + DeskGuide + AnalyticsGuide. Do NOT load FlowGuide or SalesIQGuide.

---

## 1. Workspace Ownership & Permission Model (CRITICAL)

> **⚠ OWNERSHIP TRANSFER BREAKS EVERYTHING.** This is the single most impactful infrastructure rule in this guide. Violating it causes silent failures that are extremely difficult to diagnose.

### 1.1 The Rule

**The developer account (the one writing Deluge code) MUST be the Owner or Workspace Administrator of ALL components in the stack:**

| Component | What "Ownership" Means | What Breaks Without It |
|-----------|----------------------|----------------------|
| **Zoho Flow** | Flow is created under the developer's account | Triggers may not fire; connections are environment-scoped |
| **Zoho Desk** | Desk custom functions run as the authenticated user | `invokeUrl` calls use the connection owner's token |
| **Zoho Analytics Workspace** | Developer is Owner or **Workspace Administrator** | **INSERT/UPDATE/DELETE via API fail with error 7301** |
| **OAuth Connections** | Created by the developer in the correct environment | Connection scope ≠ workspace permission (see §1.2) |

### 1.2 API Scope ≠ Workspace Owner (Error 7301)

**The #1 trap:** A connection can have `ZohoAnalytics.data.all` scope and show "Connected" status, yet STILL fail on writes. Even having **Workspace Administrator** role is not sufficient. The connection owner must be the **Workspace Owner**.

**Error 7301:** `"You (Your_Name) do not have the permission to do this operation. Only Workspace Admin has the permission."`

**Misleading error message:** The API says "Workspace Admin" but in practice, **only the Workspace Owner** can perform INSERT/UPDATE/DELETE via API. Workspace Administrators (a separate role) are still rejected.

**Flow vs Desk — different rules:**

| Environment | Can Workspace Administrator Write? | Can Workspace Owner Write? |
|-------------|-----------------------------------|---------------------------|
| **Zoho Flow** | ❌ **Read-only** — Admins can read via `invokeUrl` GET, but PUT/POST/DELETE fail with 7301 | ✅ Full read/write |
| **Zoho Desk** | ✅ **Can write** — Desk does NOT enforce the Owner-only rule. Admins and even shared users with write permission can write via `invokeUrl` | ✅ Full read/write |

**Why this matters:** A function that works in Desk (writing to Analytics) may fail identically when ported to Flow, even with the same connection, same scopes, and same workspace role. The permission model is environment-specific.

**Symptom in Flow:** READ operations (GET /data) succeed. WRITE operations (PUT /rows, POST /rows) fail with `SECURITY_NOT_PERMITTED`.

**Why READ works but WRITE fails in Flow:** Workspace Administrators can read shared views via API. The API scope `ZohoAnalytics.data.all` lets the token *call* the endpoint, but the workspace permission layer rejects the write operation because only the Owner can write via API in Flow.

### 1.3 What Happens on Ownership Transfer

When a workspace or project is transferred to a different Zoho account:

| Before Transfer | After Transfer |
|----------------|---------------|
| Developer owns workspace → full API read/write | New owner has full access; **original developer loses ownership** |
| Flow connections work end-to-end | Flow connections still show "Connected" (API scopes unchanged) |
| PUT /rows succeeds | PUT /rows fails with **error 7301** — silent, no connection error |
| Desk `invokeUrl` writes land | Desk `invokeUrl` writes fail with same 7301 |

**Critical:** The connection does NOT break or show an error. It appears healthy. The only indication is the 7301 error in the API response body.

**Note:** Being re-added as Workspace Administrator is NOT sufficient. The connection owner must be the **Workspace Owner** to perform writes via API.

### 1.4 Empirical Evidence (Example Deployment)

| Session | What Happened | Resolution |
|---------|--------------|------------|
| Session 10 (Jul 14) | Desk `TicketRouter_Function` v18→v19: all `zoho.reports.*` writes were silent no-ops | Replaced with `invokeUrl` + PUT /rows + CONFIG — **worked** (Desk connection owner had workspace access) |
| Session 13 (Jul 22) | Flow `Reasign_Open_Count_Agents` v21→v24: PUT /rows returns `SECURITY_NOT_PERMITTED` | **Unresolved** — connection has correct scopes but Flow connection owner lacks Workspace Admin role |
| Session 14 (Jul 23) | Diagnosed via API error code documentation | Error 7301 = "Only Workspace Admin has the permission" |

**Key difference:** Desk was able to work around ownership issues by restructuring API calls (using `invokeUrl` with different auth patterns). Flow has no workaround — if the connection owner lacks Workspace Admin, writes are permanently blocked.

### 1.5 Prevention Checklist

Before deploying ANY Flow or Desk function that writes to Analytics:

- [ ] Verify the **connection owner** is the **Workspace Owner** (not just Administrator) in the target Analytics workspace
- [ ] Check: Zoho Analytics → Workspace → Settings → Manage Users → confirm role is **Owner** (not "Administrator", "Editor", or "Viewer")
- [ ] If workspace was transferred, **transfer ownership back** to the developer account (adding as Admin is NOT enough)
- [ ] Test with a PUT /rows call BEFORE deploying the full function
- [ ] **Never transfer workspace ownership** without re-establishing Owner for the developer account

### 1.6 Diagnostic Pattern

If writes fail with no obvious code issue:

1. Check the API response for **error code 7301**
2. If 7301 → check workspace **ownership**, not API scope or Admin role
3. Go to Zoho Analytics → Workspace → Settings → Manage Users
4. Confirm the connection owner is listed as **Owner** (not just Administrator)
5. If not → transfer workspace ownership to the developer account
6. Re-test the write

### 1.7 Ownership Transfer Propagation Delay (Jul 2026)

**Discovery:** After transferring workspace ownership and re-authorizing connections, the 7301 error can persist for **30+ minutes** before writes start succeeding. The connection shows "Connected" status immediately, but the permission change takes time to propagate through Zoho's infrastructure.

**Observed timeline (example deployment):**
| Time | Action | Result |
|------|--------|--------|
| ~11:36 | v25 deployed with corrected view ID + new connections | 7301 on `agent_status` table INSERT |
| ~11:47 | Second trigger event | Still 7301 |
| ~12:02 | Flow function edited (re-saved) | INSERT succeeds |

**Lesson:** After fixing ownership/permissions, allow **30-60 minutes** for propagation. Re-saving the Flow function may help trigger re-authorization. Do not assume the fix failed if writes don't land immediately.

### 1.8 Creating New Connections as Permission Reset (Jul 2026)

**Discovery:** When connections are stuck in a bad permission state (correct scopes but stale tokens), **deleting and re-creating** the connections from scratch can resolve the issue. This generates fresh OAuth tokens and clears any cached permission state.

**When to use this:**
- Connection shows "Connected" but writes fail with 7301
- Re-authorizing the existing connection doesn't help
- Workspace ownership was recently transferred

**Process:**
1. Note the existing connection names and their scopes
2. Delete the old connections in Flow → Settings → Connections
3. Create new connections with the same (or new) names and scopes
4. Update all function references to use the new connection names
5. Allow 30-60 minutes for propagation (see §1.7)

### 1.9 View ID Verification from Analytics URL (Jul 2026)

**Discovery:** Using the wrong view ID causes silent failures — the API returns success but writes land in a different table (or no table at all). View IDs are NOT obvious from the table name.

**How to get the correct view ID:**
1. Open the target table in Zoho Analytics
2. Look at the URL: `https://analytics.zoho.com/workspace/<WS_ID>/view/<VIEW_ID>`
3. The `<VIEW_ID>` portion is what goes in `staticAgentStatusViewId`
4. Example: `https://analytics.zoho.com/workspace/999999999999999999/view/999999999999999999` → view ID = `999999999999999999`

**Common mistake:** Using a view ID from a different workspace, or from an imported table that has a different view ID than expected. Always verify from the live URL.

---

## 2. Global Mandates (Universal Deluge Rules)

- **ALWAYS** use the canonical function template in §2.1.
- **USE STATICS**: Hard-code Org IDs, tokens, and data-center URLs as named **static constants** at the top of the function. This makes code portable and cloneable to other orgs.
- **STATIC NAMING**: Use clear static variable names (e.g., `staticOrgId`, `staticAnalyticsDomain`) so they're easily findable when cloning.
- **EXCEPTION**: `zoho.appdetails()` does NOT work in either Flow or Desk. Use static constants (see §2.1 canonical template).
- **ERROR HANDLING**: Wrap all logic in `try/catch`. Return or log a structured map with `success`, `data`, and `error` keys (except in Desk, which is VOID).
- **ENVIRONMENT AWARENESS**: Rules for Desk and Flow differ significantly. Apply the correct constraints based on the execution environment.

### 2.1 Canonical Function Template
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

### 2.2 Standard API Call Pattern (generic)
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

### 2.3 Advanced API Call Patterns (invokeurl)

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

### 2.4 Best-Practice Rules
**Required**
- Wrap logic in **try/catch**.
- Validate inputs first; fail fast with a clear error message.
- Document function header (name, purpose, inputs, returns).
- Define `staticOrgId` and other static constants at top of function (see §2.1 template).
- Return structured JSON: `{ success: bool, data: map/list, error: string }`.

**Avoid**
- Passing IDs as Flow variables (makes code non-portable).
- Silent failures—log or return structured errors.
- Creating records inside loops unless unavoidable.

### 2.5 Pre-Deployment Checklist (Functional & Security)
- [ ] Header present
- [ ] Inputs validated
- [ ] `try/catch` implemented
- [ ] `orgId` loaded dynamically
- [ ] No hard-coded secrets
- [ ] All failures return `{success:false}`
- [ ] Errors logged consistently
- [ ] Structured map output only (no raw text)

### 2.6 Centralized Logger (optional helper)
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

### 2.7 Map/List Auto-Casting Trap (CRITICAL)

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

### 2.8 Connections Are Environment-Specific (CRITICAL)

**Each Zoho environment (Flow, Desk, SalesIQ) has its own isolated Connection pool.** A connection created in Desk is NOT visible in Flow, and vice versa. When deploying custom functions:

1. **Flow Connections** — Created in Flow → Settings → Connections. Flow can use both literal strings and variables for `connection:`.
2. **Desk Connections** — Created in Desk → Setup → Connections. Desk custom functions require OAuth connections for `zoho.desk.update`, `invokeUrl`, etc.
3. **SalesIQ Connections** — Created in SalesIQ → Settings → Automate → Workflows → Connections. SalesIQ requires literal `CONNECTION LINKNAME` strings (variables cause validation errors).

**Common scopes needed across environments:**
- Zoho Analytics: `ZohoAnalytics.data.all`, `ZohoReports.data.ALL`
- Zoho Desk: `Desk.tickets.READ`, `Desk.tickets.WRITE`, `Desk.tickets.UPDATE`

---

## 3. AI Agent Execution Guardrails

This section captures rules specifically for AI agents generating or modifying Deluge code. These guardrails override generic Zoho documentation where they conflict.

### 3.1 Environment Constraints Quick Reference

| Constraint | Flow | Desk | SalesIQ |
|------------|------|------|---------|
| Return type | Map | VOID | Map |
| Event object | `trigger` | None (hook params) | `entity` |
| Analytics API | v2 REST (via `invokeurl`) | v2 REST via `invokeurl` + PUT /rows + CONFIG | v2 REST via `analytics_connection` + manual URL encoding |
| `else` | ❌ Forbidden | ✅ Works | ✅ Works |
| `continue` | ❌ Forbidden | ✅ Works | Untested |
| `instanceof` | ❌ Forbidden | ✅ Works | Untested |
| `info` statements | ❌ Forbidden | ✅ Allowed | ✅ Allowed |
| `while` loops | ✅ Works | ❌ Forbidden | Untested |
| `range()` | ✅ Works | ❌ Forbidden | Untested |
| Helper functions | ❌ Forbidden | ✅ Allowed | Untested |
| Signature rule | Line 1, `{` on same line | Standard | Standard |
| `zoho.appdetails()` | ❌ Fails | ❌ Fails | ❌ Likely fails |
| `now()` | ✅ Works | ✅ Works | ❌ Fails |
| `keySet()` | ✅ Works | ✅ Works | ❌ Fails (all Maps) |
| `getType()` | ✅ Works | ✅ Works | ❌ Fails |
| `containsKey()` | ✅ Works | ✅ Works | ✅ Works |
| `invokeUrl` | ✅ Works | ✅ Works | ✅ Works |
| `zoho.reports.getData` | ✅ Works | ❌ Does NOT exist | ❌ Parse error |
| `zoho.encryption.urlEncode` | ✅ Works | ❌ Not available | ❌ Parse error |
| `for each` loops | ✅ Works | ✅ Works | ✅ Works |
| External non-Zoho APIs | ✅ Works | ✅ Works | ✅ Works (40s timeout) |
| Connections | OAuth + STRING var | OAuth + STRING var | OAuth only, literal CONNECTION LINKNAME |

### 3.2 Common AI Hallucination Risks

| Hallucination | Reality | Ref |
|---------------|---------|-----|
| `sortBy()` exists in Deluge | Does not exist — use extract-sort-rebuild | FlowGuide §F.1 |
| `statusType` works as search filter | Fails in all environments — use `status` | DeskGuide §D.3 |
| `dateadd()` is portable | Fails in some workspaces | AnalyticsGuide §A.3 |
| Blank lines before `try` are safe in Desk | Causes "Improper Statement" in Desk | DeskGuide §D-A |
| `invokeurl` returns error map on 4xx/5xx | Throws script error — must wrap in try/catch | §2.3 |
| API response types are stable | Deluge auto-casts — always probe type dynamically | §2.7 |
| `now()` works in all Deluge environments | ❌ Fails in SalesIQ — use `zoho.currenttime` | SalesIQGuide §S.2 |
| `keySet()` works in all Deluge environments | ❌ Fails in SalesIQ — use `entity.toString()` | SalesIQGuide §S.2 |
| SalesIQ Deluge = Desk Deluge | They are different engines | SalesIQGuide §S.2 |
| Desk Ticket ID exists in `conversation.completed` payload | ❌ NOT present | SalesIQGuide §S.1 |
| Desk API supports `email` search parameter | ❌ UNPROCESSABLE_ENTITY | SalesIQGuide §S.2 |
| `zoho.reports.*` functions work in Desk Deluge | ⚠️ Silent no-op — use `invokeUrl` + PUT /rows | DeskGuide §D.4 |
| SalesIQ can directly update Desk tickets | ❌ Wrong architecture | SalesIQGuide §S.3 |
| `parameters: Map()` works for PUT/DELETE CONFIG | ❌ Unreliable — use `zoho.encryption.urlEncode()` | FlowGuide §F.6 |
| `ifelse()` works in Flow | ❌ NOT available — parser error | FlowGuide §F.1 |
| Desk can read Analytics tables | ❌ `getData` does not exist in Desk | DeskGuide §D.4 |

### 3.3 Absolute Resolution of Document Contradictions

| Resolved Issue | Final Rule | Source |
|----------------|-----------|--------|
| **Desk whitespace before `try`** | NEVER leave a blank line between variable initializations and the opening `try` in Desk Deluge. Causes "Improper Statement" parser errors. | DeskGuide §D-A |
| **`statusType` as search filter** | Completely disregard `statusType` and `isOpen`. Always use `status` (e.g., `crit.put("status", "Open")`). Does NOT work in any environment. | DeskGuide §D.3 |

### 3.4 Defensive Type Verification (JSON/Map Auto-Casting)

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

### 3.5 Advanced Logic & Data Mutation Guardrails

#### 3.5.1 The Map.toString() JSON Payload Trap
- **Rule**: Never use `myMap.toString()` to generate a JSON payload for a standard third-party REST API. Deluge serializes maps using equal signs (e.g., `{key=value}`), which will cause HTTP 400 Bad Request errors.
- **Resolution**: For third-party APIs, use an explicitly escaped raw string body (§2.3). Only use `myMap.toString()` when passing a `CONFIG` string to Zoho Analytics v2, as Zoho's internal systems natively understand the Deluge map string format.

#### 3.5.2 Safe Data Cleansing (Avoiding getAlphaNumeric())
- **Rule**: Never use `.getAlphaNumeric()` to clean phone numbers, email addresses, or domain names. It removes structural characters like `+`, `-`, `@`, and `.`.
- **Resolution**: Use explicit regex or character replacement instead:
```deluge
// Correct pattern for phone numbers
cleanPhone = rawPhone.toString().replaceAll(" ", "").replaceAll("-", "");
```

### 3.6 Consolidated Pre-Deployment Checklist
- [ ] **Header**: Function name, purpose, and inputs documented.
- [ ] **Validation**: Input null/empty checks implemented.
- [ ] **Safety**: All logic wrapped in `try/catch`.
- [ ] **Environment**:
  - If **Flow**: No `else`, `continue`, `instanceof`, custom helper functions. `info` statements ARE supported and viewable in execution logs. Function signature on the first non-whitespace line with `{` on the same line.
  - If **Desk**: No return values; no `function()` declarations (parser rejects). Analytics writes via `invokeUrl` + PUT /rows + CONFIG only (`zoho.reports.*` are silent no-ops). All helper logic must be inlined.
  - If **SalesIQ**: No `now()`, no `keySet()`, no `zoho.desk.*`, no `zoho.authtoken`. Connection must be literal string. Return type must be Map.
- [ ] **Analytics**: `ZANALYTICS-ORGID` header included for v2 calls.
- [ ] **Hard-coding**: No Org IDs, hard-coded URLs, or tokens (use static constants).
- [ ] **Flow Cache Persistence**: If using cache, ensure string-based pattern is correctly implemented.
- [ ] **ZQL**: If using Analytics SQL, ensure compliance with ZQL constraints.

---

## 4. External Reference: The Workflow Academy (GitHub)

**URL:** https://github.com/TheWorkflowAcademy

The Workflow Academy is a Zoho developer/consultant community with 80+ public repos of Deluge scripts covering CRM, Desk, Flow, Analytics, Books, Creator, and more.

### Local Clone
The most relevant repos (CRM, Desk, Flow, Analytics) have been cloned to `Zoho/FromGithub/` for offline reference and local search. See `Zoho/FromGithub/README.md` for a complete catalog with descriptions of all 29 cloned repos.

### Remaining Repos (Online Only)
The remaining ~55 repos (Books, Creator, Inventory, Campaigns, WorkDrive, etc.) are available directly at https://github.com/TheWorkflowAcademy — browse as needed.

---

# End of Zoho Root Guide
