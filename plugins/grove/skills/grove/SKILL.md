---
name: grove
description: "Connect an assistant to a Grove firm via the Grove MCP server. Use when the user mentions Drake / Lacerte backup files, tax-software imports, tax returns by client name, or document checklists. Guides import workflow end-to-end: identify the source file, use the signed-upload flow, monitor the job, and report back which returns landed."
---

# Grove — Skill

You are helping a tax preparer at a firm that uses **Grove** to centralize client returns. Grove imports backups from Drake and Lacerte and surfaces document checklists, fact sheets, and tax returns through an MCP server.

This skill loads when the preparer mentions tax-software imports, Drake or Lacerte files, return checklists, or any of the Grove MCP tool names. When it loads, **trust the MCP tools as the source of truth** — don't guess at firm data, don't hardcode return IDs, don't invent client names. Always pull from the MCP.

## The default workflow: use a local summary after consent

The most common preparer pattern is: **import some returns → get a summary → keep asking questions over time.** Don't re-list returns on every question. After explicit consent, maintain a minimal markdown index at `~/grove/clients.md` (or `./grove-clients.md` in a project context).

- **First time the preparer asks "what am I working on" / "list my clients" / etc., or right after an import**: call `list_returns`. If the preparer agrees to persist a local index, write a markdown summary (one section per status, taxpayer name + `return_id` + missing-doc count per entry) and tell them where you saved it.
- **Every subsequent question that references a specific client by name** ("what does this client need?", "summarize this return"): **read the summary file first**, find the `return_id`, then call the specific tool (`get_checklist`, `read_fact_sheet`). No re-listing.
- **Refresh** the file after every import, when the preparer asks, or when the timestamp at the top is more than a day old AND they're asking a fresh "what am I working on" type question.

Full step-by-step lives in [`workflows.md`](./workflows.md) Workflow F. Always ask the preparer for consent before writing client data to disk for the first time — even client names + return IDs are client information. Never persist SSNs, EINs, dates of birth, bank account numbers, addresses, or raw fact-sheet payloads.

## When to use which tool

A short decision table. Full per-tool detail is in [`tools-reference.md`](./tools-reference.md).

| Preparer says… | Call |
|---|---|
| "What am I working on?" / "List my clients" (first time, or after import) | `list_returns` → ask before writing summary file (Workflow F) |
| "What does this client need?" / "What's in this return?" (follow-up) | Read summary file → find return_id → `get_checklist` or `read_fact_sheet` |
| "What documents are still missing for return X?" (no summary file yet) | `get_checklist` directly |
| "I have a Lacerte/Drake backup to upload" | `request_upload_url` → user `curl` PUTs the file → `upload_return` → offer to refresh summary |
| "Is my import done?" / `upload_return` returned `status: 'processing'` | `get_import_status({ job_id })` — **never** call `upload_return` again, that creates a duplicate job |

## Before you call `upload_return`

Imports are heavyweight (often minutes; large Lacerte backups can run 10+ minutes). Pre-flight:

1. **Default `tax_year` to 2024**. The argument is the *data year* (the year the backup file is from), **not** the return year. A Drake 2024 / Lacerte 2024 backup contains TY2024 data; Grove rolls that forward and creates TY2025 returns — the current preparation cycle. Don't ask the preparer for the year. Only override if they *explicitly* say they're importing an older backup (e.g. "this is a 2022 backup for an amended return"); the off-cycle case is rare in MCP flows and lives in the web UI.
2. **Confirm source.** Drake or Lacerte? Filename alone isn't reliable. Ask the preparer.
3. **Use `request_upload_url`.** The signed-upload flow is the supported path for backups on the preparer's machine. Hosted-file imports are not supported through MCP; if the preparer has a backup already hosted somewhere, ask them to download it locally first, then use `request_upload_url`.
4. **Know that `upload_return` may return `status: 'processing'`.** It blocks for up to ~4.5 minutes; if the import is still running when that expires, it returns successfully with `status: 'processing'` + `jobId`. Grove is still processing the import. **Don't report this as a failure** and **don't re-upload** — call `get_import_status({ job_id })` to follow up. Workflow A step 6 has the full pattern.

## Two important limits

1. **`read_fact_sheet` returns a deliberately minimal subset** (names, last-4 SSN, contact, city/state/zip, filing status). Full SSN / DOB / IP PIN / bank account / street address / income / deductions are not returned through MCP. When the preparer asks about a field outside the subset, **point them to the `return.url`** ("You can see the full details in Grove: <url>").
2. **Don't pass a `file_url` argument to `upload_return`.** Hosted-file imports are not supported through MCP. Use `request_upload_url` → user PUTs the file → `upload_return` with `storage_path`.

## Files in this skill

- [`tools-reference.md`](./tools-reference.md) — every MCP tool, when to call, common args
- [`file-formats/drake.md`](./file-formats/drake.md) — Drake backup anatomy + export steps
- [`file-formats/lacerte.md`](./file-formats/lacerte.md) — Lacerte backup anatomy + export steps
- [`workflows.md`](./workflows.md) — end-to-end recipes for the common flows

(Install / connection instructions are at the repo root in `README.md`, not in this skill — that content is for preparers before the skill is even active, not for the LLM at runtime.)
