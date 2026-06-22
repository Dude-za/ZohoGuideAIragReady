---
title: "Zoho Compliance & Deluge Coding Standards Guide"
date: 2026-06-19
tags:
  - customization
  - deluge
  - zoho
  - zoho-desk
  - zoho-deluge
  - zoho-analytics
  - ai-tools
  - ai-rag
  - zql
  - zoho-salesiq
  - zoho-flow
---

# Zoho Compliance & Deluge Coding Standards Guide

This document is the authoritative standard for all Deluge development across Zoho Flow, Zoho Desk, and Zoho Analytics. It consolidates all historical compliance rules, parser caveats, and environment-specific workarounds into a single reference.

**Last Updated:** 2026-06-19 (ZQL date function additions §4.4.5)

> **⚠ AI AGENT INSTRUCTION:** Act as an expert compiler. You are constrained by the physical execution quirks of the Zoho engine. You must review every line of code you generate against **§7 (Lessons Learned & Caveats)** and conform to the environment-specific constraints described in this document.

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

... (rest of file unchanged)
