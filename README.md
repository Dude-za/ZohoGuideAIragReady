# Zoho Compliance & Deluge Coding Standards Guide (RAG-Ready)

[![License: MIT](https://shields.io)](https://opensource.org)
[![Zoho Ecosystem](https://shields.io)](https://zoho.com)

A production-tested, LLM-optimized engineering standard and reference guide for Zoho Deluge development across **Zoho Flow**, **Zoho Desk**, **Zoho Analytics**, and **SalesIQ**.

This guide is structured as a multi-file system consumed as context by AI Agents, Custom GPTs, and RAG (Retrieval-Augmented Generation) pipelines to eliminate common AI hallucinations, syntax errors, and environment-specific execution bugs.

---

## File Structure

| File | Purpose | When to Load |
|------|---------|-------------|
| `ZohoGuideRoot.md` | Pervasive rules — ownership model, canonical template, API patterns, AI guardrails | **ALWAYS** — for any Zoho work |
| `ZohoFlowGuide.md` | Event hook custom workflows for events like logon and logoff | Flow work only |
| `ZohoDeskGuide.md` | The ticket system | Desk work only |
| `ZohoSalesIQGuide.md` | The live chat system for website widget and social media | SalesIQ work only |
| `ZohoAnalyticsGuide.md` | Custom tables, table queries, dashboards and widgets. Data goes or reflects here | Analytics work only |

**Loading rule:** Always load `ZohoGuideRoot.md` first. Then load only the environment file(s) relevant to your current task. Do NOT load all environment files — this wastes context on irrelevant constraints.

---

## Why This Guide Exists

Standard Zoho documentation is fragmented across various product silos, and generic LLMs frequently hallucinate code constructs that work in one Zoho app but completely break in another.

This guide consolidates live, empirically verified operational discoveries (collected through mid-2026 testing) that address:
* **Silent Auto-Casting Traps:** How Deluge unpredictably transforms single-element JSON arrays into Maps and numeric-keyed Maps into Lists.
* **Flow vs. Desk Compiler Discrepancies:** Strict restrictions on keywords (`else`, `continue`), formatting (no blank lines before `try` in Desk), and structural constraints.
* **ZQL Workspace Quirks:** Directives for handling workspace-specific SQL dialects, missing functions (like `dateadd()`), and duration parsing quirks.

---

## Advice from the autor 
It's not in the guide to do so, but i instruct my agent to make and keep a txt file for each deluge code module we work on, and a csv file for every table we create. It's also been usefull to use versioning, were old txt file is copied to create new, that is then updated. Using txt files for deluge code allows easy error free copy and paste, as well as giving the agent easy access to the code. Files describing tables do a similar finction. On top of that i also keep a project dossier md file for the agent to make notes on the project like view id's, to do, ect.

--- 

## RAG & AI Agent Ingestion Instructions

To get the absolute best results when using this guide with an AI coder (such as Opencode with Big Pickle Zen, or Claude 3.5 Sonnet, GPT-4o, or an IDE agent like Cursor/Windsurf), inject the raw content directly into the prompt context, or even better, download and save the files locally and have your agent ingest them at every zoho session statup.

### System Prompt / Context Injection Template
Copy and paste this instruction block into your AI tool alongside the `ZohoGuideRoot.md` content:

```text
You are an expert compiler for the Zoho Deluge engine.
Before generating, modifying, or reviewing any Deluge script or Zoho Query Language (ZQL) schema, you must strictly evaluate your output against the constraints detailed in the attached 'Zoho Compliance & Deluge Coding Standards Guide'.

Pay explicit attention to:
1. ZohoGuideRoot §2.1 — Canonical Function Templates & Global Static Mandates.
2. ZohoGuideRoot §3.1 — Environment Constraints quick reference (e.g., 'else' and 'continue' restrictions in Zoho Flow).
3. ZohoGuideRoot §2.7 & §3.4 — Defensive Type Verification for JSON Map/List casting.
4. ZohoDeskGuide §D-A — Desk Parser Formatting rules (no blank lines before try).

If the user request contradicts a constraint verified in this guide, prioritize the guide's empirically verified guardrails over generic web data.
```

---

## Key Topics Covered

### 1. Global Mandates & Standards (Root Guide)
* Canonical code scaffolding template featuring dynamic input validation and structured try/catch responses.
* Safe, robust `invokeurl` patterns for HTTP 4xx/5xx handling.
* Workspace ownership model — why transfer breaks Analytics writes silently.

### 2. Zoho Flow Architecture (Flow Guide)
* Mitigation strategies for transaction-scoped variable persistence failure across separate trigger events.
* String-based caching patterns to permanently bypass rigid internal `<KEYVALUE>` validation errors.
* Dynamic custom workarounds for non-existent Deluge functions like `sortBy()`.

### 3. Zoho Desk Guardrails (Desk Guide)
* Dealing with `VOID` return limitations.
* Documenting parameters and limits for `searchRecords()` filters (e.g., the complete failure of `statusType` vs. the success of `status=Open`).
* Handling pagination restrictions natively without banned `while` loops.

### 4. Zoho Analytics & ZQL (Analytics Guide)
* Project-tested SQL templates for calculating running metrics and retrieving the latest timestamp values via `FETCH FIRST X ROWS ONLY`.
* Navigating display names vs. raw ZQL column names in default Desk data imports.
* Utilizing implicit numeric duration parsing secrets (using the `REPLACE` function to extract total business hours into active seconds).

### 5. Zoho SalesIQ (SalesIQ Guide)
* Workflow Deluge constraints — `entity` object structure, verified/non-working constructs.
* Architecture decision: SalesIQ for capture, Desk for mutation.
* Analytics v2 write from SalesIQ with manual URL encoding (regex pre-validator workaround).

---

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

Contributions, issue reports, and updates regarding new Zoho environment discoveries are always welcome! Feel free to submit a Pull Request.

---

## Thanks:

Special thanks to Leon Kotze of Zoho for forcing me to do this.
