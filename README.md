# Gesture Changer API — Integration Guide

> **Version:** 2.0.0
> **Base URL:** `https://onlygen.mvt-soft.work`
> **Updated:** 2026-03-29

## Authentication

All endpoints (except `/health`) require an API key in the `X-API-Key` header.

Keys are issued via the Telegram @Jewketl and have the prefix `gc_`.

```
X-API-Key: gc_your_api_key_here
```

**Error responses:**
- `401` — missing, invalid, or expired key
- `403` — key has been revoked

---

## Rate Limits

- Per key: **10 requests/minute** (configurable)
- Global: **100 requests/minute**
- Exceeded: `429 Too Many Requests` with `retry_after` field

---

## Engines

The API supports two generation engines powered by Seedream:

| Engine | ID | Description |
|--------|----|-------------|
| **Seedream 5.0 Lite** | `seedream5` | Lightweight gesture replacement |
| **Seedream 4.5 Edit** | `seedream4` | Precise gesture replacement |

Default engine: `seedream5`.

Both engines accept preset gestures, custom reference photos, custom text prompts, or a combination of reference + prompt.

---

## Endpoints

### 1. Health Check

```
GET /health
```

No authentication required. Returns `{"status": "ok"}`.

---

### 2. List Presets

```
GET /api/v1/presets
```

Returns all gesture presets grouped by category.

**Response:**

```json
{
  "presets": [
    {
      "id": "v_sign",
      "name": "Peace / V Sign",
      "emoji": "✌️",
      "category": "popular",
      "preview_url": "https://onlygen.mvt-soft.work/static/presets/v_sign.jpg"
    }
  ],
  "categories": [
    {"id": "popular", "label": "Popular", "count": 8}
  ],
  "total": 23,
  "custom_gesture_supported": true
}
```

---

### 3. Generate (Async)

```
POST /api/v1/generate
Content-Type: multipart/form-data
```

Submits a generation task and returns immediately.

**Form fields:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `user_photo` | file | Yes | Source photo (JPEG/PNG, max 10 MB) |
| `gesture_id` | string | Conditional | Preset ID (e.g. `v_sign`) |
| `custom_reference` | file | No | Custom gesture reference photo |
| `custom_prompt` | string | No | Custom text prompt (max 3000 chars) |
| `webhook_url` | string | No | URL to receive completion callback |
| `engine` | string | No | `seedream5` (default) or `seedream4` |
| `aspect_ratio` | string | No | `1:1` (default), `4:3`, `3:4`, `16:9`, `9:16`, `2:3`, `3:2`, `21:9` |

**Requirements:** at least one of `gesture_id`, `custom_reference`, or `custom_prompt` must be provided. Both `custom_reference` and `custom_prompt` can be used together or separately with either engine.

**Response:**

```json
{
  "task_id": "a1b2c3d4e5f6...",
  "kie_task_id": "task_seedream_123456",
  "status": "queued",
  "engine": "seedream5",
  "estimated_seconds": 30
}
```

---

### 4. Check Status

```
GET /api/v1/status/{task_id}
```

**Response:**

```json
{
  "task_id": "a1b2c3d4e5f6...",
  "status": "completed",
  "engine": "seedream5",
  "result_url": "https://tempfile.example.com/result.jpg",
  "cost_coins": 15000,
  "error": null
}
```

**Statuses:** `queued` → `running` → `completed` | `failed`

**Note:** result URLs are temporary (20 min validity). Download or cache immediately.

---

### 5. Generate (Sync)

```
POST /api/v1/generate/sync
Content-Type: multipart/form-data
```

Same fields as async generate, but waits up to **180 seconds** for completion.

Returns `StatusResponse` directly. Returns `504` on timeout.

---

### 6. Webhook (internal)

```
POST /api/v1/webhook/{provider}
```

Receives callbacks from generation backends. Not intended for client use.

---

## Integration Examples

### cURL — Async with preset (Seedream 5.0)

```bash
curl -X POST https://onlygen.mvt-soft.work/api/v1/generate \
  -H "X-API-Key: gc_your_key" \
  -F "user_photo=@photo.jpg" \
  -F "gesture_id=v_sign" \
  -F "engine=seedream5"
```

### cURL — Async with preset (Seedream 4.5)

```bash
curl -X POST https://onlygen.mvt-soft.work/api/v1/generate \
  -H "X-API-Key: gc_your_key" \
  -F "user_photo=@photo.jpg" \
  -F "gesture_id=v_sign" \
  -F "engine=seedream4"
```

### cURL — Custom prompt (text only)

```bash
curl -X POST https://onlygen.mvt-soft.work/api/v1/generate \
  -H "X-API-Key: gc_your_key" \
  -F "user_photo=@photo.jpg" \
  -F "custom_prompt=Change the hand to a thumbs up gesture"
```

### cURL — Custom reference photo

```bash
curl -X POST https://onlygen.mvt-soft.work/api/v1/generate \
  -H "X-API-Key: gc_your_key" \
  -F "user_photo=@photo.jpg" \
  -F "custom_reference=@reference.jpg"
```

### cURL — Custom prompt + reference photo

```bash
curl -X POST https://onlygen.mvt-soft.work/api/v1/generate \
  -H "X-API-Key: gc_your_key" \
  -F "user_photo=@photo.jpg" \
  -F "custom_reference=@reference.jpg" \
  -F "custom_prompt=Match the pose exactly"
```

### cURL — Custom aspect ratio

```bash
curl -X POST https://onlygen.mvt-soft.work/api/v1/generate \
  -H "X-API-Key: gc_your_key" \
  -F "user_photo=@photo.jpg" \
  -F "gesture_id=v_sign" \
  -F "aspect_ratio=9:16"
```

### cURL — Poll status

```bash
curl https://onlygen.mvt-soft.work/api/v1/status/a1b2c3d4e5f6... \
  -H "X-API-Key: gc_your_key"
```

### Python — Full flow

```python
import httpx
import asyncio

API = "https://onlygen.mvt-soft.work"
KEY = "gc_your_key"

async def generate_gesture(photo_path: str, gesture_id: str, engine: str = "seedream5"):
    async with httpx.AsyncClient(timeout=30) as client:
        headers = {"X-API-Key": KEY}

        # Submit task
        with open(photo_path, "rb") as f:
            resp = await client.post(
                f"{API}/api/v1/generate",
                headers=headers,
                files={"user_photo": f},
                data={"gesture_id": gesture_id, "engine": engine},
            )
        resp.raise_for_status()
        task_id = resp.json()["task_id"]

        # Poll until complete
        while True:
            resp = await client.get(
                f"{API}/api/v1/status/{task_id}",
                headers=headers,
            )
            data = resp.json()
            if data["status"] in ("completed", "failed"):
                return data
            await asyncio.sleep(5)

result = asyncio.run(generate_gesture("photo.jpg", "v_sign", "seedream5"))
print(result["result_url"])
```

---

## Task Queue

Requests are processed through a Redis-backed queue with concurrency limiting.

- **Max concurrent tasks:** 3 (configurable)
- **Max queue size:** 100
- If the queue is full, the API returns `503 Service Unavailable`
- Queue status is available via `/health`

---

## Error Codes

| Code | Meaning |
|------|---------|
| 400 | Missing required fields |
| 401 | Invalid or expired API key |
| 403 | Revoked key or invalid webhook token |
| 404 | Unknown gesture_id or task_id |
| 422 | Invalid engine/aspect_ratio value or prompt validation failed |
| 429 | Rate limit exceeded |
| 503 | Queue full — try again later |
| 504 | Sync generation timeout (180s) |

---

## Embeddable Widget (iframe)

Ready-to-use widget for embedding gesture generation into CRM panels or any web page.

### Quick Start

```html
<iframe
  src="https://onlygen.mvt-soft.work/static/widget/index.html?key=YOUR_API_KEY&theme=dark"
  width="450"
  height="700"
  frameborder="0"
  allow="clipboard-write"
></iframe>
```

### Widget Parameters

| Parameter | Required | Values | Default |
|-----------|----------|--------|---------|
| `key` | Yes | API key (`gc_...`) | — |
| `theme` | No | `dark` / `light` | `light` |
| `engine` | No | `seedream5` / `seedream4` | `seedream5` |
| `lang` | No | `ru` / `en` | `ru` |

### CRM Integration via postMessage

The widget sends events to the parent window:

```js
// Listen for results
window.addEventListener('message', (event) => {
  if (event.data.type === 'onlygen_result') {
    console.log('Result URL:', event.data.result_url);
    // Insert image into chat
  }
  if (event.data.type === 'onlygen_error') {
    console.error('Error:', event.data.error);
  }
});
```

Full widget documentation: [WIDGET_INTEGRATION.md](WIDGET_INTEGRATION.md)

---

## File Constraints

- **Formats:** JPEG, PNG
- **Max size:** 10 MB
- **Validation:** magic bytes + Pillow verify + re-encode
- **Prompt max length:** 3000 characters

---

## Important Notes

- **Result URL expiry:** Generated image URLs expire after 20 minutes. Download or cache results immediately after receiving them.
- **Image quality:** Output quality is fixed at "basic" (2K resolution).
