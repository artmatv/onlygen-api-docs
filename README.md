# Gesture Changer API — Integration Guide

> **Version:** 1.1.0
> **Base URL:** `https://onlygen.mvt-soft.work`
> **Updated:** 2026-03-28

## Authentication

All endpoints (except `/health`) require an API key in the `X-API-Key` header.

Keys are issued via the Telegram admin bot and have the prefix `gc_`.

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

The API supports two generation engines:

| Engine | ID | Description | Requires reference image? |
|--------|----|-------------|---------------------------|
| **Qwen Edit** | `qwen` | Qwen Image Edit + DWPose. Chinese prompts. High quality gesture replacement | Yes (preset or custom) |
| **FLUX Kontext Dev** | `kontext` | FLUX Kontext Dev. English prompts. Prompt-only generation | No |

Default engine: `qwen`.

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

Returns all 22 gesture presets grouped by category.

**Response:**

```json
{
  "presets": [
    {
      "id": "v_sign",
      "name": "Peace / V Sign",
      "emoji": "✌️",
      "category": "basic",
      "preview_url": "https://onlygen.mvt-soft.work/static/presets/v_sign.jpg"
    }
  ],
  "categories": [
    {"id": "basic", "label": "Basic gestures", "count": 5}
  ],
  "total": 22,
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
| `custom_reference` | file | Conditional | Custom gesture reference photo (Qwen only) |
| `custom_prompt` | string | No | Custom text prompt (max 200 chars) |
| `webhook_url` | string | No | URL to receive completion callback |
| `engine` | string | No | `qwen` (default) or `kontext` |

**Requirements by engine:**
- **Qwen:** must provide `gesture_id` or `custom_reference`
- **Kontext:** must provide `gesture_id` or `custom_prompt`

**Response:**

```json
{
  "task_id": "a1b2c3d4e5f6...",
  "rh_task_id": "12345678",
  "status": "queued",
  "engine": "qwen",
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
  "engine": "qwen",
  "result_url": "https://rh-images.xiaoyaoyou.com/ComfyUI_00033.png",
  "cost_coins": 17,
  "generation_time_seconds": 35.0,
  "error": null
}
```

**Statuses:** `queued` → `running` → `completed` | `failed`

---

### 5. Generate (Sync)

```
POST /api/v1/generate/sync
Content-Type: multipart/form-data
```

Same fields as async generate, but waits up to **120 seconds** for completion.

Returns `StatusResponse` directly. Returns `504` on timeout.

---

### 6. Webhook (internal)

```
POST /api/v1/webhook/runninghub
```

Receives events from RunningHub. Not intended for client use.

---

## Integration Examples

### cURL — Async with preset (Qwen)

```bash
curl -X POST https://onlygen.mvt-soft.work/api/v1/generate \
  -H "X-API-Key: gc_your_key" \
  -F "user_photo=@photo.jpg" \
  -F "gesture_id=v_sign" \
  -F "engine=qwen"
```

### cURL — Async with preset (Kontext)

```bash
curl -X POST https://onlygen.mvt-soft.work/api/v1/generate \
  -H "X-API-Key: gc_your_key" \
  -F "user_photo=@photo.jpg" \
  -F "gesture_id=v_sign" \
  -F "engine=kontext"
```

### cURL — Kontext with custom prompt

```bash
curl -X POST https://onlygen.mvt-soft.work/api/v1/generate \
  -H "X-API-Key: gc_your_key" \
  -F "user_photo=@photo.jpg" \
  -F "custom_prompt=Change the hand to a thumbs up gesture" \
  -F "engine=kontext"
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

async def generate_gesture(photo_path: str, gesture_id: str, engine: str = "qwen"):
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

result = asyncio.run(generate_gesture("photo.jpg", "v_sign", "kontext"))
print(result["result_url"])
```

---

## Error Codes

| Code | Meaning |
|------|---------|
| 400 | Missing required fields for the selected engine |
| 401 | Invalid or expired API key |
| 403 | Revoked key or invalid webhook token |
| 404 | Unknown gesture_id or task_id |
| 422 | Invalid engine value or prompt validation failed |
| 429 | Rate limit exceeded |
| 504 | Sync generation timeout (120s) |

---

## File Constraints

- **Formats:** JPEG, PNG
- **Max size:** 10 MB
- **Validation:** magic bytes + Pillow verify + re-encode
- **Prompt max length:** 200 characters
