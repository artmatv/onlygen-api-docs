<p align="center">
  <img src="../static/logo/onlygen-logo-cropped.svg" alt="OnlyGen" width="180">
</p>

# Gesture Changer API — LLM Integration Reference

> Machine-readable API specification for LLM agents and code generators.
> Version: 2.3.0 | Updated: 2026-04-06

## API Overview

```yaml
name: Gesture Changer API
version: 2.0.0
base_url: "https://onlygen.mvt-soft.work"
auth:
  type: header
  header_name: X-API-Key
  prefix: "gc_"
  format: "gc_{token_urlsafe_32}"  # 43 base64 chars
rate_limits:
  per_key: "10/minute"
  global: "100/minute"
engines:
  - id: "seedream5"
    name: "Seedream 5.0 Lite"
    description: "Lightweight gesture replacement, better identity preservation"
    default: true
  - id: "seedream4"
    name: "Seedream 4.5 Edit"
    description: "Precise gesture replacement, maximum realism"
    default: false
aspect_ratios: ["1:1", "4:3", "3:4", "4:5", "5:4", "16:9", "9:16", "2:3", "3:2", "21:9", "9:21"]
result_url_expiry: "20 minutes"
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
      id: string          # e.g. "peace_palm", "finger_gun", "l_pose"
      name: string        # e.g. "Peace (Palm)"
      emoji: string       # e.g. "✌️"
      category: string    # popular|cute|face_touch|playful|number
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
    formats: [jpeg, png, webp]
    max_size: "10MB"
  gesture_id:
    type: string
    required: false
    description: "Preset ID from /presets"
    example: "peace_palm"
  custom_reference:
    type: file
    required: false
    description: "Custom gesture reference photo"
  custom_prompt:
    type: string
    required: false
    max_length: 3000
    description: "Custom text prompt for generation"
  webhook_url:
    type: string
    required: false
    description: "URL to POST completion callback"
  engine:
    type: string
    required: false
    default: "seedream5"
    enum: ["seedream5", "seedream4"]
  aspect_ratio:
    type: string
    required: false
    default: "1:1"
    enum: ["1:1", "4:3", "3:4", "4:5", "5:4", "16:9", "9:16", "2:3", "3:2", "21:9", "9:21"]
  reference_type:
    type: string
    required: false
    default: "gesture"
    enum: ["gesture", "pose", "scene", "style"]
    description: "What to copy from custom_reference image"
    values:
      gesture: "Hand gesture and finger positions"
      pose: "Body pose, posture, arm placement"
      scene: "Setting, background, environment"
      style: "Lighting, color grading, photographic style"
validation:
  - "At least one of gesture_id, custom_reference, or custom_prompt must be provided"
  - "custom_reference and custom_prompt can be used together or separately with either engine"
  - "reference_type only applies when custom_reference is provided"
response_schema:
  task_id: string        # hex UUID, use for polling
  kie_task_id: string    # external task ID on generation backend
  status: string         # always "queued" on creation
  engine: string         # "seedream5" or "seedream4"
  estimated_seconds: 30
  queue_position: integer  # position in queue (0 = not queued / processing)
```

### GET /api/v1/status/{task_id}

```yaml
auth: true
path_params:
  task_id: string  # from GenerateResponse.task_id
response_schema:
  task_id: string
  status: enum[queued, running, completed, failed]
  engine: string         # "seedream5" or "seedream4"
  result_url: string|null    # temporary image URL when completed (expires in 20min)
  cost_coins: integer|null   # processing time in ms
  error: string|null         # error message when failed
  queue_position: integer    # position in queue (0 = not queued / processing)
polling:
  recommended_interval: 5s
  backoff: 1.5x
  max_interval: 30s
  typical_completion: 10-30s
```

### GET /api/v1/quota

```yaml
auth: true
response_schema:
  generations_used: integer
  generation_limit: integer
  generations_remaining: integer
  plan: string          # "basic" or "pro"
notes:
  - "Only successful generations count against the quota"
  - "Billing period resets every 30 days"
```

### GET /api/v1/download/{task_id}

```yaml
auth: true
path_params:
  task_id: string  # from GenerateResponse.task_id
response: binary image with Content-Disposition attachment header
notes:
  - "Proxy-downloads the result image (avoids CORS issues with storage URLs)"
  - "Only works for completed tasks"
  - "Returns 400 if task not completed, 404 if not found"
```

### POST /api/v1/generate/sync

```yaml
auth: true
content_type: multipart/form-data
request_fields: "Same as POST /api/v1/generate (minus webhook_url)"
timeout: 180s
response_schema: "Same as GET /api/v1/status/{task_id}"
notes:
  - "Blocks until task completes or 180s timeout"
  - "Returns 504 on timeout"
```

### POST /api/v1/webhook/{provider}

```yaml
auth: HMAC-SHA256 signature
purpose: "Internal — receives generation backend callbacks"
not_for_client_use: true
```

## Error Codes

```yaml
400: "Missing required fields"
401: "Invalid or expired API key"
403: "Revoked API key or invalid webhook signature"
404: "Unknown gesture_id or task_id"
422: "Invalid engine/aspect_ratio value or prompt validation failed"
429: "Rate limit exceeded (retry_after in response)"
503: "Queue full — retry after delay"
504: "Sync generation timeout"
```

## Task Queue

```yaml
backend: Redis
max_concurrent_tasks: 10
max_queue_size: 100
behavior_when_full: "503 Service Unavailable"
health_endpoint: "GET /health returns queue_length, queue_max_concurrency, queue_max_size"
```

## Available Presets (23 total)

```yaml
categories:
  popular (8): ["l_pose", "peace_palm", "peace_back", "thumbs_up", "ok_sign_palm", "open_palm", "open_palm_back", "fist"]
  cute (2): ["korean_love", "heart_hands"]
  face_touch (3): ["pointing_face", "chin_rest", "shush"]
  playful (6): ["finger_gun", "rock_horns_palm", "rock_horns_thumb", "call_me", "pointing_camera", "crossed_fingers_palm"]
  number (4): ["index_up", "three_fingers_palm", "three_fingers_back", "four_fingers_palm"]
```

## Integration Patterns

### Pattern 1: Async with polling (recommended)

```
1. POST /api/v1/generate → get task_id
2. Loop: GET /api/v1/status/{task_id}
   - if status == "completed" → use result_url (download within 20min!)
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

async def generate(photo_bytes: bytes, gesture_id: str, engine: str = "seedream5") -> dict:
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
async function generate(photoFile, gestureId, engine = "seedream5") {
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

## Embeddable Widget

```yaml
type: iframe
url: "https://onlygen.mvt-soft.work/static/widget/index.html"
params:
  key:
    type: string
    required: true
    description: "API key (gc_...)"
  theme:
    type: string
    enum: ["dark", "light"]
    default: "light"
  engine:
    type: string
    enum: ["seedream5", "seedream4"]
    default: "seedream5"
  lang:
    type: string
    enum: ["ru", "en"]
    default: "ru"
embed_example: '<iframe src="https://onlygen.mvt-soft.work/static/widget/index.html?key=API_KEY&theme=dark" width="450" height="700" frameborder="0"></iframe>'
postmessage_events:
  onlygen_result:
    task_id: string
    result_url: string
    status: "completed"
  onlygen_error:
    error: string
recommended_size: "450x700px (min 360x600px)"
```

### cURL

```bash
# Generate with preset (Seedream 5.0)
curl -X POST ${BASE_URL}/api/v1/generate \
  -H "X-API-Key: ${API_KEY}" \
  -F "user_photo=@photo.jpg" \
  -F "gesture_id=peace_palm" \
  -F "engine=seedream5"

# Generate with preset (Seedream 4.5)
curl -X POST ${BASE_URL}/api/v1/generate \
  -H "X-API-Key: ${API_KEY}" \
  -F "user_photo=@photo.jpg" \
  -F "gesture_id=peace_palm" \
  -F "engine=seedream4"

# Generate with custom prompt
curl -X POST ${BASE_URL}/api/v1/generate \
  -H "X-API-Key: ${API_KEY}" \
  -F "user_photo=@photo.jpg" \
  -F "custom_prompt=Change hand to thumbs up"

# Generate with custom reference photo
curl -X POST ${BASE_URL}/api/v1/generate \
  -H "X-API-Key: ${API_KEY}" \
  -F "user_photo=@photo.jpg" \
  -F "custom_reference=@reference.jpg"

# Generate with custom prompt + reference
curl -X POST ${BASE_URL}/api/v1/generate \
  -H "X-API-Key: ${API_KEY}" \
  -F "user_photo=@photo.jpg" \
  -F "custom_reference=@reference.jpg" \
  -F "custom_prompt=Match pose exactly"

# Generate with custom aspect ratio
curl -X POST ${BASE_URL}/api/v1/generate \
  -H "X-API-Key: ${API_KEY}" \
  -F "user_photo=@photo.jpg" \
  -F "gesture_id=peace_palm" \
  -F "aspect_ratio=9:16"

# Check status
curl ${BASE_URL}/api/v1/status/{task_id} \
  -H "X-API-Key: ${API_KEY}"

# Sync generate (blocks up to 180s)
curl -X POST ${BASE_URL}/api/v1/generate/sync \
  -H "X-API-Key: ${API_KEY}" \
  -F "user_photo=@photo.jpg" \
  -F "gesture_id=peace_palm"
```
