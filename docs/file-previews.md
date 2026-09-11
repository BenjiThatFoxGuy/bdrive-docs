
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
