# Grove — Common Workflows

End-to-end recipes for things preparers actually ask for. These chain MCP tools in the right order so the LLM doesn't have to invent the flow.

---

## A. Import a Drake or Lacerte backup

1. Confirm `source` (`drake` or `lacerte`) with the preparer if not obvious. **Default `tax_year` to 2024**. `tax_year` is the *data year* (the year the backup file is from), not the return year — a 2024 Drake/Lacerte backup carries TY2024 data, which Grove rolls forward into TY2025 returns. Don't ask the preparer about year unless they specifically mention an older backup.
2. Call `request_upload_url({ source, filename })`. Capture `upload_url`, `storage_path`, `instructions`.
3. **Tell the preparer how to upload**: show them the `instructions` field verbatim — it has a copy-pasteable `curl -X PUT --upload-file …` command. Don't paraphrase; the URL is single-use.
4. **Wait** for the preparer to say they've uploaded. Don't proceed without confirmation — `upload_return` will fail or hang if the file isn't there.
5. Call `upload_return({ source, tax_year, storage_path })`. This blocks for up to ~4.5 minutes.
6. Branch on `status`:
   - **`completed`** — happy path. Report the `returns` array; each entry has a clickable `url` the preparer can open in Grove.
   - **`partially_completed`** — some clients made it. Report the returns that landed (same `returns` array), mention that some failed, and point the preparer at the import-jobs page in Grove for failure detail.
   - **`processing`** — the import is taking longer than the call's polling window. `returns[]` is empty here; Grove is still processing the import. Capture the `jobId`. Tell the preparer something like *"This is a big import — Grove is still processing it. I'll check on it in a moment."*, then call `get_import_status({ job_id: jobId })`. Follow up until you get `completed` or `partially_completed` — but **cap it at 3-4 follow-ups (~15-20 min)**: if it's still `processing` after that, the import is likely stuck; tell the preparer to check the import-jobs page in Grove rather than polling forever. **Never** call `upload_return` again with the same file — that would create a duplicate job.
7. **Offer to refresh the local summary** (see Workflow F) once the import finishes. After an import, the new returns aren't in the preparer's working summary file — with consent, regenerate it so follow-up questions by client name have the new returns indexed.

---

## B. Triage what's missing across a firm's open returns

Preparer asks something like *"which clients still need their W-2s?"* or *"who hasn't submitted documents yet?"*

1. `list_returns({ status: 'awaiting-documents' })` (or `'documents-received'`, `'in-progress'`, etc. depending on what they're triaging)
2. For each return, `get_checklist({ return_id })`
3. Filter items by `status: 'pending'` and `required: true`
4. Group/summarize for the preparer: by client, by document type, by what's been outstanding longest.

Be conservative on volume — if there are 40 returns, summarize at the firm level first ("12 clients still missing W-2s; here are the top 3") and offer to drill into one before dumping the full table.

---

## C. Draft a "please send these documents" client email

1. Get the return ID — preferably ask the preparer for the client name, then `list_returns` to find it.
2. `get_checklist({ return_id })`.
3. Filter to pending + required items.
4. Compose an email that names each document by `title` + `details` (e.g. "W-2 from Example Employer Inc." not just "W-2"). The `details` field is exactly what distinguishes multi-instance docs.
5. Show the draft to the preparer; let them tweak before sending. Don't fabricate context the preparer didn't give you (don't invent the client's first name, don't claim deadlines).

---

## D. Answer "what's in this return?"

1. `list_returns()` — find a return whose `taxpayer_name` contains the name the preparer gave. Multiple matches? Ask the preparer which.
2. `read_fact_sheet({ return_id })`.
3. **What you'll actually get back is a minimal subset** — taxpayer / spouse / dependents (names + last-4 SSN + relationship), filing status, city/state/zip, phone/email. Income, deductions, credits, full SSN, DOB, bank, and street address are not returned through MCP. Don't try to guess what's missing.
4. Useful framing of what you CAN say: who's on the return (taxpayer + spouse + dependents), filing status, where they live (city/state), contact info.
5. For income / deductions / "does this return claim a home office?" type questions — and anything else outside the returned subset — **share the `return.url`** ("You can see everything else in Grove: <url>"). Don't just say "I can't see that" without offering the link.

---

## E. Cross-reference clients to documents

Preparer asks *"who's still missing a 1095-A?"* (cross-firm pattern).

1. `list_returns()` — get all returns for the current cycle (filter by `tax_year` if they specify)
2. For each, `get_checklist({ return_id })`
3. Find items where `title === '1095-A'` (or category matches) and `status` is `pending`
4. Return the list of taxpayer names

This is N+1 queries — for a firm with 100+ returns, ask the preparer first if they want to wait or narrow the scope (e.g. only returns in `awaiting-documents`).

---

## F. Generate a working summary file (the "persistent index")

The single highest-leverage workflow. After an import — or any time the preparer says "give me a summary of my returns" / "list my clients" / "what am I working on" — offer to **write a markdown file to disk** that the preparer can read, annotate, and reference across sessions. The agent uses it as a persistent lookup so a follow-up by client name resolves quickly (without re-listing all returns).

**Default location:** `~/grove/clients.md` (stable across sessions; create the `~/grove/` directory if it doesn't exist). If the preparer is in a project workspace and would rather have it there, write to `./grove-clients.md` instead. Ask before the first write and remember the choice for the session.

**Steps:**
1. Call `list_returns({ limit: 100 })` (raise the limit if a single firm has more).
2. For each return, optionally call `get_checklist({ return_id })` to count missing-required documents. If the preparer is impatient or the firm is large, skip checklist counts on first pass and offer to fill them in on demand.
3. Write the file. Suggested shape:

   ```markdown
   # Grove — Client returns

   _Last refreshed: <ISO timestamp>. Firm: <firm name>._

   ## In progress (<count>)

   ### <Taxpayer name> — TY<year>
   - Status: <status>
   - Return: `<return_id>` · [open in Grove](<return.url>)
   - Updated: <relative time>
   - Missing required documents: <count> (<short list, top 3>)

   ## Awaiting documents (<count>)
   …
   ```

   Group sections by status. Keep the `return_id` in backticks so it's easy to extract later (and visible to the preparer for support purposes).

4. Confirm the file path back to the preparer.

**Then, when they follow up** with client-specific questions:

1. **Read the local summary file first** (you have shell/file access — use it). Find the matching taxpayer by name.
2. Extract their `return_id` from the markdown.
3. Call the relevant tool (`get_checklist`, `read_fact_sheet`) with that ID.
4. Answer the specific question.

This pattern means you don't re-`list_returns` on every question — the file is the cached index. **Re-refresh** the file:
- After every import (proactive offer per Workflow A step 7).
- When the preparer says "refresh the summary" or "what's new."
- When the timestamp at the top is more than a day old AND the preparer is asking a "what am I working on" type question.

**Treat the summary file as persisted client data — get consent before writing.** Even client display names and return IDs are client information. Before the first write, ask the preparer something like *"OK to save a summary of your firm's returns at `~/grove/clients.md`? It'll include client names, return IDs, statuses, and pending-doc counts."* Don't assume implicit consent from a request to "list clients" — they may have meant "show me in chat" not "save to disk."

Even with consent, **never write any of the following to disk**: SSNs, EINs, dates of birth, bank account numbers, addresses, raw fact-sheet payloads, or fields outside the documented `read_fact_sheet` subset. If the preparer asks for a fact-sheet-level summary, do that in chat rather than writing it down.

If the preparer declines persistent storage, fall back to listing returns in-chat each session — they're choosing tighter data handling over cross-session convenience.

---

## What to never do

- **Don't try to download the backup file or inspect its bytes from the LLM.** The MCP server's `upload_return` is the only path that processes backups; the format is binary and proprietary.
- **Don't imply the assistant can edit Grove records directly.** The current toolset is read-only for fact sheets and checklists; only `upload_return` mutates state (by creating returns). If the preparer asks to edit a fact sheet or check off a checklist item from Claude, tell them that's a Grove web-UI action today.
- **Don't paste raw fact-sheet payloads back to the preparer in chat.** Summarize only the documented fields returned by `read_fact_sheet`.
