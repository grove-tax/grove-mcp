# Lacerte — Backup File Format

## The year mapping

Lacerte's program version corresponds to the tax year it processes. A Lacerte 2024 install produces backups of TY2024 data; Grove rolls TY2024 data forward to create TY2025 returns (`tax_year: 2024` in `upload_return` → returns with `return_year = 2025`).

Grove imports Lacerte 2024 and earlier. If the preparer is on a newer version, the import may not complete successfully — direct them to Grove support to confirm support status for their version.

## What Grove expects

A **`.zip` archive of a Lacerte backup folder**. Lacerte's native Backup tool writes a *folder* (containing one or more backup files) — it does not produce a single archive. Grove needs that folder wrapped in `.zip`. Max upload size: 100 MB.

## How the preparer exports one

1. **Create a new empty folder** on disk (Desktop is fine). They'll need to find it again in step 4.
2. **Lacerte YYYY → select the clients to import → Client → Backup**
3. Lacerte prompts for a directory — point it at the folder from step 1, click **OK**.
4. Lacerte writes the backup into that folder.
5. **Wrap the folder in a `.zip`** — either the preparer does it (right-click → Compress → Zip File on Windows; equivalent on macOS), or *you* do it on their behalf (see below).
6. Upload the resulting `.zip` to Grove.

## When the preparer hands you the raw folder

If the preparer says "I have the Lacerte backup folder, can you upload it?" — they've skipped step 5. If they want help and you have shell access:

```bash
# From the parent of the folder, replacing <folder> with the actual name
cd "<parent-dir>"
zip -r "<folder>.zip" "<folder>"
```

(On Windows / PowerShell environments: `Compress-Archive -Path "<folder>" -DestinationPath "<folder>.zip"`.)

Then proceed with `request_upload_url` + `curl` PUT + `upload_return` against the resulting `.zip`. Confirm the resulting archive is under 100 MB before uploading.

If you do not have shell access in your current environment, walk the preparer through the manual right-click → Compress flow instead.

## Things to watch for

- **Zipped the wrong thing.** The most common preparer mistake: zipping *one of the files inside* the backup folder, or uploading a loose backup file directly. Grove needs the whole folder wrapped in `.zip` — not a single file from inside it. If you wrap it on the preparer's behalf, you avoid this class of mistake entirely.
- **Wrong Lacerte version.** Confirm the program version if the preparer says "the latest backup" — newer versions than Lacerte 2024 may not be supported yet (see the year mapping note above).
- **Password-protected backups.** Lacerte supports password protection on backups. Grove does not currently accept password-protected archives; if the preparer mentions setting a password, ask them to re-export without one.

## Inside the archive (LLM context only — don't parse from the model)

The archive contains Lacerte's per-client binary backup files plus index metadata. No client-uploaded documents — those flow through a separate Grove path.

Parsing happens server-side inside Grove. Never try to read the bytes from the LLM side.
