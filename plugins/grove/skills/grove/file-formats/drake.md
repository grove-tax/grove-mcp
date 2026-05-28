# Drake — Backup File Format

## The year mapping

Drake versions its program by tax year, and its per-client data files inherit that year:

| Drake program | Per-client file extension | Holds TY data for | Grove rolls forward to create |
|---|---|---|---|
| Drake 2023 | `.D23` | TY2023 | TY2024 returns |
| Drake 2024 | `.D24` | TY2024 | TY2025 returns |
| Drake 2025 | `.D25` | TY2025 | TY2026 returns (next cycle) |

So when the preparer says "I have a Drake 2024 backup" they mean **D24 files**, which are **TY2024 data**, and Grove turns those into **TY2025 returns** (`tax_year: 2024` in `upload_return` → returns created with `return_year = 2025`).

This is the canonical disambiguator if the preparer's wording is ambiguous: **ask which Drake version they ran the export from**, and the file extension will tell you the rest.

## What the preparer actually uploads

A Selective Backup archive. Drake's native output from the Selective Backup tool is `.7z`; Grove also accepts a `.zip` (some preparers re-wrap before uploading). The archive contains the `.D<YY>` per-client files plus Drake's index metadata.

Max size: 100 MB.

## How the preparer exports one

Time: ~2 minutes for a typical book.

1. **Drake YYYY → Tools → File Maintenance → Backup**
2. Pick a save location (Desktop is fine)
3. Click **Selective Backup** at the top of the dialog (not Full Backup)
4. In the left panel, tick **Tax Returns** only — leave Letters / Setup / Macros / Templates unchecked
5. Pick the clients to include (or Select All), click **Backup**
6. Drake writes the archive — they upload that to Grove

Drake's default filename for the archive looks like `DrakeBackup_YYYYMMDD.7z`, but preparers often rename it. Don't rely on the filename to identify the source — rely on the preparer telling you, or on the `.D<YY>` files inside.

## Things to watch for

- **Tax-year vs Drake-version confusion.** Use the version → file-extension table above. If they say "Drake 2024" and you call `upload_return({ tax_year: 2025 })` you'll produce returns for the wrong cycle.
- **Full Backup instead of Selective Backup.** A Full Backup includes program files and templates — much larger and more likely to bust the 100 MB cap. If the file is unusually large, ask whether they used Selective Backup.
- **Password-protected backups.** Drake supports password protection. Grove does not currently accept password-protected backups; ask the preparer to re-export without a password.

## Inside the archive (LLM context only — don't parse from the model)

The archive contains:
- Per-client `.D<YY>` data files (Drake's proprietary binary format)
- Drake's index/metadata files

No client-uploaded documents (W-2 PDFs, 1099s, etc.) — those flow through a separate Grove upload path, not the Drake import.

Parsing happens server-side inside Grove. Never try to inspect the bytes from the LLM side.
