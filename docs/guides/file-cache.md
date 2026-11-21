# Local File Cache

BDrive can keep a copy of your files locally so a file that is requested again
is not downloaded from Telegram a second time. It does not matter which client
asks: the web UI, the API, rclone, or thumbnail generation all share the same
cache.

The cache is **off by default**.

## Configuration

```toml
[files.cache]
enabled = true
memory-max-mb = 2048        # total memory budget
memory-max-file-mb = 256    # largest file the memory tier holds
memory-ttl-minutes = 60     # how long an unused memory entry lives
disk-enabled = true
disk-path = "/var/cache/teldrive"
disk-max-mb = 20480         # total disk budget
disk-ttl-minutes = 1440     # how long an unused disk entry lives
```

The same options exist in `config.yml` under `files.cache`.

## How it works

- Files are stored as whole entries, keyed by file ID, and hold the decrypted
  bytes - so decryption happens once per file rather than once per request.
- Small files (up to `memory-max-file-mb`) go to the memory tier, everything
  larger goes to the disk tier.
- Each tier evicts the least recently used entry when its budget is exceeded,
  and drops entries that have not been read within their TTL.
- A file that is only partially downloaded (a client that disconnects early) is
  never stored, so a truncated file cannot be served later.
- The disk tier is a plain directory. Nothing is mounted automatically: point
  `disk-path` at a bind mount to keep the data off the container's writable
  layer, or leave it as-is to use ephemeral container storage (the cache is then
  cold again after a container recreate).
- Deleting or renaming a file drops its cached entry.

## Cloudflare

Only responses whose credentials travel in the URL (a share link's `hash` or an
ephemeral `token` parameter) are marked as edge-cacheable. Everything else under
`/api` is sent as `private, no-store`, because Cloudflare's cache key ignores
`Cookie` and `Authorization` - caching a cookie-authenticated response would let
anyone who guesses the path fetch it.

If you serve BDrive through Cloudflare, the practical consequence is:

- public share links and tokenised download URLs are cached at the edge for a day
- your session, settings and API responses are never cached by Cloudflare

> [!NOTE]
> A cached response lives until its TTL even if the token behind it expires. Use
> short-lived share links for anything sensitive; deleting the file is what
> actually revokes access.
