# Grove MCP — Tools Reference

Each tool runs against the authenticated preparer's firm. The firm is resolved from the OAuth session — you don't pass `firm_id` anywhere.

---

## `list_returns`

Use to answer "what am I working on" / "find a named return" / "show me TY24 in progress." Most-recently-updated first, capped at 100.

**Args** (all optional):
- `tax_year` — integer 2000–2100. Filter to a year (e.g. `2024`). Omit for all.
- `status` — Grove status string. Common values: `not-started`, `awaiting-documents`, `documents-received`, `in-progress`, `internal-review`, `ready-for-client`, `client-approved`, `ready-to-file`, `filed`. Omit for all statuses.
- `limit` — 1–100. Defaults to 25.

**Returns** an array of: `{ id, taxpayer_name, tax_year, status, created_at, updated_at, url }`. The `url` field is an absolute clickable link to the return in Grove — hand it to the preparer when they want to "open" a return, rather than just listing its name.

**When to use**: as the *first* step in nearly every preparer prompt that references "a return" without giving an ID. Don't guess return IDs — list, then pick.

---

## `read_fact_sheet`

Returns a **deliberately minimal subset** of the fact sheet for one return — just enough for "who is on this return" / "what's their contact info" / "what's the filing status." Fields outside this subset are **not exposed** through MCP; direct authorized users to the Grove web UI for full return details.

**Args**:
- `return_id` — UUID. Get from `list_returns`, or from `upload_return`'s response after an import.

**Returns** `{ return: { id, taxpayer_name, tax_year }, fact_sheet, updated_at, _note }`.

`fact_sheet` shape (or `null` if no fact sheet has been generated yet):

```
{
  taxpayer: { first_name, middle_name, last_name, ssn_last4 } | null,
  spouse: { first_name, middle_name, last_name, ssn_last4 } | null,
  dependents: [{ first_name, last_name, ssn_last4, relationship }],
  filing_status: 'single' | 'married_filing_jointly' | ... | null,
  address: { city, state, zip } | null,         // no street, no apt
  contact: { phone, email } | null,
}
```

`ssn_last4` is the last 4 digits as a string (`"6789"`) or `null` if absent. There is no field that returns a full SSN.

**When to use**: answer "who's on this return", confirm a client's contact info, identify a return when the preparer says a name without an ID, or draft a follow-up email. For deeper questions — income, deductions, specific line items, full SSN/DOB/bank/IP PIN, marital dates, anything else not in the subset — **point the preparer at `return.url`** ("you can see the full details in Grove: <return.url>"). Authorized preparers can use that link to open the full return in Grove.

**Don't**:
- Try to infer redacted fields. If `ssn_last4` is `"6789"` you don't know the full SSN; do not invent one.
- Promise the preparer you "remember" data not in the response.
- Paste the `_note` field back to them verbatim — it's a hint for you, not user-facing copy.
- Just say "I can't see that" with no next step. Always offer the `return.url` link as the alternative.

---

## `get_checklist`

Returns the document checklist for one return — every document Grove expects the client to provide (W-2s, 1099s, K-1s, schedules) with status (pending / uploaded / completed / not_relevant).

**Args**:
- `return_id` — UUID. Get from `list_returns`, or from `upload_return`'s response after an import.

**Returns** `{ return: { id, taxpayer_name, tax_year }, items: [...] }`. Each item has `title`, `details` (per-instance: employer name, payer name), `category`, `status`, `source`, `required`, `description`, `completed`.

**When to use**:
- Compose follow-up emails that list each missing document clearly
- Cross-firm audits ("which clients are still missing 1095-A?")
- Determine return readiness before scheduling client review

The `details` field is what distinguishes multi-instance docs (three different W-2s, two K-1s) — use it when listing.

---

## `request_upload_url`

Issues a one-shot signed upload URL the preparer (or their script) PUTs the backup file to. This is the **preferred path** for any local file — bytes never go through the MCP server.

**Args**:
- `source` — `'lacerte'` or `'drake'`.
- `filename` — original filename, e.g. `"TY24.zip"` or `"DrakeBackup_20250309.7z"`. Used for import metadata and display.

**Returns** `{ upload_url, storage_path, expires_at, instructions }`. The `instructions` field has a copy-pasteable `curl` example.

**When to use**: first step of every Drake / Lacerte import where the file is on the preparer's computer.

**Lifetime**: URL is valid for 2 hours, single-use.

---

## `upload_return`

Starts the import job. Blocks for up to ~4.5 minutes; most imports finish in this window, but a large Lacerte batch (50+ clients) may not. Returns the created tax return IDs + clickable URLs once the job is done.

**Args**:
- `source` — `'lacerte'` or `'drake'`. Required.
- `tax_year` — integer, the **data year** (the year the backup file's contents are from — NOT the return year). **Pass `2024` by default**: that's a Drake 2024 / Lacerte 2024 backup, which carries TY2024 data, which Grove rolls forward into TY2025 returns — the current preparation cycle. Only use a different value if the preparer explicitly mentions an older backup. The created return's `return_year` defaults to `tax_year + 1` for the roll-forward case.
- `filename` — original filename. Optional; derived from `storage_path` if omitted.
- `storage_path` — what `request_upload_url` returned. Required.

**Returns** `{ jobId, status, returns: [{ return_id, taxpayer_name, url }, ...] }`. `status` is one of:
- `completed` — all clients imported.
- `partially_completed` — some clients imported, others failed. `returns[]` has what made it; failed-client detail lives on the job row in Grove (point the preparer at that page).
- `processing` — the import was still running when the call's ~4.5min polling window expired. `returns[]` is `[]`. **This is not a failure** — Grove is still processing the import. Follow up via `get_import_status({ job_id: jobId })`. Never call `upload_return` again with the same file (would create a duplicate job).

**When to use**: after `request_upload_url` + the preparer has PUT the file.

**Don't** pass a `file_url` argument. Hosted-file imports are not supported through MCP. If a preparer has a backup at a public URL, ask them to download it locally first, then use `request_upload_url`.

---

## `get_import_status`

Follow-up to `upload_return` when it returned `status: 'processing'`. Same response shape as `upload_return`, same ~4.5min polling window — keep calling until you get a terminal status.

**Args**:
- `job_id` — the `jobId` from the prior `upload_return` response. Required.

**Returns** the same `{ jobId, status, returns }` shape as `upload_return`. If the job is still mid-flight at the end of the polling window, you'll get `status: 'processing'` again — that's expected, just call once more. Invalid or inaccessible job IDs return an error.

**When to use**: only after an `upload_return` (or earlier `get_import_status`) returned `status: 'processing'` with a `jobId`. **Don't** call this speculatively or without a `jobId` you obtained from a prior tool call.

**Pattern**: when an import is taking a while, narrate progress to the preparer between calls — *"still importing; checking again in a moment…"* — so they don't think you're stuck. Each call can wait up to ~4.5 minutes; one or two follow-ups handles almost any real import.
