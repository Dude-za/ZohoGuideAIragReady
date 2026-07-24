# Zoho Analytics Guide — Custom Tables, Queries, Dashboards and Widgets

**Scope:** Zoho Analytics — custom tables, table queries, dashboards and widgets. Data goes or reflects here.

> This file is environment-specific. Load `ZohoGuideRoot.md` first for pervasive rules (canonical template, API patterns, ownership model, pre-deployment checklist).

**Last Updated:** 2026-07-24

---

## §A.1 — Creating Tables in Analytics — CSV Header-Plus-Sample Method

**Best practice:** When creating a new table in Analytics, import a CSV with a header row (column names) plus at least one sample data row. The header defines column names; Analytics auto-detects types from the sample data. Delete the sample row after import.

**Workflow:**
1. Create a `.csv` file with column names as the header row and one representative data row
2. In Zoho Analytics → Workspace → **Create → Table → Import Data → CSV**
3. Drop in the CSV — Analytics parses the header as column names and detects types from the sample values
4. Verify column names and types in the mapping step, finish import
5. Delete the sample data row from the table after creation

**Why not JSON or XML:**
- JSON/XML table descriptions are parsed as **data rows**, not schema definitions
- The column names, types, and descriptions end up as cell values in the table
- Only CSV imports treat the first row as a header for column naming
- JSON/XML are still useful as **documentation** (kept alongside the project files) but cannot drive the import itself

**Example — table creation files:**

| File | Purpose |
|------|---------|
| `example_table.csv` | Import file (header + 1 sample row) |
| `example_table.json` | Documentation only (schema reference) |

---

## §A.2 — ZQL (Zoho Query Language) Standards

### ZQL Basics & Rules

**What is ZQL?** ZQL (Zoho Query Language) is Zoho's SQL dialect used in Analytics Query Tables. It supports standard SQL SELECT statements with some variations.

**Key ZQL Rules:**
- **Comments:** Use `/* */` syntax (not `//`)
- **Column names:** Use double quotes: `"columnName"`
- **String values:** Use single quotes: `'value'`
- **Row limiting:** Use `FETCH FIRST X ROWS ONLY`
- **Type conversion:** Use `to_number(column)`

### ZQL Templates (Project-Tested)

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

### General Observations

- **No SQL in KPI widgets** → KPIs must bind to a dataset (table/query table) and aggregate a **single row**; use Query Tables for any SQL logic and then point the KPI to that Query Table.
- **Type conversion**: Prefer `to_number(<expr>)` over `CAST(... AS ...)` for numeric coercion.
- **Row limiting**: Some engines reject `LIMIT`; use `FETCH FIRST 1 ROWS ONLY` or a **MAX(timestamp)** subquery.
- **Identifiers**: Quote reserved/mixed-case names with **double quotes** (e.g., `"key"`, `"timestamp"`).
- **Time math**: If `NOW() - INTERVAL '24' HOUR` is unsupported, use `dateadd('hour', -24, current_timestamp())`.
- **Report cache vs table types**: After changing a column to number in a Query Table, the **report** can still cache it as *Actual(text)*. Re-add the field as a **measure (Sum/Avg)** or **Reset All** to refresh bindings.

### Reliable Patterns Adopted

- **Latest value KPIs** via Query Tables that return **one row** (e.g., latest OnlineCount or RouterState), so KPI aggregation becomes a no-op.
- **Heatmaps/Trends** via Query Tables that pre-aggregate (`DATE()`, `HOUR()`, `AVG(...)`).
- **24h windows** via `ADDDATE(LOCALTIMESTAMP(), -1)` for cross-workspace compatibility (day-level only; see §A.3 for hour-level caveats).
- **Numeric enforcement** via `to_number(...)` inside the SELECT.

### Minimal ZQL Snippets (Project-Tested)

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
  ⚠ **Avoid `dateadd()`:** Not available in fallback-dialect workspaces. `ADDDATE()` is preferred and works in all tested workspaces (see §A.3). Note: `ADDDATE()` supports day-level only; for hour-level windows, see §A.3 for workspace-specific options.

---

## §A.3 — ZQL Date Functions & Empirical Findings

### Empirically Verified ZQL Date Functions

Tested in fallback-dialect workspaces. ZQL dialect varies by workspace — see §A-G for workspace selection hierarchy.

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

### Imported Desk Tables — Column Names

**IMPORTANT:** When importing Zoho Desk data into Analytics, the column names shown in the UI are **display names**, not the ZQL column names.

**Tickets Table — Column Name Mapping:**

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

**Agents Table — Column Name Mapping:**

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

### SalesIQ Conversations Table

**SalesIQ Conversations table:** Imported to Analytics as `Conversations`. Links to tickets via `Zoho Desk Ticket ID`. Key columns: `Initiated Time`, `End Time`, `Chat Duration in Minutes`, `Status` (Attended/Closed/Bot Transfer Missed), `Ended By`, `Missed Reason`. Useful for cross-referencing chat outcomes against ticket data.

### Channel Values (Analytics Import)

Beyond Email/Chat/Phone, SalesIQ integration creates per-platform tags: `FB Messenger`, `IG Messenger`, `Instagram`, `Facebook`. The Analytics import preserves these verbatim. When filtering in ZQL or dashboard widgets, expect the full set.

Full channel tag list: `Chat`, `FB Messenger`, `IG Messenger`, `Instagram`, `Facebook`, plus `Web` (legacy/Zendesk import, inactive since May 2026).

---

## §A.4 — Duration String Parsing — REPLACE Auto-Converts to Seconds

**Discovery:** Zoho's duration format `"X days YY:ZZ hrs"` (e.g., `"0 days 00:02 hrs"`) is stored as text, but passing it through `REPLACE(col, 'days ', '')` **auto-converts to total seconds**.

**Working pattern:**
```sql
-- Returns total seconds (numeric) — divide by 60 for minutes
REPLACE(tickets."Total Response Time in Business Hours", 'days ', '') / 60 AS response_minutes

-- Can be aggregated with AVG
AVG(REPLACE(tickets."Total Response Time in Business Hours", 'days ', '') / 60) AS avg_response_minutes
```

**Tested examples:**

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

## §A.5 — Analytics v2 REST API

> Environment-specific write patterns (Flow vs Desk vs SalesIQ) are documented in their respective environment guides. This section covers the **Analytics-side** of the API: headers, base URL, upserts, and response format.

→ See ZohoGuideRoot §1 for workspace ownership model and error 7301 prevention.
→ See ZohoFlowGuide §F.6 for Flow → Analytics write patterns.
→ See ZohoDeskGuide §D.4 for Desk → Analytics write patterns.
→ See ZohoSalesIQGuide for SalesIQ → Analytics write patterns.

### Headers & Base URL

- **Headers**: Always include `"ZANALYTICS-ORGID": <org_id>`.
- **Base URL**: Must be data-center correct (e.g., `analyticsapi.zoho.com` vs `.eu`).
- **Upserts**: Use **Update Row** with `addIfNotExist: true` instead of Bulk Import for state management.
- **Response**: Export responses store records in the `data` key, not `rows`.

### ⚠ WRITE PERMISSIONS

PUT /rows will return error 7301 `SECURITY_NOT_PERMITTED` if the connection owner is not the **Workspace Owner** in the target workspace. READ works fine for regular users and Administrators — only writes require Owner role. → See ZohoGuideRoot §1 for the full ownership model.

### ⚠ CONFIG ENCODING

Always use `zoho.encryption.urlEncode(cfg.toString())` and append to the URL string. Do NOT use `parameters: Map()` for PUT/DELETE — the CONFIG parameter may not be passed correctly, causing silent failures. → See ZohoGuideRoot §3.5 for proven patterns.

### Analytics v2 — Update Row (Preferred for State Upserts)

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

### Analytics v2 — Export (Read Rows)

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

### Batch Constraints (Flow Only)

When using the preferred `PUT` method on `/rows` with `addIfNotExist: true` for bulk state updates, the payload MUST be capped at a maximum of **100 rows per transaction**. If processing an incoming list larger than 100 items, chunk the payload or process via sequential execution — otherwise Analytics will reject the request with a payload size exception.

→ See ZohoAnalyticsGuide §A.5 (Batch Constraints)

### Map.toString() JSON Payload Trap

Never use `myMap.toString()` to generate a JSON payload for a standard third-party REST API. Deluge serializes maps using equal signs (e.g., `{key=value}`), which causes HTTP 400 Bad Request errors. Only use `myMap.toString()` when passing a `CONFIG` string to Zoho Analytics v2, as Zoho's internal systems natively understand the Deluge map string format.

→ See ZohoAnalyticsGuide §A.5 (Map.toString() JSON Payload Trap)

### View ID Verification from Analytics URL

Using the wrong view ID causes silent failures — the API returns success but writes land in a different table (or no table at all). View IDs are NOT obvious from the table name.

**How to get the correct view ID:**
1. Open the target table in Zoho Analytics
2. Look at the URL: `https://analytics.zoho.com/workspace/<WS_ID>/view/<VIEW_ID>`
3. The `<VIEW_ID>` portion is what goes in the view ID constant
4. Example: `https://analytics.zoho.com/workspace/999999999999999999/view/999999999999999999` → view ID = `999999999999999999`

**Common mistake:** Using a view ID from a different workspace, or from an imported table that has a different view ID than expected. Always verify from the live URL.

→ See ZohoGuideRoot §1.9

---

## §A-A — Analytics Lessons Learned

### Debug Logging via Analytics (Recommended Architecture)

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

### Analytics v1 Rules (Desk Only)

- **⚠ `zoho.reports.*` functions are silent no-ops in Desk** — they return success but do NOT write data.
- For Desk → Analytics writes, use `invokeUrl` + PUT /rows + CONFIG.
- `zoho.reports.*` only works reliably in **Flow** (see §A.5 for v2 REST pattern).
- **⚠ WRITE PERMISSIONS:** Even with correct API calls, writes will fail with error 7301 if the connection owner is not the **Workspace Owner** (Administrator role is NOT sufficient). → See ZohoGuideRoot §1.

### Deleting Rows (Empirically Verified)

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

### Encryption URL Encode for Analytics v2

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

→ See ZohoGuideRoot §7.13

### Workspace Ownership Transfer Breaks Analytics Writes

**Issue:** Flow `invokeUrl` PUT /rows to Analytics returns `SECURITY_NOT_PERMITTED` (error 7301) despite connection having correct API scopes and showing "Connected" status.

**Root cause:** The connection owner is not the **Workspace Owner** in the target Analytics workspace. API scopes (`ZohoAnalytics.data.all`) control endpoint access; only the Workspace Owner can perform writes via API in Flow. Even Workspace Administrator role is not sufficient in Flow.

**Misleading error:** The API error says "Only Workspace Admin has the permission" but in practice, only the **Owner** role works for API writes in Flow.

**Flow vs Desk distinction:**
- **Flow:** Workspace Administrators are **read-only**. Only the Owner can write. This is an environment-specific permission boundary.
- **Desk:** Does NOT enforce the same rule. Desk can write to Analytics even with Administrator or shared-user roles.

**Fix:** Zoho Analytics → Workspace → Settings → Manage Users → **transfer workspace ownership** back to the developer account. Adding as Administrator is NOT enough for Flow.

**Prevention:** The developer account should own ALL components: code, triggers, connections, AND workspaces. Never transfer workspace ownership without re-establishing Owner for the developer account.

→ See ZohoGuideRoot §1 for full ownership model.

### Ownership Transfer Propagation Delay

After transferring workspace ownership and re-authorizing connections, the 7301 error can persist for **30+ minutes** before writes start succeeding. The connection shows "Connected" status immediately, but the permission change takes time to propagate through Zoho's infrastructure.

**Observed timeline:**
| Time | Action | Result |
|------|--------|--------|
| ~11:36 | v25 deployed with corrected view ID + new connections | 7301 on agent_status INSERT |
| ~11:47 | Second trigger event | Still 7301 |
| ~12:02 | Flow function edited (re-saved) | INSERT succeeds |

**Lesson:** After fixing ownership/permissions, allow **30-60 minutes** for propagation. Re-saving the Flow function may help trigger re-authorization. Do not assume the fix failed if writes don't land immediately.

→ See ZohoGuideRoot §1.7

### Creating New Connections as Permission Reset

When connections are stuck in a bad permission state (correct scopes but stale tokens), **deleting and re-creating** the connections from scratch can resolve the issue. This generates fresh OAuth tokens and clears any cached permission state.

**When to use this:**
- Connection shows "Connected" but writes fail with 7301
- Re-authorizing the existing connection doesn't help
- Workspace ownership was recently transferred

**Process:**
1. Note the existing connection names and their scopes
2. Delete the old connections in Flow → Settings → Connections
3. Create new connections with the same (or new) names and scopes
4. Update all function references to use the new connection names
5. Allow 30-60 minutes for propagation (see above)

→ See ZohoGuideRoot §1.8

### Safe Data Cleansing (Avoiding getAlphaNumeric)

Never use `.getAlphaNumeric()` to clean phone numbers, email addresses, or domain names. It removes structural characters like `+`, `-`, `@`, and `.`.

**Correct pattern:**
```deluge
// Correct pattern for phone numbers
cleanPhone = rawPhone.toString().replaceAll(" ", "").replaceAll("-", "");
```

→ See ZohoAnalyticsGuide §A-A (Safe Data Cleansing)

### Transaction Scope and Loop Re-initialization

When running a `for each` loop inside Zoho Flow that contains a `try/catch` block, state variables must be re-initialized at the immediate top of the loop body to prevent transaction-scope freezing.

**Pattern:**
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

→ See ZohoFlowGuide §F-G.2 (Transaction Scope and Loop Re-initialization)

---

## §A-G — Analytics AI Guardrails

### ZQL Workspace Selection Hierarchy

When generating ZQL date/time functions, select based on the target workspace profile:

| Need | Fallback Dialect (e.g. `YOUR_WORKSPACE_NAME`) | Standard Dialect |
|------|-------------------------------------------|------------------|
| Current timestamp | `LOCALTIMESTAMP()` | `current_timestamp()` |
| Day-level offset | `ADDDATE(LOCALTIMESTAMP(), -N)` | `dateadd('day', -N, current_timestamp())` |
| Hour extraction | `EXTRACT(HOUR FROM column)` | `HOUR(column)` |

**Rules:**
- Prefer **Fallback Dialect** for all `YOUR_WORKSPACE_NAME` workspace queries — `dateadd()` fails regardless of quoting in that workspace.
- `LIMIT` is unsupported everywhere — use `FETCH FIRST N ROWS ONLY`. → See ZohoGuideRoot §3.1
- Column names in imported tables are **display names**, not API keys — verify via ZQL Explorer.

### LIMIT Unsupported

`LIMIT` is unsupported in ZQL across all tested workspaces. Always use `FETCH FIRST N ROWS ONLY` for row limiting, or use a `MAX(timestamp)` subquery pattern instead.

### Common AI Hallucination Risks (Analytics-Specific)

| Hallucination | Reality | Ref |
|---------------|---------|-----|
| `dateadd()` is portable | Fails in fallback-dialect workspaces | §A.3 |
| `LIMIT` works in ZQL | Unsupported — use `FETCH FIRST N ROWS ONLY` | §A-G |
| Column names are case-insensitive | Must match display names exactly (e.g., `"Ticket Owner"`, not `"ticket_owner"`) | §A.3 |
| `REPLACE` on duration columns is string replacement | Implicitly converts to numeric seconds | §A.4 |
| `zoho.reports.*` writes work in Desk | Silent no-op — returns success, no data written | §A-A |
| `parameters: Map()` works for PUT/DELETE CONFIG | Unreliable — CONFIG may not be passed correctly. Always use `zoho.encryption.urlEncode()` + URL string | §A.5 |
| API response key is `rows` | It's `data` — use `resp.get("data")` | §A.5 |
| `HOUR(column)` works in ZQL | Unsupported — use `EXTRACT(HOUR FROM column)` | §A.3 |
| `TO_SECONDS(col)` works in ZQL | NOT supported | §A.3 |
| `current_date` works in ZQL | NOT supported — use `LOCALTIMESTAMP()` | §A.3 |
