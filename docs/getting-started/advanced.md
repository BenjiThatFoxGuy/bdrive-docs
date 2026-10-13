# Advanced Usage

This guide covers advanced BDrive configuration options to enhance performance and security.

## Multi-Threaded Streams

If you experience buffering issues, enable multi-threaded options to significantly improve streaming performance:

```toml
[tg.stream]
multi-threads = 6
stream-buffers = 20
```
For Rclone use these options:
```
--vfs-read-chunk-size=32M
--vfs-read-chunk-streams=4
--teldrive-threaded-streams=1
```

> [!NOTE] 
> - You may not need this if your telegram dc is close to your location.
> - Keep `multi-threads` value at 8 or lower to prevent excessive CPU usage.
> - You can tweak these params to find the best performance for your connection.Don't set `vfs-read-chunk-size` too high, as it may cause buffering issues.

## File Encryption

Enable file encryption to secure your data:

```toml
[tg.uploads]
encryption-key = "your-key"
```

You can generate a secure random encryption key using the [key generator tool](/docs/getting-started/usage.md#generate-secret-keys) in the usage guide.

> [!NOTE]
> - Add `encrypt_files = true` in your rclone config when enabling encryption
> - Store your encryption key securely - you can't recover files without it
> - BDrive's encryption is more secure than rclone's crypt implementation as it generates a random salt for each file part rather than using the same salt for all files
> - Enabling encryption in BDrive makes the UI fully compatible with encrypted files

## Adding Bot Tokens

Bot tokens are essential for optimal Telegram API interaction. To create and add bot tokens:

1. Open Telegram and search for `@BotFather`
2. Start a chat and type `/newbot`
3. Follow the prompts to set a name and username (must end with `bot`)
4. BotFather will provide your bot token
5. Add 7-8 bot tokens in the BDrive UI Settings for better upload/download speeds

> [!WARNING]
> Bots will be automatically added as admins in your channel when set through the UI. If this fails, add them manually.
> For newly created Telegram sessions, you must wait 20-30 minutes before adding bots to a channel. You'll see a **`FRESH_CHANGE_ADMINS_FORBIDDEN`** error if you try too soon.

## Using a Thumbnail Resizer

BDrive supports on-the-fly image resizing using `imgproxy` for thumbnail viewing:

::: code-group

```yml [docker-compose.yml]
services:
  imgproxy:
    image: darthsim/imgproxy
    container_name: imgproxy
    environment:
      IMGPROXY_ALLOW_ORIGIN: "*"
      IMGPROXY_ENFORCE_WEBP: true
      IMGPROXY_MALLOC: "jemalloc"
    restart: always
    ports:
      - 8000:8080
```
:::

Start the service:
```sh
docker compose up -d
```

For better performance:
- Deploy imgproxy behind Cloudflare or another web server with caching
- Enter the URL of your deployed resizer service in the BDrive UI settings

## Zip Downloads

BDrive bundles multiple files — or a whole folder — into a single archive on the server and streams
it straight to the browser. Nothing is buffered to disk or to memory, so a folder larger than the
server's RAM downloads fine.

This is on by default. Only set these if you want to change that:

```toml
[files]
enable-zip-download = true
zip-max-files = 10000
zip-max-size = 0
zip-max-concurrent = 4
```

- `enable-zip-download` — turn the feature off entirely. The **Download as Zip** action disappears
  from the UI, and both zip endpoints return `403`.
- `zip-max-files` — reject a request that would bundle more than this many files. Set `0` for no
  limit.
- `zip-max-size` — reject a request whose total uncompressed size exceeds this many bytes. Set `0`
  for no limit.
- `zip-max-concurrent` — how many archives may stream at once. Requests beyond this get `503`
  rather than queueing. Set `0` for no limit.

Both limits are checked before the response starts, so an oversized request gets a proper `413`
instead of a download that dies part-way through.

> [!NOTE]
> Already-compressed files (video, images, audio, archives) are stored in the zip rather than
> deflated. Re-compressing them costs CPU and saves nothing.

> [!TIP]
> Share links can be zipped too. If the share owner has no bot tokens configured, BDrive falls back
> to their own Telegram session, so zipping works either way.
