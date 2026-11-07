---
name: schedulin
description: Schedule and publish social media posts to Instagram, TikTok, YouTube, X, LinkedIn, Facebook, Pinterest, Threads, Bluesky, Reddit and more through the Schedulin API. Use when the user wants to post now, schedule a post for later, add a post to a channel's queue, upload media, or check what's scheduled.
version: 1.0.0
metadata:
  openclaw:
    requires:
      env:
        - SCHEDULIN_API_KEY
      bins:
        - curl
    primaryEnv: SCHEDULIN_API_KEY
    envVars:
      - name: SCHEDULIN_API_KEY
        required: true
        description: Schedulin API key, sent in the x-api-key header. Create one at https://dashboard.schedulin.app/settings/api-keys
    emoji: "🗓️"
    homepage: https://developers.schedulin.app
---

# Schedulin

[Schedulin](https://www.schedulin.app) schedules and publishes social media posts across every major network from one API. Use this skill to post now, schedule for later, queue posts into a channel's next free slot, upload media, and review what's scheduled.

## Setup

1. Create a Schedulin account at https://www.schedulin.app and connect at least one social channel in the dashboard.
2. Create an API key at https://dashboard.schedulin.app/settings/api-keys.
3. Set it as `SCHEDULIN_API_KEY`.

## Making requests

- Base URL: `https://api.schedulin.app/v0`
- Auth: send the key in the `x-api-key` header. Never use `Authorization: Bearer` for an API key; it's rejected with 401.
- Request bodies are JSON (`Content-Type: application/json`), except the media uploads below, which use `multipart/form-data` or raw bytes.
- Rate limits: up to 300 `GET` requests and 60 writes (`POST`, `PUT`, `PATCH`, `DELETE`) per minute, and 300 requests per minute combined. Responses include `RateLimit-Limit`, `RateLimit-Remaining`, and `RateLimit-Reset` (seconds until the combined window resets). On `429`, wait for the `Retry-After` header (seconds) and retry.

```bash
curl https://api.schedulin.app/v0/social-accounts \
  -H "x-api-key: $SCHEDULIN_API_KEY"
```

## Workflow

1. **Find the channel.** `GET /v0/social-accounts` returns `{ "data": [...] }`. Each item has `id`, `platform`, `username` and `status`. Use the `id` as `socialAccountId`. Never guess an ID, and skip channels whose `status` isn't `connected`.
2. **Check platform rules when unsure.** `GET /v0/platforms` lists caption limits, media rules and whether a platform needs `platformConfiguration`. Instagram, TikTok, Pinterest and YouTube need media; X, LinkedIn, Threads, Bluesky and Facebook allow text-only posts.
3. **Create the post** with `POST /v0/posts`:

| Goal | `action` | Extra fields |
| --- | --- | --- |
| Publish now | `"now"` | none |
| Schedule | `"schedule"` | `scheduledAt` (ISO 8601, UTC) |
| Next queue slot | `"queue"` | none |
| Save a draft | `"draft"` | none |

```bash
curl -X POST https://api.schedulin.app/v0/posts \
  -H "x-api-key: $SCHEDULIN_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "socialAccountId": "acc_123",
    "caption": "Weekend sale starts now",
    "media": [{ "url": "https://example.com/sale.png" }],
    "action": "schedule",
    "scheduledAt": "2030-01-15T15:00:00Z"
  }'
```

4. **Confirm the result** with `GET /v0/posts/{id}`. A `now` post starts as `PROCESSING`, `schedule` and `queue` posts start as `SCHEDULED` and move to `PROCESSING` at their time, and drafts stay `DRAFT`. Publishing ends as `COMPLETED` or `FAILED`.

## Media

- **Public URL:** pass it in `media: [{ "url": "..." }]` and Schedulin downloads it. To reuse a file, import it once with `POST /v0/media/from-url` (`{ "url": "...", "alt": "..." }`) and use the returned `url`.
- **Local file under 4.5 MB:** `POST /v0/media/upload` as `multipart/form-data` with the file in a `file` field.
- **Larger files:** a three-step upload.

```bash
# 1. Get a presigned URL. contentType and key are required.
curl -X POST https://api.schedulin.app/v0/media/presign \
  -H "x-api-key: $SCHEDULIN_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{ "contentType": "image/jpeg", "key": "launch.jpg", "size": 248173 }'
# -> { "url": "...", "key": "<your-org>/launch.jpg", "method": "PUT" }

# 2. PUT the bytes within 10 minutes, with the same Content-Type as step 1.
curl -X PUT "<url>" -H "Content-Type: image/jpeg" --data-binary @launch.jpg

# 3. Register the upload using the key returned in step 1.
curl -X POST https://api.schedulin.app/v0/media/register \
  -H "x-api-key: $SCHEDULIN_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{ "key": "<your-org>/launch.jpg", "name": "launch.jpg" }'
```
- Only use URLs that are public or that the API returned. Never invent a media URL.
- The order of `media` is the carousel order.

## Other common calls

| Task | Request |
| --- | --- |
| List scheduled posts | `GET /v0/posts?status=SCHEDULED&limit=25&page=1` |
| Get one post | `GET /v0/posts/{id}` |
| Edit an unpublished post | `PUT /v0/posts/{id}` with `caption`, `media` or `scheduledAt` |
| Publish a draft | `POST /v0/posts/{id}/publish` |
| Delete a post | `DELETE /v0/posts/{id}` (published posts stay live on the platform) |
| Upcoming queue slots | `GET /v0/social-accounts/{id}/next-slots?limit=5` |

## Guidance

- Confirm the channel, caption and time with the user before publishing or scheduling. Drafts are safe to create without asking.
- Before deleting a post, fetch it with `GET /v0/posts/{id}`, show the user its channel, caption, status and scheduled time, and delete only after they confirm that exact post. Never delete based on a guessed or inferred ID.
- Times are UTC. Convert from the user's time zone and state the converted time back to them.
- If a request fails, read the error `message`. It usually names the field to fix, such as a caption over the platform limit or a missing media file.

Full docs: https://developers.schedulin.app (index for agents: https://developers.schedulin.app/llms.txt)
