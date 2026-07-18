# File Deduplication

The `deduplicate` command finds files that share identical content and links duplicates back to a single canonical copy, so your Telegram storage isn't holding multiple copies of the same bytes.

## What Does the Command Do?

BDrive can compute a BLAKE3 content hash for files as they're uploaded (see [How Hashes Are Computed](#how-hashes-are-computed) below). The `deduplicate` command uses those hashes to:

1. **Group files by hash, per user**: For each user, files sharing the same `hash` are grouped together. Only `active` (non-deleted) and non-encrypted files are considered — encrypted files are always skipped because their ciphertext isn't comparable across independent uploads, even when the plaintext is identical. Deduplication never crosses user boundaries; a file will only ever be linked to another file owned by the same user.
2. **Link duplicates to a canonical copy**: Within each group, the oldest active file is treated as the canonical copy. Every other file in the group has its `referencedFileId` set to point at the canonical file's ID. Duplicate files don't have their own Telegram parts removed — they simply become references; the canonical file's parts are what a reference eventually reads through.
3. **Backfill missing hashes** (`--backfill`): Older files created before hashing existed (or created through a path that skipped hashing) have `hash = NULL` and are otherwise invisible to the grouping pass. With `--backfill`, the command first finds every such file for the target user(s), re-reads its content back from Telegram, and computes a hash for it. Backfilled files are then folded into the same run's grouping pass alongside files that already had a hash, so newly-backfilled files can be deduplicated in the very same run.
4. **Report a summary**: after processing, the command prints counts of processed users, duplicate groups found, files linked, hashes backfilled, and files skipped.

Nothing here uploads or re-uploads file content to Telegram — backfilling only *reads* existing parts back to compute a hash, and linking only updates database rows.

## Command Options

- `--dry-run`
  Compute and report what *would* change without writing anything to the database. Duplicate groups, link counts, and (if combined with `--backfill`) backfilled hashes are all computed in-memory and included in the summary, but no `hash` or `referencedFileId` value is persisted.

- `--user <name>`
  Restrict the run to a single Telegram username. If omitted, the command prompts interactively, letting you pick a specific user or choose to run against all users.

- `--backfill`
  Before grouping files by hash, find files belonging to the target user(s) with `hash IS NULL`, compute a hash for each from their existing Telegram parts, and persist it (unless `--dry-run` is also set). These backfilled files are then included in the same run's duplicate-grouping pass.

- `-c, --config string`
  Specify a custom config file path (default is `$HOME/.teldrive/config.toml`).

- `-h, --help`
  Display help information for the `deduplicate` command.

## How Hashes Are Computed

A file's `hash` is a BLAKE3 tree hash of its content. It can end up populated through any of four paths:

1. **Chunked upload** (`POST /uploads/{id}` followed by file creation): as bytes stream through the upload endpoint, BLAKE3 block hashes are computed over 16MB blocks. This is controlled by the `hashing` query parameter on the upload request, but that parameter **defaults to `true`** server-side — so hashing happens transparently even for clients that never set it explicitly. When the file record is created, the per-part block hashes are concatenated into the final tree hash.
2. **Zero-length files**: the hash of empty content is computed directly at creation time; no data needs to round-trip through Telegram.
3. **Direct-parts creation**: a file can also be created by supplying a `parts` array directly instead of going through the upload-and-hash flow above. In this case the server best-effort computes a hash immediately after creation by reading the file's just-uploaded parts back from Telegram. This is a real, API-exposed creation path (not just a test fixture), so files created this way are hashed too — but because it requires an extra round-trip read of the content, a failure here is logged and swallowed rather than failing the file creation.
4. **`--backfill` (this command)**: for files that predate hashing entirely, or that fell through path 3's best-effort attempt, `--backfill` sweeps them up by reading their content back from Telegram and computing the hash after the fact.

## Web UI (Settings → Deduplication)

The same retroactive deduplication is available from the web UI under **Settings → Deduplication**, so you don't have to drop to the CLI. The tab has three parts:

- **Stats overview** — a read-only summary of the current dedup state for your account: how many duplicate groups exist and how many files are already linked to a canonical copy (backed by `GET /dedup/stats`).
- **Run deduplication** — toggles for **Dry run** and **Backfill** (mirroring `--dry-run` and `--backfill`), plus a **Run** button. Because a run — especially with backfill — can take minutes while content is re-read from Telegram, runs execute **asynchronously**: starting one creates a job (`POST /dedup/jobs`) and the tab polls its status (`GET /dedup/jobs/{id}`) until it finishes, showing the same summary counts the CLI prints.
- **Per-file duplicates** — from a file's actions menu, **View duplicates** lists other files that share the same content hash (`GET /files/{id}/duplicates`), backed by the same grouping logic the command uses.

::: warning Admin targeting not yet enabled
The CLI can target a specific user or all users. The web UI currently runs deduplication only against **your own files** — the "target another user / all users" controls are present but disabled, pending an admin-authorization mechanism (BDrive has no admin role concept yet). Requests that try to target other users are rejected server-side.
:::

## Behavior Notes and Limitations

- **Copy-on-write**: if a file's content is later modified while it has a `referencedFileId` set, the reference is cleared and the file becomes its own canonical copy. If the new content happens to match a *different* existing file's hash, it's re-linked to that file instead.
- **File copies stay dedup-eligible**: copying a file preserves its `hash`, since the content is identical, so a copy can immediately participate in future deduplication runs.
- **Deleting a canonical file does not relink references**: if a file that other files reference via `referencedFileId` is deleted, BDrive currently only logs that other files still reference it — it does not clean up or relink those references. Files that referenced the deleted file are left with a `referencedFileId` pointing at a row that no longer exists. Keep this in mind before deleting files that may be dedup canonicals; running `deduplicate` again afterward will not repair these stale references on its own.

## Example Usage

Dry run against all users, without touching the database:
```bash
teldrive deduplicate --dry-run
```

Deduplicate a specific user's files:
```bash
teldrive deduplicate --user yourTelegramUsername
```

Run without `--user` to be prompted interactively for a user (or all users):
```bash
teldrive deduplicate
```

Backfill missing hashes for legacy files, then deduplicate:
```bash
teldrive deduplicate --backfill
```

Preview a backfill without persisting anything:
```bash
teldrive deduplicate --backfill --dry-run
```

Using a custom config file:
```bash
teldrive deduplicate -c /path/to/config.toml
```

## Understanding the Summary Output

After a run, `deduplicate` prints a summary with the following fields:

- **Processed Users**: number of users the command examined.
- **Duplicate Groups**: number of distinct content hashes for which more than one active file existed.
- **Total Files Linked**: number of files that had `referencedFileId` set (or, under `--dry-run`, would have been set) to point at a canonical file.
- **Hashes Backfilled**: number of previously `hash = NULL` files for which a hash was computed this run (only present when `--backfill` is used).
- **Skipped Files**: files that could not be processed (for example, content that could no longer be read back from Telegram during backfill).
