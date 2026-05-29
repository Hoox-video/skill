---
name: hoox-api
description: >
  Guide for interacting with the Hoox public API to generate videos programmatically.
  Use when the user asks to create a video via the Hoox API, integrate Hoox into an application,
  generate scripts or videos programmatically, list voices/avatars, export videos, or manage avatars
  through the API. Covers authentication, workflow, all endpoints, and credit system.
---

# Hoox API

Base URL: `https://app.hoox.video/api/public/v1`

Full documentation: [docs.hoox.video](https://docs.hoox.video)

## Prerequisites

1. **Enterprise plan** on [app.hoox.video](https://app.hoox.video) -- API access is Enterprise-only
2. **API key** -- generate in Dashboard > Space Settings > API. Format: `hx_live_...`
3. **Credits** -- every operation (script generation, video generation, export, avatar creation) costs credits

## Authentication

All requests require the `Authorization` header:

```
Authorization: Bearer hx_live_your_api_key_here
```

Rate limit: **100 requests/minute** per workspace. Monitor via response headers:

```
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 95
X-RateLimit-Reset: 1640995200
```

## Video Creation Workflow

```mermaid
flowchart TD
    A[List voices and avatars] --> B{Script ready?}
    B -- No --> C[POST /script/generate]
    C --> D[POST /generation/start]
    B -- Yes --> D
    D --> E[GET /generation/status/jobId]
    E -- pending/processing --> E
    E -- completed --> F[POST /export/start]
    F --> G[GET /export/status/jobId]
    G -- pending/processing --> G
    G -- completed --> H[Download MP4 from video_url]
```

### Step 1 -- Discover resources

```bash
# List voices (filter by language, gender, tags)
curl -H "Authorization: Bearer $HOOX_API_KEY" \
  "https://app.hoox.video/api/public/v1/voice/list?language=en"

# List avatar looks (filter by gender, place, emotion, etc.)
curl -H "Authorization: Bearer $HOOX_API_KEY" \
  "https://app.hoox.video/api/public/v1/avatar/list?gender=female&place=office"
```

### Step 2 -- Generate script (optional)

```bash
curl -X POST "https://app.hoox.video/api/public/v1/script/generate" \
  -H "Authorization: Bearer $HOOX_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"prompt": "Create a 60-second product demo script for our new wireless earbuds, focusing on benefits, social proof, and a strong call-to-action to buy now.", "duration": 60, "webSearch": true}'
# Response: { "script": "...", "cost": 5, "extractedImages": [...] }
```

### Step 3 -- Start video generation

Two modes: **from prompt** (AI writes script + generates video) or **from script** (you provide the script).

```bash
# From prompt (duration required)
curl -X POST "https://app.hoox.video/api/public/v1/generation/start" \
  -H "Authorization: Bearer $HOOX_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "prompt": "Your prompt here...",
    "duration": 60,
    "voice_id": "VOICE_ID",
    "avatar_id": "AVATAR_LOOK_ID",
    "format": "vertical",
    "animate_image": true
  }'

# From script (voice_id required)
curl -X POST "https://app.hoox.video/api/public/v1/generation/start" \
  -H "Authorization: Bearer $HOOX_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "script": "Your script text here...",
    "voice_id": "VOICE_ID",
    "format": "vertical"
  }'
# Response: { "job_id": "...", "status": "pending", "estimated_credits": 50 }
```

### Step 4 -- Poll generation status

```bash
curl -H "Authorization: Bearer $HOOX_API_KEY" \
  "https://app.hoox.video/api/public/v1/generation/status/$JOB_ID"
# Response: { "job_id": "...", "status": "completed", "result": { "video_id": "...", "cost": 45 } }
```

Poll every 10s. Statuses: `pending` -> `processing` -> `completed` | `failed`.

### Step 5 -- Export to MP4

```bash
curl -X POST "https://app.hoox.video/api/public/v1/export/start" \
  -H "Authorization: Bearer $HOOX_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"video_id": "VIDEO_ID"}'
# Response: { "job_id": "...", "status": "pending", "estimated_credits": 0 }
```

First export for API-generated videos is free (cost included in generation).

### Step 6 -- Poll export status

```bash
curl -H "Authorization: Bearer $HOOX_API_KEY" \
  "https://app.hoox.video/api/public/v1/export/status/$EXPORT_JOB_ID"
# When completed: { "status": "completed", "result": { "video_url": "https://..." } }
```

Poll every 5s. Download the MP4 from `result.video_url`.

## Webhooks

Pass `webhook_url` in generation/export/avatar requests to receive async POST notifications instead of polling.

4 task types with different payload structures:

| Task | Identifier fields | Triggered by |
|------|------------------|--------------|
| `video_generate` | `job_id` | `/generation/start` |
| `video_export` | `job_id` | `/export/start` |
| `avatar_create` | `avatar_id`, `look_id` | `/avatar/create` |
| `avatar_edit` | `avatar_id`, `look_id` | `/avatar/edit` |

Success payload example:

```json
{
  "task": "video_generate",
  "job_id": "run_abc123",
  "status": "completed",
  "result": { "video_id": "...", "thumbnail_url": "...", "cost": 5 }
}
```

Failure payload: same structure with `"status": "failed"` and `"error": { "code": "...", "message": "..." }`.

Retry behavior: 3 attempts with 1s, 2s, 4s delays. Your endpoint must respond 200 within 10s.

## Error Handling

All errors follow one of these formats:

```json
{
  "error": "Error message",
  "code": "ERROR_CODE"
}
```

Or for validation errors:

```json
{
  "error": "Validation failed",
  "details": [{ "code": "ERROR_CODE", "message": "Details", "field": "param_name" }]
}
```

| Code | Status | Description |
|------|--------|-------------|
| `invalid_api_key` | 401 | Invalid or missing API key |
| `plan_required` | 403 | Enterprise plan required |
| `insufficient_credits` | 402 | Not enough credits |
| `rate_limit_exceeded` | 429 | Too many requests |
| `missing_content` | 400 | No prompt/script/voice_url/avatar_url provided |
| `missing_voice` | 400 | Script provided without voice_id |
| `invalid_voice_id` | 400 | Voice ID not found |
| `invalid_avatar_id` | 400 | Avatar look ID not found |
| `invalid_format` | 400 | Invalid video format |
| `job_not_found` | 404 | Job ID not found |
| `unauthorized_job` | 403 | Job belongs to another space |
| `video_not_found` | 404 | Video ID not found |

---

## Endpoint Reference

### POST /script/generate

Generate an AI video script from a prompt. [Full docs](https://docs.hoox.video/api-reference/script/generate)

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `prompt` | string | Yes | Script topic/instructions (max 10,000 chars) |
| `duration` | number | No | Target duration in seconds (10-600, default: 60) |
| `urls` | string[] | No | URLs to extract content from (max 10) |
| `webSearch` | boolean | No | Enable web search for context (default: false) |

Response: `{ "script": "...", "cost": 5, "extractedImages": ["https://..."] }`

---

### POST /generation/start

Start async video generation. Returns a `job_id` to poll. [Full docs](https://docs.hoox.video/api-reference/generation/start)

Two modes: provide `prompt` (AI generates script then video) or provide `script` directly.
At least one of `prompt`, `script`, `voice_url`, or `avatar_url` must be provided.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `prompt` | string | Conditional | Video topic. Requires `duration`. |
| `duration` | number | Conditional | Duration in seconds (10-600). Required with `prompt`. |
| `script` | string | Conditional | Pre-written script. Requires `voice_id`. |
| `voice_id` | string | Conditional | Voice ID from `/voice/list`. Required with `script` (unless `avatar_url`). |
| `voice_url` | string | No | URL to custom voice audio file |
| `avatar_id` | string | No | Avatar **look** ID from `/avatar/list` |
| `avatar_url` | string | No | URL to custom avatar image |
| `format` | string | No | `vertical` (default), `square`, `ads`, `custom` |
| `width` | number | No | Custom width 1-5000 (only with `format: "custom"`) |
| `height` | number | No | Custom height 1-5000 (only with `format: "custom"`) |
| `media_urls` | string[] | No | URLs to images/videos to include |
| `web_urls` | string[] | No | URLs to extract content from (max 10) |
| `web_search` | object | No | `{ script?: boolean, images?: boolean }` |
| `animate_image` | boolean | No | Animate still images into video clips (Pro/Enterprise) |
| `animate_mode` | string | No | `standard` or `pro` |
| `animate_image_max` | number | No | Max images to animate |
| `emotion_enhancement` | boolean | No | Enhance voice emotions |
| `use_veo3` | boolean | No | Use Veo 3 for avatar rendering |
| `webhook_url` | string | No | URL for completion callback |
| `use_space_media` | boolean | No | Use media from space library (default: true) |
| `save_media_to_space` | boolean | No | Save generated media to space (default: true) |
| `enable` | object | No | `{ subtitles?: boolean, broll?: boolean, music?: boolean, transitions?: boolean }` |

Response (201): `{ "job_id": "run_abc123", "status": "pending", "estimated_credits": 50 }`

---

### GET /generation/status/{jobId}

Poll video generation progress. [Full docs](https://docs.hoox.video/api-reference/generation/status)

Status values: `pending`, `processing`, `completed`, `failed`.

When completed:

```json
{
  "job_id": "run_abc123",
  "status": "completed",
  "progress_step": 100,
  "current_step": null,
  "result": {
    "video_id": "vid_xyz789",
    "thumbnail_url": "https://...",
    "duration": 62.5,
    "cost": 45,
    "created_at": "2025-01-15T10:30:00.000Z"
  }
}
```

When failed: `{ "status": "failed", "error": { "code": "GENERATION_FAILED", "message": "..." } }`

---

### POST /export/start

Export a completed video to downloadable MP4. [Full docs](https://docs.hoox.video/api-reference/export/start)

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `video_id` | string | Yes | Video ID from generation result |
| `format` | string | No | `vertical`, `square`, `ads`, `custom` |
| `width` | number | No | Custom width 1-5000 (with `format: "custom"`) |
| `height` | number | No | Custom height 1-5000 (with `format: "custom"`) |
| `avatar_model` | string | No | `standard`, `premium`, `ultra`, `veo-3-fast`, `veo-3`, `veo-3-lite`, `omni-flash`, `ora-lite`, `ora-standard`, `ora-pro`, `seedance-2`, `seedance-2-fast`, `seedance-2-1080p` |
| `webhook_url` | string | No | URL for completion callback |

Avatar model compatibility:
- Avatars with `previewUrl` (video-based) -> only `standard`
- Avatars without `previewUrl` (image-based) -> `premium`, `ultra`, `veo-3`, `veo-3-fast`, `veo-3-lite`, `omni-flash`, `ora-lite`, `ora-standard`, `ora-pro`, `seedance-2`, `seedance-2-fast`, `seedance-2-1080p`
- Videos created with `use_veo3: true` -> `veo-3`, `veo-3-fast`, or other AI models

### Export Credit Costs

Total cost = Base export cost + High resolution surcharge + Avatar model cost.

| Component | Condition | Cost |
|-----------|-----------|------|
| **Base Export** | API-generated video | **0 credits** (included) |
| | Dashboard-created video | 5 credits / 30s |
| **High Res** | > 1920px (W) OR > 1080px (H) | +2.5 credits / 30s |
| **Avatar Model** | `standard` | Free |
| | `premium` | 0.8 credits / sec (visibility) |
| | `ultra` | 2.3 credits / sec (visibility) |
| | `veo-3-fast` / `veo-3-lite` (1080p) | 2.3 credits / sec (8s segments) |
| | `veo-3` | 5 credits / sec (8s segments) |
| | `veo-3-lite` (720p) | 0.5 credits / sec (4s/6s segments) |
| | `ora-lite` | 0.5 credits / sec (8s segments) |
| | `ora-standard` | 1 credits / sec (8s segments) |
| | `ora-pro` | 2 credits / sec (8s segments) |
| | `seedance-2` | varies |
| | `seedance-2-fast` | varies |
| | `seedance-2-1080p` | varies |

*Note: Premium/Ultra models bill by visibility duration; Veo/Ora bill by fixed segments.*

First export for API-generated videos is free.

Response (201): `{ "job_id": "run_export_456", "status": "pending", "estimated_credits": 0 }`

---

### GET /export/status/{jobId}

Poll export progress. [Full docs](https://docs.hoox.video/api-reference/export/status)

When completed:

```json
{
  "job_id": "run_export_456",
  "status": "completed",
  "progress": 100,
  "result": {
    "video_url": "https://cdn.hoox.video/exports/...",
    "thumbnail_url": "https://...",
    "cost": 0,
    "created_at": "2025-01-15T10:35:00.000Z"
  }
}
```

---

### POST /video/duplicate

Duplicate a completed video, optionally changing voice and/or avatar. [Full docs](https://docs.hoox.video/api-reference/video/duplicate)

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `video_id` | string | Yes | Source video ID |
| `voice_id` | string | No | New voice ID (re-generates audio) |
| `avatar_look_id` | string | No | New avatar look ID |

Response (201): `{ "video_id": "...", "status": "done", "changes_applied": { "voice_changed": true, "avatar_changed": false } }`

The duplicated video needs to be exported via `/export/start` to get the final MP4.

---

## Asset Generation

Assets are standalone AI-generated images or videos (not tied to the video generation workflow).

### GET /asset/models

List available AI models for image/video generation.

Query params: `type` (`image` or `video`), `provider`, `tag`.

Providers: `google`, `openai`, `sora`, `kling`, `seedance`, `zimage`, `flux`, `gemini`, `reve`, `bytedance`, `xai`, `other`.

Tags: `text-to-image`, `image-to-video`, `text-to-video`, `upscale`, etc.

```json
{
  "models": [
    {
      "name": "flux-2-pro",
      "label": "Flux 2 Pro",
      "type": "image",
      "provider": "flux",
      "tags": ["text-to-image"],
      "base_cost": 2,
      "has_audio": false,
      "max_resolution": "4k",
      "required_plans": [],
      "capabilities": {
        "max_images": 4,
        "min_images": 0,
        "supported_aspect_ratios": ["1:1", "16:9", "9:16", "3:2", "2:3", "4:3", "3:4"],
        "default_aspect_ratio": "1:1"
      }
    }
  ],
  "count": 1
}
```

### GET /asset/models/{name}

Get full details for a specific model. Returns everything from the list plus:
- `max_prompt_length`
- `restricted_countries`
- `settings`: configurable options (key, type, label, default, options)
- `input_schema`: JSON-schema-style description of accepted inputs

---

### POST /asset/pricing

Calculate credit cost before starting a generation.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `model` | string | Yes | Model name |
| `duration` | string | No | Target duration (e.g. "6s", "10") |
| `resolution` | string | No | e.g. "720p", "1080p" |
| `model_settings` | object | No | Extra model-specific settings |
| `image_count` | number | No | Number of reference images |
| `generation_count` | number | No | Number of outputs (default 1) |

```json
{
  "model": "kling-v3-pro",
  "cost_per_generation": 8,
  "total_cost": 16,
  "currency": "credits",
  "generation_count": 2,
  "normalized": {
    "duration": "6s",
    "duration_in_seconds": 6
  }
}
```

---

### POST /asset/start

Start an AI asset generation job. Returns immediately with asset IDs.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `type` | string | Yes | `image` or `video` |
| `model` | string | Yes | Model name from `/asset/models` |
| `prompt` | string | Yes | Description of the desired output |
| `references` | array | No | Generic reference inputs (most models). Each item: `{"url": "https://...", "reference_id": "asset_id", "source": "avatar"\|"asset"\|"upload"}`. Provide `url` or `reference_id` (or both). `reference_id` is resolved automatically from the space media library. |
| `[slot_name]` | object | Depends | Named input slots for models that have `inputSlots` (e.g. `start_image`, `end_image`, `motion_video`). Each value: `{"url": "...", "reference_id": "..."}`. Check `/asset/models/{name}` `input_schema` for slot names and requirements. Cannot be combined with `references`. |
| `generation_count` | number | No | Number of outputs to generate (1–4, default 1) |
| `duration` | string | No | Video duration — `"4s"`, `"6s"`, `"8s"`, `"5"` … `"15"`, `"auto"` |
| `aspect_ratio` | string | No | `"16:9"`, `"9:16"`, `"1:1"`, `"3:2"`, `"2:3"`, `"4:3"`, `"3:4"`, `"auto"` |
| `resolution` | string | No | `"720p"`, `"1080p"`, `"4k"` (model-dependent) |
| `model_settings` | object | No | Extra model-specific options (see `/asset/models/{name}`) |
| `parent_media_id` | string | No | Asset ID to use as video-to-video source |
| `avatar_description` | string | No | Seedance UGC only — subject appearance/actions |
| `webhook_url` | string | No | URL for async completion callback |

**Reference inputs — two modes:**

*Generic mode* (most text-to-image / text-to-video models):
```json
{
  "references": [
    { "url": "https://cdn.example.com/photo.jpg", "source": "upload" },
    { "reference_id": "asset_abc123", "source": "asset" }
  ]
}
```

*Named slot mode* (image-to-video and similar models — check `input_schema` from `/asset/models/{name}`):
```json
{
  "start_image": { "url": "https://cdn.example.com/start.jpg" },
  "end_image":   { "reference_id": "asset_abc123" }
}
```

Credits are debited immediately. Poll `/asset/status/{assetId}` to track progress.

Response (201):
```json
{
  "asset_ids": ["asset_abc123"],
  "status": "generating",
  "cost_per_generation": 8,
  "total_cost": 8,
  "count": 1
}
```

---

### GET /asset/status/{assetId}

Check the status of an asset generation job.

Status values: `pending` → `generating` → `completed` | `failed`.

Poll every 5–10s.

When completed:
```json
{
  "asset_id": "asset_abc123",
  "status": "completed",
  "type": "video",
  "url": "https://cdn.hoox.video/assets/...",
  "thumbnail_url": "https://...",
  "width": 1920,
  "height": 1080,
  "duration_seconds": 6,
  "cost": 8,
  "created_at": "2025-01-15T10:30:00.000Z",
  "generation_params": {
    "model": "kling-v3-pro",
    "prompt": "...",
    "aspect_ratio": "16:9",
    "resolution": "1080p",
    "duration": "6s"
  }
}
```

When failed: `{ "status": "failed", "error": { "code": "...", "message": "..." } }`

---

### GET /voice/list

List available voices. [Full docs](https://docs.hoox.video/api-reference/voices/list)

Query params: `language`, `gender`, `tags` (comma-separated), `onlyPublic` (`true` to exclude custom voices).

```json
[
  {
    "id": "voice_en_sarah",
    "name": "Sarah",
    "language": "en",
    "accent": "American",
    "gender": "female",
    "tags": ["professional", "warm"],
    "preview": "https://...",
    "source": "config"
  }
]
```

### GET /voice/{id}

Get a single voice by ID. [Full docs](https://docs.hoox.video/api-reference/voices/get)

Same response shape as list items.

---

### GET /avatar/list

List available avatar looks. [Full docs](https://docs.hoox.video/api-reference/avatars/list)

Query params (all optional, comma-separated for multi-value):

| Param | Values |
|-------|--------|
| `gender` | `male`, `female` |
| `age_range` | `senior`, `adult`, `young_adult`, `adolescent` |
| `ethnicity` | `black-or-african-american`, `white-western-european`, `white-eastern-european`, `hispanic-or-latino`, `middle-eastern-or-north-african`, `east-asian`, `southeast-asian`, `south-asian` |
| `hair_color` | `black`, `brown`, `blonde`, `red`, `gray`, `white`, `bald` |
| `place` | `bathroom`, `beach`, `car`, `bedroom`, `podcast`, `home`, `office`, `gym`, `outdoor`, `kitchen`, `restaurant`, `studio`, `street`, `store`, `cafe`, `classroom`, `hospital`, `hotel`, `park`, `other` |
| `action` | `working`, `eating`, `drinking`, `cooking`, `exercising`, `reading`, `driving`, `listening_to_music`, `presenting`, `calling`, `relaxing`, `shopping`, `traveling`, `applying_skincare` |
| `emotion` | `happy`, `calm`, `focused`, `relaxed`, `neutral`, `excited`, `confident`, `sad`, `skeptical`, `bored`, `engaged` |
| `accessories` | `microphone`, `laptop`, `phone`, `headphones`, `earbuds`, `glasses`, `watch`, `drink`, `skincare`, `book`, `bag`, `camera`, `pillow`, `earrings`, `necklace`, `bracelet`, `ring` |
| `selfie` | `true`, `false` |
| `onlyPublic` | `true` to exclude custom avatars |

```json
[
  {
    "id": "look_abc123",
    "avatar_id": "avatar_emma",
    "avatar_name": "Emma",
    "look_name": "Office Professional",
    "gender": "female",
    "age_range": "adult",
    "ethnicity": "white-western-european",
    "hair_color": "brown",
    "place": "office",
    "accessories": ["laptop"],
    "action": "working",
    "emotion": "focused",
    "selfie": false,
    "format": "vertical",
    "thumbnail": "https://...",
    "preview": "https://...",
    "model_available": ["standard"]
  }
]
```

`model_available`: `["standard"]` = video-based avatar, `["premium", "ultra", "veo-3", "veo-3-fast", "veo-3-lite", "omni-flash", "ora-lite", "ora-standard", "ora-pro", "seedance-2", "seedance-2-fast", "seedance-2-1080p"]` = image-based avatar.

Use the look `id` as `avatar_id` in `/generation/start`.

### GET /avatar/{id}

Get avatar details with all its looks. [Full docs](https://docs.hoox.video/api-reference/avatars/get)

```json
{
  "id": "avatar_emma",
  "name": "Emma",
  "gender": "female",
  "age": "30",
  "age_range": "adult",
  "ethnicity": "white-western-european",
  "hair_color": "brown",
  "thumbnail": "https://...",
  "looks": [
    {
      "id": "look_abc123",
      "name": "Office Professional",
      "place": "office",
      "accessories": ["laptop"],
      "action": "working",
      "emotion": "focused",
      "selfie": false,
      "format": "vertical",
      "thumbnail": "https://...",
      "preview": "https://...",
      "status": "ready",
      "model_available": ["standard"]
    }
  ]
}
```

### GET /avatar/{id}/look/{lookId}

Get a specific look. [Full docs](https://docs.hoox.video/api-reference/avatars/get-look)

```json
{
  "id": "look_abc123",
  "avatar_id": "avatar_emma",
  "avatar_name": "Emma",
  "look_name": "Office Professional",
  "gender": "female",
  "age_range": "adult",
  "ethnicity": "white-western-european",
  "hair_color": "brown",
  "place": "office",
  "accessories": ["laptop"],
  "action": "working",
  "emotion": "focused",
  "selfie": false,
  "format": "vertical",
  "thumbnail": "https://...",
  "preview": "https://...",
  "model_available": ["standard"],
  "status": "ready",
  "error_message": null
}
```

---

### POST /avatar/create

Create a custom avatar from a prompt or images. [Full docs](https://docs.hoox.video/api-reference/avatars/create)

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `prompt` | string | No | Description for AI-generated avatar |
| `style` | string | No | `iphone` (default), `selfie`, `podcast`, `car`, `conference`, `raw` |
| `format` | string | No | `vertical` (default) or `horizontal` (only for prompt-based) |
| `resolution` | string | No | `2K` (default) or `4K` (costs more credits) |
| `name` | string | No | Avatar name |
| `place` | string | No | Location/setting description |
| `imageUrls` | string[] | No | Alias for `images` |
| `images` | string[] | No | Array of image URLs to use as-is for looks (not reference) |
| `elementImages` | string[] | No | Reference images for the AI to incorporate (logos, products) |
| `webhook_url` | string | No | URL for completion callback |

Provide either `prompt` (AI generates image) or `images` (use provided images directly).

**Credit Costs:**
- **Prompt-based (2K)**: 2 credits
- **Prompt-based (4K)**: 4 credits
- **Image-based**: Free

Response (201): `{ "avatar_id": "...", "look_id": "..." }`

Generation happens in background. Poll with `/avatar/status` or use `webhook_url`.

### POST /avatar/edit

Create a new look for an existing avatar. [Full docs](https://docs.hoox.video/api-reference/avatars/edit)

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `avatar_id` | string | Yes | Avatar ID |
| `look_id` | string | Yes | Source look ID to use as reference |
| `prompt` | string | Yes | Description of the new look |
| `format` | string | No | `vertical` or `horizontal` |
| `resolution` | string | No | `2K` (default) or `4K` (costs more credits) |
| `style` | string | No | `selfie`, `podcast`, `car`, `iphone`, `conference`, `raw` |
| `upscale` | boolean | No | Upscale generated image (+1.5 credits) |
| `imageUrls` | string[] | No | Additional reference images |
| `look_name` | string | No | Name for the new look |
| `place` | string | No | Location/setting |
| `tags` | string[] | No | Tags for the new look |
| `webhook_url` | string | No | URL for completion callback |

If `avatar_id` refers to a public avatar, it is automatically duplicated to your space.

**Credit Costs:**
- **Base (2K)**: 2 credits
- **Base (4K)**: 4 credits
- **Upscale**: +1.5 credits

Response (201): `{ "avatar_id": "...", "look_id": "..." }`

### GET /avatar/status

Check avatar look generation status. [Full docs](https://docs.hoox.video/api-reference/avatars/status)

Query params: `avatar_id` (required), `look_id` (required).

Status values: `pending`, `completed`, `failed`.

```json
{
  "avatar_id": "avatar_123",
  "look_id": "look_456",
  "status": "completed",
  "result": {
    "id": "look_456",
    "avatar_name": "Emma",
    "look_name": "Casual",
    "gender": "female",
    "place": "office",
    "accessories": ["laptop"],
    "action": "working",
    "emotion": "focused",
    "selfie": false,
    "format": "vertical",
    "thumbnail": "https://...",
    "model_available": ["premium", "ultra", "veo-3-fast", "veo-3", "veo-3-lite", "omni-flash", "ora-lite", "ora-standard", "ora-pro", "seedance-2", "seedance-2-fast", "seedance-2-1080p"]
  }
}
```
