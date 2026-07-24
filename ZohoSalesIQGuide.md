# Zoho SalesIQ Guide — Live Chat System for Website Widget and Social Media

**Scope:** Zoho SalesIQ — the live chat system for website widget and social media.

> This file is environment-specific. Load `ZohoGuideRoot.md` first for pervasive rules (canonical template, API patterns, ownership model, pre-deployment checklist).

**Last Updated:** 2026-07-24

---

## §S.1 — Workflow Deluge Constraints

SalesIQ Workflows have their own Deluge engine that differs from both Desk and Flow. These findings are from trial-and-error testing with `conversation.completed` event workflows.

---

### §S.1.1 — Event Object Structure

SalesIQ Workflows use an `entity` object to pass event data. The structure differs significantly from what Zoho docs suggest — empirical discovery via `entity.toString()` dumps is essential. `keySet()` is NOT available.

**Empirically verified top-level fields (`conversation.completed` event, Jul 2026):**

| Field | Type | Description | Example |
|-------|------|-------------|---------|
| `id` | string | SalesIQ conversation ID | `"999999999999999999"` |
| `conversation_id` | string | Same as id | `"999999999999999999"` |
| `visitor_conversation_id` | string | Internal hash | `"siq99999..."` |
| `reference_id` | string | External reference | `"99999"` |
| `ended_by` | string | Who ended it | `"agent"`, `"visitor"`, `"system"` |
| `status` | string | Chat outcome | `"Attended Online"`, `"Missed"` |
| `question` | string | Visitor's first message / email | `"visitor@example.com"` |
| `department_id` | string | Department ID (NOT a sub-object) | `"999999999999999999"` |
| `thread_number` | number | Thread count | `1` |
| `app_id` | string | SalesIQ app/widget ID | `"999999999999999999"` |
| `chat_initiated_url` | string | Page where chat started | `"https://www.yourcompany.com/page/home"` |
| `opened_time` | string | Epoch milliseconds (start) | `"9999999999999"` |
| `end_time` | string | Epoch milliseconds (end) | `"9999999999999"` |

**Empirically verified sub-objects (field names per official docs, Jul 2026):**

| Sub-object | Key Fields | Access Pattern |
|------------|------------|----------------|
| `visitor` | `name`, `first_name`, `last_name`, `email_id` (⚠ NOT `email`), `phone`, `id`, `type`, `channel`, `salutation` | `entity.get("visitor").get("email_id")` |
| `owner` | `name`, `email_id`, `id`, `type` (note: `email` also exists but `email_id` is standard) | `entity.get("owner").get("email_id")` |
| `participants` | List of operator IDs | `entity.get("participants")` |
| `supervisors` | List of supervisor IDs | `entity.get("supervisors")` |
| `chat_status` | `label`, `status_code`, `state_key`, `state`, `status_key`, `id` (⚠ NOT `name` or `type`) | `entity.get("chat_status").get("label")` |
| `company` | `name`, `id`, `domain`, `status` | `entity.get("company").get("name")` |
| `customer_info` | Custom fields (variable) | `entity.get("customer_info").get("Branch")` |
| `_meta` | Internal metadata (may include `delugeerror` on failure) | `entity.get("_meta")` |

**Access pattern examples (RAG-optimized — each path is a complete, copyable snippet):**

```deluge
// Visitor email (⚠ use email_id, NOT email)
visitorEmail = entity.get("visitor").get("email_id").toString();

// Visitor name
visitorName = entity.get("visitor").get("name").toString();

// Owner email (agent who handled the chat)
ownerEmail = entity.get("owner").get("email_id").toString();

// Chat status label
chatLabel = entity.get("chat_status").get("label").toString();

// Company name
companyName = entity.get("company").get("name").toString();

// Custom field from customer_info
branchName = entity.get("customer_info").get("Branch").toString();
```

**Key structural differences from assumed layout:**
- ❌ No `conversation` sub-object — data is flat at the top level
- ❌ No `operator` key — use `owner` instead
- ❌ No `department` Map — use `department_id` string directly
- ❌ Sub-object fields use `email_id` NOT `email` (common Zoho convention)
- ❌ Timestamps are epoch milliseconds, NOT ISO 8601 strings
- ✅ `visitor` sub-object exists as expected
- ✅ `ended_by` value is `"agent"` (lowercase), not `"Operator"`

---

### §S.1.2 — Verified Working Constructs

| Construct | Status | Notes |
|-----------|--------|-------|
| `entity.get("key")` | ✅ Works | Standard Map access |
| `entity.toString()` | ✅ Works | Full JSON dump for debugging |
| `info "text"` | ✅ Works | Visible in Workflow Execution Logs |
| `ifnull(val, default)` | ✅ Works | Standard Deluge function |
| `.toString()` | ✅ Works | Type coercion |
| `.left(n)` | ✅ Works | String truncation |
| String concatenation (`+`) | ✅ Works | Standard |
| `try { } catch (e) { }` | ✅ Works | Full multi-line blocks |
| `for each` loops | ✅ Works | List iteration confirmed (Test I) |
| `if/else` | ✅ Works | Standard |
| `Map()` constructor | ✅ Works | Capital M |
| `List()` constructor | ✅ Works | Standard |
| `map.put(key, val)` | ✅ Works | Standard |
| `map.containsKey("key")` | ✅ Works | Works on Maps returned by invokeUrl (Test B, D, J) |
| `zoho.currenttime` | ✅ Works | Special built-in keyword. Returns current datetime. NOTE: `zoho` is NOT a general namespace object — only `currenttime` is recognized by the parser. |
| `return` | ✅ Works | Script must return a Map (not string) |
| `invokeUrl` | ✅ Works | Confirmed available. Syntax: `invokeUrl [ url: type: connection: headers: body: ]` (Tests A–K) |
| `invokeUrl` with `parameters:` Map | ✅ Works | GET params can be passed as Map instead of URL string (Test K) |
| Connections in invokeUrl | ✅ Works | `connection: "literal_name"` — literal string required (Test B, D, E, G, J) |

---

### §S.1.3 — Verified Non-Working Constructs

| Construct | Error Message | Alternative |
|-----------|---------------|-------------|
| `now()` | `Not able to find 'now' function` | Use `zoho.currenttime` instead |
| `keySet()` on ANY Map | `Not able to find 'keySet' function` | Use `containsKey()` for known keys, or dump `toString()` to discover keys manually. `keySet()` is completely absent from SalesIQ's Deluge engine — it's not just an `entity` issue. |
| `getType()` | `Not able to find 'getType' function` | Not available. Response types are always Map from invokeUrl. |
| `zoho.authtoken` | `Variable 'zoho' is not defined` | PARSE-TIME error — try/catch CANNOT catch. Use Connections exclusively for API auth. |
| `zoho.encryption` | Same parse error | `zoho` is NOT a general namespace. Use manual URL encoding if needed. |
| `zoho.encryption.urlEncode` | Same parse error | NOT available. `zoho` not a namespace object. |
| `zoho.desk.*` (any) | Same parse error | Desk integration tasks NOT imported in SalesIQ Deluge. |
| `zoho.desk.searchRecords` | Same parse error | Parse-time failure, not runtime. Cannot be try/caught. |
| `zoho.reports.createRow` | Connection validation error | Connections from Desk/Flow don't carry over. Create in SalesIQ separately. |
| `replaceAll("{", ...)` | `{ is not a valid regular expression` | `replaceAll` uses Java regex. Use `replaceAll("[{]", ...)` for literal braces. See §S.6 for full regex escaping guide. |

---

## §S.2 — Capability Map (v5.3 Probe)

The full capability map, built across v5.0–v5.3 iterations. The v5.3 probe (`salesiq_probe_v5_deluge.txt`) tested 12 capabilities across both Desk and Analytics Connections.

**Connection Setup:**
- `desk_connection` → Zoho Desk, scopes: `Desk.tickets.READ`, `Desk.tickets.WRITE`, `Desk.tickets.UPDATE`
- `analytics_connection` → Zoho Analytics, scopes: `ZohoAnalytics.data.all`, `ZohoReports.data.ALL`

### Test Results (Tests A–K)

| Test | What | Result | Detail |
|------|------|--------|--------|
| **A** | `invokeUrl` syntax + Desk reachability | ✅ SUCCESS | Function exists, no auth needed for basic ping |
| **B** | Desk search tickets by `email` | ❌ UNPROCESSABLE_ENTITY | `email` param NOT supported on Desk tickets endpoint |
| **B2** | Desk search tickets by `contactEmail` | ❌ no_data_key | Also unsupported |
| **C** | Desk PATCH ticket owner | ⏭️ Skipped | Requires working ticket search first |
| **D** | Desk READ (simple GET) | ✅ `hasData=true` | Connection-based read works — returns `"data"` key |
| **E** | Analytics API ping | ✅ `hasData=true` | Connection-based read works |
| **F** | External (non-Zoho) URL | ✅/⏱️ SUCCESS/TIMEOUT | Reachable but 40s timeout limit |
| **G** | **Analytics WRITE to table** | ⚠️ **SILENT FAILURE** | `POST /rows` returned HTTP 200 with `{"status":"failure","summary":"LESS_THAN_MIN_OCCURANCE"}` — requires URL-encoded `CONFIG` parameter. API does NOT throw on HTTP 200 failure bodies. **Must always check response content, not just exception state.** |
| **I** | `for each` loop on List | ✅ WORKS | Standard iteration confirmed |
| **J** | Desk search by `status=Open` | ✅ `hasData=true count=3` | Status filter works, returns `"data"` key |
| **K** | `invokeUrl` with `parameters:` Map | ✅ SUCCESS | GET params as Map syntax works |

### Key Architectural Finding — Ticket Discovery

The Desk API tickets endpoint (`/api/v1/tickets`) does NOT support `email` or `contactEmail` as query parameters. This means we cannot search for the auto-created ticket by visitor email directly from SalesIQ Deluge. Alternative discovery paths:
- Search contacts by email (`/api/v1/contacts?email=X`) — requires `Desk.contacts.READ` scope on the connection
- Search tickets by `departmentId` + time window — iterate results in Deluge
- **Analytics pivot (recommended):** Write probe data to `salesiq_probe_log` via Analytics v2 REST, reconcile via lightweight Flow

### Note on `zoho.reports.createRow`

❌ Fails at validation in SalesIQ. Connection references are checked at validation time, not runtime. Try/catch does NOT bypass this. Connections from Desk/Flow do not carry over — must be configured in SalesIQ separately.

### Known Differences From Other Environments

| Feature | Flow | Desk | SalesIQ |
|---------|------|------|---------|
| Event object | `trigger` | None (hook → function params) | `entity` |
| `now()` | ✅ Works | ✅ Works | ❌ Fails |
| `keySet()` | ✅ Works | ✅ Works | ❌ Fails (all Maps, not just entity) |
| `getType()` | ✅ Works | ✅ Works | ❌ Fails |
| `info` | ❌ Forbidden | ✅ Allowed | ✅ Allowed |
| `invokeUrl` | ✅ Works | ✅ Works | ✅ Works (Test A) |
| `containsKey()` | ✅ Works | ✅ Works | ✅ Works on invokeUrl response Maps (Tests B, D, J) |
| Connections | ✅ Required | ✅ Required | ✅ Works (literal string in connection:) |
| `connection:` value type | STRING variable ok | STRING variable ok | ❌ Must be literal CONNECTION LINKNAME |
| `zoho.authtoken` | ✅ Available | ✅ Available | ❌ UNAVAILABLE — parse error, use Connections |
| `zoho.encryption.urlEncode` | ✅ Works | ❌ Not available | ❌ Parse error — `zoho` is not a general namespace |
| `zoho.desk.*` | ✅ Works | ✅ Works | ❌ Parse error — tasks not imported |
| `zoho.reports.createRow` | ✅ Works | ⚠️ **Silent no-op** — returns success but does NOT write data (confirmed v11, Jul 2026). Use `invokeUrl` + PUT /rows + CONFIG instead. | ❌ Connection validation fails |
| Analytics v2 REST write | ✅ Works (via invokeUrl) | ❌ Not available | ⚠️ `POST /rows` requires URL-encoded `CONFIG` param (`zoho.encryption.urlEncode` blocked). Try `POST /import` endpoint. HTTP 200 trap — check response body. |
| `for each` loop | ✅ Works | ✅ Works | ✅ Works (Test I) |
| External API (non-Zoho) | ✅ Works | ✅ Works | ✅ Works (40s timeout, Test F) |
| `parameters:` Map in invokeUrl | ✅ Works | ✅ Works | ✅ Works (Test K) |

---

## §S.3 — Architecture Decision: SalesIQ for Capture, Desk for Mutation

**Critical lesson from v5.0–v5.9 probes:** SalesIQ Deluge and Desk Deluge are fundamentally different environments. Trying to do ticket mutation in SalesIQ is the wrong architecture.

### The Correct Split

| Responsibility | Run In | Why |
|----------------|--------|-----|
| Capture conversation data | **SalesIQ Workflow → Analytics** | SalesIQ has the `entity` object with conversation data. Analytics v2 write is possible (§S.6). |
| Read/update Desk tickets | **Desk Workflow Rule → Desk Function** | Desk has `zoho.desk.*` integration tasks, `invokeUrl` + PUT /rows for Analytics writes, and no regex pre-validator. |
| Cross-system orchestration | **Zoho Flow** | Flow has both Desk and Analytics v2 access, plus `zoho.encryption.urlEncode`. |

### Why SalesIQ Cannot Do Ticket Mutation

| Attempted | Result | Root Cause |
|-----------|--------|------------|
| Search ticket by email | ❌ UNPROCESSABLE_ENTITY | Desk tickets endpoint doesn't accept `email` param |
| Search ticket by `contactEmail` | ❌ no_data_key | Also unsupported |
| Read desk ticket ID from `entity` | ❌ NOT in payload | The Desk Ticket ID is not included in `conversation.completed` |
| `zoho.desk.*` functions | ❌ Parse error | Desk integration tasks not imported in SalesIQ Deluge |
| `invokeUrl` to Desk PATCH | Skipped | No way to discover the right ticket ID |

### Why Desk Can Do Ticket Mutation

| Capability | Available in Desk |
|------------|------------------|
| `zoho.desk.getRecordById()` | ✅ |
| `zoho.desk.searchRecords()` | ✅ |
| `zoho.desk.update()` | ✅ |
| `zoho.reports.createRow()` | ⚠️ Silent no-op (use invokeUrl + PUT /rows) |
| `keySet()` | ✅ |
| Workflow Rule - On Update of Ticket | ✅ — fires on every ticket state change |
| Channel-based filters in workflow rules | ✅ — can filter to SalesIQ channels only |

### Recommended Pipeline

```
SalesIQ                          Desk                           Analytics
conversation.completed           On Update of Ticket            Reporting
       │                              │                            │
       ▼                              ▼                            ▼
[SalesIQ Probe]                [Desk Custom Function]        [Query Tables]
Capture entity data            Read full ticket state        Dashboards
Write to Analytics (v2)        Apply business logic          Management view
       │                       Update ticket owner
       │                              │
       └──────────────────────────────┘
         (no direct link needed)
```

**Key insight:** The SalesIQ → Desk integration creates the ticket. We don't need to link them from SalesIQ. Instead, a Desk workflow rule catches the ticket update, reads the full ticket state from `getRecordById`, and applies any needed corrections.

---

## §S.4 — Debug Strategy & Workflow Setup

### Debug Strategy for SalesIQ

Since `keySet()` is unavailable and the `entity` structure is undocumented, use a two-phase approach:

**Phase 1 — Discovery (current):**
Deploy a probe that dumps `entity.toString()` via `info` statements, then inspect the execution log to see all available fields.

**Phase 2 — Hard-coded access (after discovery):**
Replace the dump with targeted `.get()` calls using the discovered key names. All `.get()` returns must use `ifnull(..., "")` guards since nullable fields are common.

### Workflow Setup — Action Platform Selection (Critical)

When creating a SalesIQ Data Workflow, the **Action Platform** step offers three choices:
- **Webhooks** — POST event data to external URL
- **SQL Query** — accepts only SELECT SQL statements ❌
- **Deluge Script** — runs Deluge code ✅

**The trap:** SQL Query and Deluge Script look similar in the UI. Selecting SQL Query and pasting Deluge code gives: `"Only SELECT query statements (SQL) are allowed"`.

**Always select Deluge Script** under Actions when running custom logic.

### Return Type Constraint (Critical)

**Discovery:** SalesIQ Workflow Deluge scripts MUST return a **Map** value. Returning a string (even via `return "v1"`) causes:

```
Unable to cast the 'X' value into a 'MAP' value because the input is in an invalid format
```

**Fix:** Always wrap return values in a Map:
```deluge
result = Map();
result.put("status", "ok");
result.put("version", probeVersion);
return result;
```

Note: Despite the return-type error, the script **does execute** — all `info` statements above the `return` line fire successfully and appear in execution logs. The execution is marked "failed" only because of the invalid return type.

### CRITICAL: Parse-Time vs Runtime Errors

**Discovery:** In SalesIQ Deluge, if the parser does not recognize a variable/namespace (e.g., `zoho.authtoken`), it fails at **parse time**, before any code executes. This means:

- ❌ **try/catch does NOT help** — the script never runs
- ❌ The error is caught at validation, not execution
- ✅ Only `zoho.currenttime` is recognized as a `zoho.*` built-in
- ✅ All other `zoho.*` references must be treated as **parse errors**, not runtime errors

**Rule of thumb:** If a Zoho Deluge feature is not specifically documented as available in SalesIQ, assume it causes a parse error. Trial-and-error via `info` + try/catch only works for features where `zoho` is at least recognized by the parser.

**Current known-safe constructs:** `zoho.currenttime`, standard Deluge (variables, if/else, loops, Map, List, try/catch, `ifnull`, `entity.get`, `info`, string operations).

**Anything outside that set must be deployed and tested — failure is parse-time, not catchable.**

---

## §S.5 — Connection & Null Safety

### invokeUrl Connection Parameter — Literal String Required

**Discovery (2026-07-05):** The `connection` parameter of `invokeUrl` in SalesIQ Deluge requires a **literal CONNECTION LINKNAME** type, not a STRING variable.

**Fails:**
```deluge
connName = "desk_connection";
resp = invokeUrl [ url: url type: GET connection: connName ];
// Error: "'connections' value is of type 'STRING' which does not match the required data type 'CONNECTION LINKNAME'"
```

**Works:**
```deluge
resp = invokeUrl [ url: url type: GET connection: "desk_connection" ];
```

**Rule:** Always hard-code the connection name as a literal string in the `connection:` parameter. Variables that resolve to strings are not accepted.

### Null Safety Pattern (SalesIQ-Specific)

Every chained `.get()` must be null-guarded at EVERY level — both the sub-object AND the field within it:
```deluge
// WRONG — throws if either sub-object or field is null:
// visitorName = entity.get("visitor").get("name").toString();

// CORRECT — guard at both levels:
visitorMap = entity.get("visitor");
visitorName = "";
if(visitorMap != null) {
    visitorName = ifnull(visitorMap.get("name"), "").toString();
}
```

---

## §S.6 — replaceAll Regex & Analytics Write

### SalesIQ `replaceAll` Has a Regex Pre-Validator

**Critical Discovery (2026-07-06–07, v5.5–v5.7 probes):** SalesIQ's Deluge engine has a **regex pre-validator** that validates `replaceAll` patterns at **parse time** (not runtime). Invalid patterns crash the entire script — try/catch does NOT help. The pre-validator is stricter than Java's `Pattern.compile()`.

**Pre-validator rejects:**
| Pattern | Error | Why |
|---------|-------|-----|
| `replaceAll("{", ...)` | `{ is not a valid regular expression` | `{` is literal, not regex — use char class |
| `replaceAll("\\{", ...)` | `\{ is not a valid regex` | `\{` not a standard regex escape |
| `replaceAll("\\[", ...)` | `\[ is not a valid regular expression` | `\[` not a standard regex escape |
| `replaceAll("[[]", ...)` | Same error | Pre-validator rejects `[` inside char class |

**Pre-validator accepts (empirically verified):**
- `replaceAll("\"", ...)` — `\"` is standard regex escape for `"`
- `replaceAll(":", ...)` — literal, no regex conflict
- `replaceAll(",", ...)` — literal
- `replaceAll(" ", ...)` — literal
- `replaceAll("'", ...)` — literal
- `replaceAll("1", ...)`, `replaceAll("=", ...)` — literal

**Character class workaround for braces (VERIFIED to work in v5.7–v5.9):**
```deluge
// SAFE: [{] — character class containing literal {
encStr = encStr.replaceAll("[{]", "%7B");
encStr = encStr.replaceAll("[}]", "%7D");
```

**Characters that CANNOT be matched (no workaround found):**
- `[`, `]`, `(`, `)`, `*`, `+`, `?`, `|`, `$`, `^`, `.`, `\` — the pre-validator rejects escape sequences AND character classes containing these
- **Workaround for Analytics URL encoding:** Leave these characters unencoded in the URL — most Zoho APIs accept them as-is

> **⚠ CRITICAL — Square brackets `[` and `]` cannot be encoded in SalesIQ.** The standard JSON format of a Deluge map string (`cfg.toString()`) uses square brackets for arrays. If a criteria string or `selectedColumns` list contains `[` or `]`, the URL serialization will break or silently drop characters. **Always pass query criteria completely flat** — use pure alphanumeric column keys and avoid JSON arrays entirely inside the SalesIQ layer. If you need to pass a list of columns, encode them as a comma-separated string inside the CONFIG map, not as a JSON array.

**Final rule for URL encoding CONFIG in SalesIQ:**
```deluge
// ONLY these replaceAll calls are safe to use:
encStr = encStr.replaceAll("\"", "%22");
encStr = encStr.replaceAll(":", "%3A");
encStr = encStr.replaceAll(",", "%2C");
encStr = encStr.replaceAll(" ", "%20");
encStr = encStr.replaceAll("'", "%27");
// Do NOT attempt to encode { } [ ] ( ) — leave them raw
```
The remaining characters (`{`, `}`, alphanumerics) work without encoding in Zoho Analytics URLs.

### Analytics v2 Write from SalesIQ — Definitive Working Pattern

**Discovered across v5.4–v5.9 (Jul 2026).** Writing to Analytics from SalesIQ Deluge is possible but required solving 4 separate errors in sequence:

| Attempt | Version | Error | Fix |
|---------|---------|-------|-----|
| `POST /rows` with `{"DATA":[...]}` body | v5.3 | `LESS_THAN_MIN_OCCURANCE` | Analytics v2 requires `CONFIG` URL param |
| `POST /import` | v5.4 | `URL_RULE_NOT_CONFIGURED` (8525) | `/import` endpoint blocked for Connection |
| Missing `ZANALYTICS-ORGID` header | v5.7 | `ORGID_NOT_PRESENT_IN_THE_HEADER` (8083) | Add `"ZANALYTICS-ORGID": <orgId>` header |
| Decimal in NUMBER column | v5.8 | `DATA_VALIDATION_ERROR` (7507) | Use `.toLong()` for numeric values |

**Working approach:**

```deluge
// ── Static Constants ──
staticAnalyticsDomain = "https://analyticsapi.zoho.com";
staticWsId = "YOUR_WORKSPACE_ID";
staticViewId = "YOUR_VIEW_ID";
staticAnalyticsOrgId = "YOUR_ANALYTICS_ORG_ID";

// ── Build CONFIG ──
cfg = Map();
cols = Map();
cols.put("column1", value1);
cols.put("column2", value2);
cfg.put("columns", cols);
cfg.put("addIfNotExist", true);
cfg.put("criteria", "1=2");  // Always false → forces INSERT

// ── URL-encode CONFIG (safe chars only) ──
cfgStr = cfg.toString();
encStr = cfgStr;
encStr = encStr.replaceAll("\"", "%22");
encStr = encStr.replaceAll(":", "%3A");
encStr = encStr.replaceAll(",", "%2C");
encStr = encStr.replaceAll(" ", "%20");
encStr = encStr.replaceAll("'", "%27");
// { } left raw — Zoho APIs accept them

// ── Execute PUT ──
putUrl = staticAnalyticsDomain + "/restapi/v2/workspaces/" + staticWsId
         + "/views/" + staticViewId + "/rows?CONFIG=" + encStr;

headers = Map();
headers.put("ZANALYTICS-ORGID", staticAnalyticsOrgId);
headers.put("Content-Type", "application/json");

resp = invokeUrl
[
    url: putUrl
    type: PUT
    headers: headers
    connection: "analytics_connection"
];

// ⚠ ALWAYS check response body — HTTP 200 with failure body is a trap
respStr = resp.toString();
if(respStr.contains("\"status\":\"failure\""))
{
    info "Analytics Write Failed: " + respStr;
}
// Also parse as Map for structured access (keySet() forbidden in SalesIQ — use containsKey())
try
{
    respMap = resp.toMap();
    if(respMap.containsKey("status") && respMap.get("status").equals("failure"))
    {
        info "Analytics Error: " + ifnull(respMap.get("summary"), "unknown");
    }
}
catch (e_resp) { dummy = "caught"; }
```

### URL Encoding vs zoho.encryption.urlEncode

| Environment | Method | Availability |
|-------------|--------|-------------|
| Flow | `zoho.encryption.urlEncode()` | ✅ Works |
| Desk | `zoho.encryption.urlEncode()` | ❌ Not available |
| SalesIQ | `zoho.encryption.urlEncode()` | ❌ Parse error — `zoho` not a namespace |

SalesIQ's `zoho` built-in only exposes `zoho.currenttime`. All other `zoho.*` references cause parse-time failures.

---

## §S.7 — Reference Data

### Known `ended_by` Values (Empirically Verified)

| Value | Description | Discovered |
|-------|-------------|-----------|
| `agent` | Operator ended chat | v5.3 |
| `visitor` | Visitor ended chat | v5.3 |
| `system` | System/idle timeout ended chat | v5.3 |
| `bot` | Bot ended chat | v5.4 |
| `scheduler` | Scheduled/automated closure | v5.9 |

Note: The raw `ended_by` uses lowercase (`"agent"`, `"bot"`). Map to display labels in your script.

### Desk Custom Function Trigger Patterns (Desk-Specific — Included for SalesIQ→Desk Pipeline Context)

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

### SalesIQ → Desk Hook Timing (Empirically Verified Jul 2026)

**Discovery:** When SalesIQ creates a Desk ticket, the workflow rule fires at ticket creation **START** — before the ticket ID is written back to the SalesIQ Conversation record.

**Event sequence:**
```
1. Chat ends in SalesIQ
2. SalesIQ starts creating ticket in Desk
3. ← Workflow rule fires HERE (ticket creation START)
4. ← At this point, SalesIQ Conversation has NO "Zoho Desk Ticket ID" yet
5. Ticket creation completes in Desk
6. SalesIQ writes "Zoho Desk Ticket ID" back to Conversation record
```

**Impact:** Any custom function triggered at step 3 cannot read the ticket ID from the SalesIQ Conversation — it doesn't exist yet. The ticket ID is only available after step 5.

**Practical consequence:** The `desk_ticket_id` column in `salesiq_probe_log` is always empty when written at hook time. This is NOT a bug — it's a timing constraint. The ticket ID gets populated by Zoho automatically after ticket creation completes.

**Verified:** 163 conversations in probe log, all with empty `desk_ticket_id`. 1,284 tickets in 3-day export, all correctly created. The SalesIQ → Desk lifecycle works as designed.

**Rule:** Do not attempt to read the Desk ticket ID from the SalesIQ Conversation record inside a ticket-creation hook. It will always be null.

---

## §S.8 — Probe Iteration Summary

| Version | Key Change | Result |
|---------|-----------|--------|
| v1 | Initial probe — `now()` fails | ❌ Parse error |
| v2 | Fixed `now()` → `zoho.currenttime`, null safety | ❌ Connection not found |
| v3 | Stripped Analytics write, `info` only | ❌ Wrong action type (SQL Query) |
| v4 | Recreated with Deluge Script, corrected entity fields, returns Map | ✅ **Live** |
| v5.0 | Desk + Analytics Connection setup | ✅ Connections configured |
| v5.1–5.3 | 12 capability tests across both Connections | ⚠️ Full capability map |
| v5.4 | `POST /import` for Analytics write | ❌ `URL_RULE_NOT_CONFIGURED` |
| v5.5 | `POST /rows` + CONFIG with `replaceAll` regex | ❌ `\[` pre-validator error |
| v5.6 | Character class `[[]` for bracket | ❌ Same pre-validator error |
| v5.7 | Zero-regex encoding, criteria `"1=2"` | ✅ No regex errors; ❌ `ORGID_NOT_PRESENT` |
| v5.8 | Added `ZANALYTICS-ORGID` header | ✅ Header sent; ❌ Decimal in number column |
| v5.9 | `.toLong()` for duration, `"scheduler"` label | ✅ **Row written to Analytics** |
