# Zoho Compliance & Deluge Coding Standards Guide (RAG-Ready)

[![License: MIT](https://shields.io)](https://opensource.org)
[![Zoho Ecosystem](https://shields.io)](https://zoho.com)

An production-tested, LLM-optimized engineering standard and reference guide for Zoho Deluge development across **Zoho Flow**, **Zoho Desk**, **Zoho Analytics**, and **SalesIQ**. 

This document was explicitly structured to be consumed as context by AI Agents, Custom GPTs, and RAG (Retrieval-Augmented Generation) pipelines to eliminate common AI hallucinations, syntax errors, and environment-specific execution bugs.

---

## 🚀 Why This Guide Exists

Standard Zoho documentation is fragmented across various product silos, and generic LLMs frequently hallucinate code constructs that work in one Zoho app but completely break in another. 

This repository consolidates live, empirically verified operational discoveries (collected through mid-2026 testing) that address:
* **Silent Auto-Casting Traps:** How Deluge unpredictably transforms single-element JSON arrays into Maps and numeric-keyed Maps into Lists.
* **Flow vs. Desk Compiler Discrepancies:** Strict restrictions on keywords (`else`, `continue`), formatting (no blank lines before `try` in Desk), and structural constraints.
* **ZQL Workspace Quirks:** Directives for handling workspace-specific SQL dialects, missing functions (like `dateadd()`), and duration parsing quirks.

---

## 🤖 RAG & AI Agent Ingestion Instructions

To get the absolute best results when using this guide with an AI coder (such as Claude 3.5 Sonnet, GPT-4o, or an IDE agent like Cursor/Windsurf), inject the raw content directly into the prompt context.

### System Prompt / Context Injection Template
Copy and paste this instruction block into your AI tool alongside the `MyZohoGuide.md` content:

```text
You are an expert compiler for the Zoho Deluge engine. 
Before generating, modifying, or reviewing any Deluge script or Zoho Query Language (ZQL) schema, you must strictly evaluate your output against the constraints detailed in the attached 'Zoho Compliance & Deluge Coding Standards Guide'.

Pay explicit attention to:
1. §1.1 Canonical Function Templates & Global Static Mandates.
2. §2.1 & §9.5 Environment Constraints (e.g., 'else' and 'continue' restrictions in Zoho Flow).
3. §1.6 & §9.4 Defensive Type Verification for JSON Map/List casting.
4. §7.9.1 Desk Parser Formatting rules (no blank lines before try).

If the user request contradicts a constraint verified in this guide, prioritize the guide's empirically verified guardrails over generic web data.
```

---

## 🛠️ Key Topics Covered

### 1. Global Mandates & Standards
* Canonical code scaffolding template featuring dynamic input validation and structured try/catch responses.
* Safe, robust `invokeurl` patterns for HTTP 4xx/5xx handling.

### 2. Zoho Flow Architecture
* Mitigation strategies for transaction-scoped variable persistence failure across separate trigger events.
* String-based caching patterns to permanently bypass rigid internal `<KEYVALUE>` validation errors.
* Dynamic custom workarounds for non-existent Deluge functions like `sortBy()`.

### 3. Zoho Desk Guardrails
* Dealing with `VOID` return limitations.
* Documenting parameters and limits for `searchRecords()` filters (e.g., the complete failure of `statusType` vs. the success of `status=Open`).
* Handling pagination restrictions natively without banned `while` loops.

### 4. Zoho Analytics & ZQL (SQL) Dialects
* Project-tested SQL templates for calculating running metrics and retrieving the latest timestamp values via `FETCH FIRST X ROWS ONLY`.
* Navigating display names vs. raw ZQL column names in default Desk data imports.
* Utilizing implicit numeric duration parsing secrets (using the `REPLACE` function to extract total business hours into active seconds).

---

## 📜 License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details. 

Contributions, issue reports, and updates regarding new Zoho environment discoveries are always welcome! Feel free to submit a Pull Request.
