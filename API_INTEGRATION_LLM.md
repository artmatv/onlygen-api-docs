# Gesture Changer API — LLM Integration Reference

> Machine-readable API specification for LLM agents and code generators.
> Version: 1.1.0 | Updated: 2026-03-28

## API Overview

```yaml
name: Gesture Changer API
version: 1.1.0
base_url: "https://onlygen.mvt-soft.work"
auth:
  type: header
  header_name: X-API-Key
  prefix: "gc_"
  format: "gc_{random_hex_64}"
rate_limits:
  per_key: "10/minute"
  global: "100/minute"
engines:
  - id: "qwen"
    name: "Qwen Image Edit"
    requires_reference: true
    prompt_language: "Chinese (auto from preset)"
    default: true
  - id: "kontext"
    name: "FLUX Kontext Dev"
    requires_reference: false
    prompt_language: "English"
    default: false
```

## Endpoints

### GET /health

```yaml
auth: false
response: { "status": "ok" }
```

### GET /api/v1/presets

```yaml
auth: true
response_schema:
  presets:
    type: array
    items:
      id: string          # e.g. "v_sign", "finger_gun", "l_pose_up"
      name: string        # e.g. "Peace / V Sign"
      emoji: string       # e.g. "✌️"
      category: string    # basic|cute|face_touch|chill|number|playful|formal|interactive|luck
      preview_url: string # full URL to preview image
  categories:
    type: array
    items:
      id: string
      label: string
      count: integer
  total: integer
  custom_gesture_supported: boolean  # always true
```

### POST /api/v1/generate

```yaml
auth: true
content_type: multipart/form-data
request_fields:
  user_photo:
    type: file
    required: true
    formats: [jpeg, png]
    max_size: "10MB"
  gesture_id:
    type: string
    required: conditional
    description: "Preset ID. Required for qwen if no custom_reference. Required for kontext if no custom_prompt."
    example: "v_sign"
  custom_reference:
    type: file
    required: conditional
    description: "Custom gesture reference photo. Qwen engine only. Required if no gesture_id."
  custom_prompt:
    type: string
    required: conditional
    max_length: 200
    description: "Custom text prompt. For kontext: required if no gesture_id."
  webhook_url:
    type: string
    required: false
    description: "URL to POST completion callback"
  engine:
    type: string
    required: false
    default: "qwen"
    enum: ["qwen", "kontext"]
    description: "Generation engine"
response_schema:
  task_id: string        # hex UUID, use for polling
  rh_task_id: string     # internal RunningHub ID
  status: string         # always "queued" on creation
  engine: string         # "qwen" or "kontext"
  estimated_seconds: 30
engine_requirements:
  qwen:
    - "gesture_id OR custom_reference must be provided"
    - "custom_reference is a file upload (gesture reference image)"
  kontext:
    - "gesture_id OR custom_prompt must be provided"
    - "custom_reference is ignored"
    - "prompts should be in English"
```

### GET /api/v1/status/{task_id}

```yaml
auth: true
path_params:
  task_id: string  # from GenerateResponse.task_id
response_schema:
  task_id: string
  status: enum[queued, running, completed, failed]
  engine: string         # "qwen" or "kontext"
  result_url: string|null    # image URL when completed
  cost_coins: integer|null
  generation_time_seconds: float|null
  error: string|null         # error message when failed
polling:
  recommended_interval: 5s
  backoff: 1.5x
  max_interval: 30s
  typical_completion: 20-60s
```

### POST /api/v1/generate/sync

```yaml
auth: true
content_type: multipart/form-data
request_fields: "Same as POST /api/v1/generate (minus webhook_url)"
timeout: 120s
response_schema: "Same as GET /api/v1/status/{task_id}"
notes:
  - "Blocks until task completes or 120s timeout"
  - "Returns 504 on timeout"
```

### POST /api/v1/webhook/runninghub

```yaml
auth: webhook_secret_token (query param)
purpose: "Internal — receives RunningHub task events"
not_for_client_use: true
```

## Error Codes

```yaml
400: "Missing required fields for selected engine"
401: "Invalid or expired API key"
403: "Revoked API key"
404: "Unknown gesture_id or task_id"
422: "Invalid engine value or prompt validation failed"
429: "Rate limit exceeded (retry_after in response)"
504: "Sync generation timeout"
```

## Available Presets (22 total)

```yaml
categories:
  basic: ["l_pose_up", "v_sign", "thumbs_up", "ok_sign", "open_palm"]
  cute: ["heart_fingers", "finger_heart", "air_kiss"]
  face_touch: ["chin_rest", "cheek_touch", "temple_tap"]
  chill: ["hang_loose", "rock_on"]
  number: ["one_finger", "two_fingers", "three_fingers"]
  playful: ["finger_gun", "pinch", "snap"]
  formal: ["handshake", "salute"]
  interactive: ["high_five", "fist_bump"]
  luck: ["crossed_fingers"]
```

## Integration Patterns

### Pattern 1: Async with polling (recommended)

```
1. POST /api/v1/generate → get task_id
2. Loop: GET /api/v1/status/{task_id}
   - if status == "completed" → use result_url
   - if status == "failed" → handle error
   - else → sleep(5s) and retry
```

### Pattern 2: Sync (simple but blocks)

```
1. POST /api/v1/generate/sync → get completed result directly
   - Handle 504 timeout
```

### Pattern 3: Webhook (server-to-server)

```
1. POST /api/v1/generate with webhook_url
2. Receive POST callback at webhook_url when done
```

## Code Templates

### Python (httpx)

```python
import httpx

async def generate(photo_bytes: bytes, gesture_id: str, engine: str = "qwen") -> dict:
    async with httpx.AsyncClient(timeout=30) as c:
        r = await c.post(
            f"{BASE_URL}/api/v1/generate",
            headers={"X-API-Key": API_KEY},
            files={"user_photo": ("photo.jpg", photo_bytes, "image/jpeg")},
            data={"gesture_id": gesture_id, "engine": engine},
        )
        r.raise_for_status()
        return r.json()

async def poll(task_id: str) -> dict:
    async with httpx.AsyncClient(timeout=10) as c:
        while True:
            r = await c.get(
                f"{BASE_URL}/api/v1/status/{task_id}",
                headers={"X-API-Key": API_KEY},
            )
            data = r.json()
            if data["status"] in ("completed", "failed"):
                return data
            await asyncio.sleep(5)
```

### JavaScript (fetch)

```javascript
async function generate(photoFile, gestureId, engine = "qwen") {
  const form = new FormData();
  form.append("user_photo", photoFile);
  form.append("gesture_id", gestureId);
  form.append("engine", engine);

  const res = await fetch(`${BASE_URL}/api/v1/generate`, {
    method: "POST",
    headers: { "X-API-Key": API_KEY },
    body: form,
  });
  return res.json();
}

async function poll(taskId) {
  while (true) {
    const res = await fetch(`${BASE_URL}/api/v1/status/${taskId}`, {
      headers: { "X-API-Key": API_KEY },
    });
    const data = await res.json();
    if (["completed", "failed"].includes(data.status)) return data;
    await new Promise(r => setTimeout(r, 5000));
  }
}
```

### cURL

```bash
# Generate with preset (Qwen)
curl -X POST ${BASE_URL}/api/v1/generate \
  -H "X-API-Key: ${API_KEY}" \
  -F "user_photo=@photo.jpg" \
  -F "gesture_id=v_sign" \
  -F "engine=qwen"

# Generate with preset (Kontext)
curl -X POST ${BASE_URL}/api/v1/generate \
  -H "X-API-Key: ${API_KEY}" \
  -F "user_photo=@photo.jpg" \
  -F "gesture_id=v_sign" \
  -F "engine=kontext"

# Generate with custom prompt (Kontext only)
curl -X POST ${BASE_URL}/api/v1/generate \
  -H "X-API-Key: ${API_KEY}" \
  -F "user_photo=@photo.jpg" \
  -F "custom_prompt=Change hand to thumbs up" \
  -F "engine=kontext"

# Check status
curl ${BASE_URL}/api/v1/status/{task_id} \
  -H "X-API-Key: ${API_KEY}"

# Sync generate (blocks up to 120s)
curl -X POST ${BASE_URL}/api/v1/generate/sync \
  -H "X-API-Key: ${API_KEY}" \
  -F "user_photo=@photo.jpg" \
  -F "gesture_id=v_sign"
```
