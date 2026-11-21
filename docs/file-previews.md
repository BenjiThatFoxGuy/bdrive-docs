
## Instance Settings

Thumbnail and image resizer configuration is persisted server-side. Admin users can configure the resizer host through the Settings page, and all viewers (including guests browsing shared folders) will use the same configuration.

### API

- `GET /api/settings/thumbnail` — returns the current thumbnail config (public, no auth required)
- `PUT /api/settings/thumbnail` — update the thumbnail config (admin only)

Request/response body:
```json
{
  "resizerHost": "https://resizer.example.com",
  "resizerWidth": 360,
  "resizerQuality": 80
}
```

### Video thumbnail endpoint

Video files have a dedicated thumbnail source endpoint:

- `GET /api/files/thumb/{fileId}?hash=` — returns a JPEG thumbnail for a video file

On the first request, the server generates the thumbnail using ffmpeg and caches
it as a Telegram message in the same channel as the original file. Subsequent
requests serve the cached image. imgproxy fetches this endpoint and resizes the
result for the UI, using the same `resizerHost` / `resizerWidth` / `resizerQuality`
settings as image thumbnails.
