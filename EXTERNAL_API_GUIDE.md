# SendAPI External API — Integration Guide

## Overview

The SendAPI External API lets you integrate omnichannel posting into your application. Your users connect their platform accounts (YouTube, Telegram, TikTok, Instagram, etc.) and your app publishes content to all of them in a single API call.

**Base URL:** `https://api.sendapi.labslumen.com/api/v1`

**Authentication:** Bearer token with an API key (`sk_live_...`)

**All requests must include:**

```
Authorization: Bearer sk_live_your_key_here
Content-Type: application/json
```

---

## Quick Start

### 1. Get Your API Key

Your API key will be provided by the SendAPI team. It looks like `sk_live_a1b2c3d4e5f6...`. Store it securely — it cannot be recovered.

### 2. Create a User

Each of your end-users needs a corresponding SendAPI user. Use your internal user ID as the `external_user_id`:

```bash
curl -X POST https://api.sendapi.labslumen.com/api/v1/users \
  -H "Authorization: Bearer sk_live_your_key_here" \
  -H "Content-Type: application/json" \
  -d '{
    "external_user_id": "user_12345",
    "email": "user@example.com",
    "language": "en",
    "plan": "free"
  }'
```

**Response (201):**

```json
{
  "id": 42,
  "external_user_id": "user_12345",
  "email": "user@example.com",
  "language": "en",
  "is_claimed": false,
  "plan_name": "free",
  "created_at": "2026-04-05T10:00:00Z"
}
```

Save the returned `id` — you'll use it in all subsequent calls for this user.

### 3. Connect a Platform Account

Your user needs to authorize their platform accounts via OAuth. Generate an OAuth URL and redirect them to it:

```bash
curl -X POST https://api.sendapi..labslumencom/api/v1/users/42/accounts/youtube/connect \
  -H "Authorization: Bearer sk_live_your_key_here"
```

**Response (200):**

```json
{
  "oauth_url": "https://accounts.google.com/o/oauth2/auth?client_id=...&state=..."
}
```

Redirect the user to `oauth_url` in their browser. After they authorize, the account is automatically linked.

### 4. Create a Post

Once the user has at least one connected account, you can publish content:

```bash
curl -X POST https://api.sendapi.labslumen.com/api/v1/users/42/posts \
  -H "Authorization: Bearer sk_live_your_key_here" \
  -H "Content-Type: application/json" \
  -d '{
    "title": "My awesome video",
    "description": "Check this out!",
    "tags": ["tech", "tutorial"],
    "platforms": ["youtube", "telegram"],
    "media_type": "video",
    "media_source": "https://your-cdn.com/video.mp4"
  }'
```

**Response (201):**

```json
{
  "id": 123,
  "title": "My awesome video",
  "description": "Check this out!",
  "platforms": ["youtube", "telegram"],
  "status": "pending",
  "media_type": "video",
  "error_message": null,
  "platform_settings": null,
  "created_at": "2026-04-05T10:05:00Z",
  "updated_at": "2026-04-05T10:05:00Z"
}
```

The post is now queued for processing. It will be published to all specified platforms asynchronously.

### 5. Check Post Status

Poll the post to see when it's done:

```bash
curl https://api.sendapi.labslumen.com/api/v1/users/42/posts/123 \
  -H "Authorization: Bearer sk_live_your_key_here"
```

**Response (200):**

```json
{
  "id": 123,
  "title": "My awesome video",
  "description": "Check this out!",
  "platforms": ["youtube", "telegram"],
  "status": "success",
  "media_type": "video",
  "error_message": null,
  "platform_settings": null,
  "created_at": "2026-04-05T10:05:00Z",
  "updated_at": "2026-04-05T10:05:30Z"
}
```

Or use [webhooks](#webhooks) to get notified automatically.

---

## Authentication

Every request to `/api/v1/*` must include your API key:

```
Authorization: Bearer sk_live_your_key_here
```

If the key is missing, invalid, or your plan doesn't include API access, you'll receive:

| Status | Meaning                                                    |
| ------ | ---------------------------------------------------------- |
| `401`  | Missing or invalid API key                                 |
| `403`  | Plan doesn't include API access, or no active subscription |

---

## Rate Limiting

Requests are rate-limited per API key based on your plan. Every response includes rate limit headers:

```
X-RateLimit-Limit: 600
X-RateLimit-Remaining: 599
X-RateLimit-Reset: 1712300460
```

When you exceed the limit, you'll receive a `403 Forbidden` response with a message indicating when to retry.

---

## Plans

Each user in your organization is assigned a subscription plan that controls their usage limits. You choose which plan to assign each user.

### List Available Plans

```
GET /plans
```

Returns all active plans with their limits:

```json
[
  {
    "name": "free",
    "display_name": "Free",
    "uploads_limit": 10,
    "accounts_limit": 3,
    "file_size_limit": 68157440,
    "history_days": 30,
    "priority": false,
    "features": null
  },
  {
    "name": "pro",
    "display_name": "Pro",
    "uploads_limit": 50,
    "accounts_limit": 10,
    "file_size_limit": 524288000,
    "history_days": 365,
    "priority": false,
    "features": { "ai_generations_limit": 500, "twitter_posts_limit": 30 }
  },
  {
    "name": "business",
    "display_name": "Business",
    "uploads_limit": 500,
    "accounts_limit": 0,
    "file_size_limit": 2147483648,
    "history_days": 0,
    "priority": true,
    "features": { "ai_generations_limit": 2000, "twitter_posts_limit": 100 }
  }
]
```

Use the `name` field when creating or updating users. A value of `0` means unlimited.

### Subscription Period

Each user's plan has a `period_end` date. You provide this when creating or updating a user. If omitted, it defaults to 30 days from now.

**Important:** When `period_end` passes without renewal, the user is downgraded to the Free plan. To prevent this, update the user with a new `period_end` before expiry. You will receive `subscription.expiring` webhooks when a subscription is within 3 days of expiry (checked daily).

---

## Users

Users represent your end-users within SendAPI. Each user has their own connected accounts, posts, and subscription plan, isolated from other users.

### Create User

```
POST /users
```

| Field              | Type     | Required | Description                                            |
| ------------------ | -------- | -------- | ------------------------------------------------------ |
| `external_user_id` | string   | Yes      | Your internal user ID (1-255 chars, unique per org)    |
| `email`            | string   | No       | User's email address                                   |
| `language`         | string   | No       | Language code (e.g., `"en"`, `"ru"`)                   |
| `plan`             | string   | No       | Plan name (defaults to `"free"` if not set)            |
| `period_end`       | datetime | No       | Subscription end date (ISO 8601, defaults to +30 days) |

**Errors:**

- `400 Bad Request` — invalid plan name
- `409 Conflict` — `external_user_id` already exists in your organization

### Update User

Change a user's plan, timezone, or subscription period. Usage counters are **not** reset on plan change — the new plan's limits apply to the already-consumed usage for the current billing period.

```
PATCH /users/{user_id}
```

| Field        | Type     | Required | Description                          |
| ------------ | -------- | -------- | ------------------------------------ |
| `plan`       | string   | No       | New plan name                        |
| `timezone`   | string   | No       | IANA timezone (e.g. Europe/Moscow)   |
| `period_end` | datetime | No       | New subscription end date (ISO 8601) |

At least one field must be provided.

**Response:** returns the updated user object.

**Errors:**

- `400 Bad Request` — invalid plan name
- `404 Not Found` — user does not exist in your organization

### List Users

```
GET /users?page=1&per_page=20
```

Returns a paginated list:

```json
{
  "items": [{ "id": 42, "external_user_id": "user_12345", "plan_name": "free", ... }],
  "total": 150,
  "page": 1,
  "per_page": 20
}
```

### Get User

```
GET /users/{user_id}
```

### Delete User

```
DELETE /users/{user_id}
```

Returns `204 No Content`. Removes the user from your organization (does not delete their data if they've claimed their account).

### Generate Claim Token

```
POST /users/{user_id}/claim-token
```

Generates a one-time URL that lets the user create their own SendAPI account with all their connected platforms already attached.

**Response:**

```json
{
  "claim_url": "https://sendapi.labslumen.com/claim?token=abc123...",
  "expires_in": 86400
}
```

Show this URL to your user (in-app banner, email, etc.). The token expires in 24 hours.

---

## Accounts

Platform accounts (YouTube, Telegram, etc.) belong to individual users.

### List Accounts

```
GET /users/{user_id}/accounts
```

**Response:**

```json
[
  {
    "id": 1,
    "platform": "youtube",
    "platform_account_id": "UC...",
    "account_name": "My Channel",
    "account_email": "user@gmail.com"
  },
  {
    "id": 2,
    "platform": "telegram",
    "platform_account_id": "-100123456",
    "account_name": "My Channel",
    "account_email": null
  }
]
```

### Connect Account (OAuth platforms)

```
POST /users/{user_id}/accounts/{platform}/connect
```

**Supported platforms:** `youtube`, `twitter`, `tiktok`, `instagram`, `threads`, `facebook`

Returns a real OAuth URL for the platform. Redirect the user's browser to this URL. After they authorize, the account is automatically linked to the user.

**Response:**

```json
{
  "oauth_url": "https://accounts.google.com/o/oauth2/auth?client_id=...&state=..."
}
```

**Errors:**

- `400 Bad Request` — invalid platform name or platform doesn't support OAuth

### Connect Telegram Channel

Telegram uses a different connection flow (no OAuth). The SendAPI bot must be added as a channel admin first.

```
POST /users/{user_id}/accounts/telegram/connect
```

| Field        | Type   | Required | Description                      |
| ------------ | ------ | -------- | -------------------------------- |
| `channel_id` | string | Yes      | Telegram channel ID or @username |

**Flow:**

1. User adds `@sendapi_bot` as an administrator to their Telegram channel
2. Your app calls this endpoint with the channel ID or @username
3. The bot verifies it has admin access
4. Account is created and returned

**Response (201):**

```json
{
  "id": 3,
  "platform": "telegram",
  "platform_account_id": "-1001234567890",
  "account_name": "My Channel",
  "account_email": null
}
```

**Errors:**

- `400 Bad Request` — bot is not an admin of the channel, or chat is not a channel/group
- `409 Conflict` — channel already connected to this user

### Disconnect Account

```
DELETE /users/{user_id}/accounts/{account_id}
```

Returns `204 No Content`.

---

## Posts

### Upload Media (Resumable Chunked Protocol — Recommended)

Large files (> 100 MB) MUST use the chunked protocol. It survives network drops, works behind CDNs with per-request body limits, and returns the same `file_path` you pass as `media_source` when creating a post.

The protocol is four endpoints: **init → PUT chunks → complete**, with an optional **status** poll to support resume after a disconnect.

#### 1. Initialize a session

```
POST /upload/chunked
```

Body: `{ "filename": "<name>", "file_size": <bytes> }`

```bash
curl -X POST https://api.sendapi.labslumen.com/api/v1/upload/chunked \
  -H "Authorization: Bearer sk_live_your_key_here" \
  -H "Content-Type: application/json" \
  -d '{"filename": "video.mp4", "file_size": 536870912}'
```

**Response (201):**

```json
{
  "upload_id": "8f3b1c2e-…",
  "chunk_size": 65536,
  "max_file_size": 1073741824
}
```

#### 2. Upload each chunk

```
PUT /upload/chunked/{upload_id}?offset={byte_offset}
```

Body: raw bytes (length = `chunk_size`; the last chunk may be shorter). `Content-Type: application/octet-stream`. Safe to retry at the same offset on failure — the server overwrites idempotently.

```bash
curl -X PUT "https://api.sendapi.labslumen.com/api/v1/upload/chunked/<upload_id>?offset=0" \
  -H "Authorization: Bearer sk_live_your_key_here" \
  -H "Content-Type: application/octet-stream" \
  --data-binary @chunk0.bin
```

**Response (200):** `{ "chunk_index": 0, "bytes_received": 65536 }`

#### 3. (Optional) Check status to resume

```
GET /upload/chunked/{upload_id}/status
```

**Response (200):** `{ "upload_id", "filename", "expected_size", "received_bytes", "complete" }`

If a network drop interrupted your upload, call this endpoint — `received_bytes` tells you the offset to resume from.

#### 4. Complete

```
POST /upload/chunked/{upload_id}/complete
```

**Response (200):**

```json
{
  "file_path": "/tmp/sendapi/a1b2c3d4.mp4",
  "filename": "video.mp4",
  "file_size": "536870912",
  "sha256": "deadbeef…"
}
```

Pass `file_path` as `media_source` when creating the post.

#### Python example

```python
import os
import httpx

API = 'https://api.sendapi.labslumen.com/api/v1'
KEY = 'sk_live_your_key_here'


def upload(path: str) -> str:
    """Upload a file with the chunked protocol; returns the server file_path."""
    size = os.path.getsize(path)
    headers = {'Authorization': f'Bearer {KEY}'}
    with httpx.Client(headers=headers, timeout=300) as client:
        init = client.post(
            f'{API}/upload/chunked',
            json={'filename': os.path.basename(path), 'file_size': size},
        ).json()

        chunk_size = init['chunk_size']
        with open(path, 'rb') as f:
            offset = 0
            while True:
                chunk = f.read(chunk_size)
                if not chunk:
                    break
                client.put(
                    f'{API}/upload/chunked/{init["upload_id"]}',
                    params={'offset': offset},
                    content=chunk,
                    headers={'Content-Type': 'application/octet-stream'},
                )
                offset += len(chunk)

        return client.post(
            f'{API}/upload/chunked/{init["upload_id"]}/complete'
        ).json()['file_path']


server_path = upload('video.mp4')
# Pass server_path as media_source when creating the post.
```

---

### Upload Media File (Deprecated single-shot)

> **Deprecated — sunset 2026-05-20.** Responses carry `Deprecation: true` and `Sunset` headers per RFC 8594. Cannot support files > 100 MB when the API is fronted by a CDN. Migrate to the chunked protocol above.

```
POST /upload/file
```

Send the file as multipart form data with the field name `data`.

```bash
curl -X POST https://api.sendapi.labslumen.com/api/v1/upload/file \
  -H "Authorization: Bearer sk_live_your_key_here" \
  -F "data=@video.mp4"
```

**Response (201):** same shape as chunked `complete` — `{ "file_path", "filename", "file_size", "sha256" }`.

### Create Post

```
POST /users/{user_id}/posts
```

| Field                        | Type     | Required | Description                                                                                                     |
| ---------------------------- | -------- | -------- | --------------------------------------------------------------------------------------------------------------- |
| `title`                      | string   | Yes      | Post title (1-200 chars)                                                                                        |
| `description`                | string   | No       | Post description (max 5000 chars)                                                                               |
| `tags`                       | string[] | No       | List of tags                                                                                                    |
| `platforms`                  | string[] | Yes      | Target platforms (at least one)                                                                                 |
| `media_type`                 | string   | Yes      | `"photo"` or `"video"`                                                                                          |
| `media_source`               | string   | No       | Public URL (`https://...`) or server path from upload endpoint (max 2048 chars)                                 |
| `target_account_ids`         | int[]    | No       | Specific account IDs to publish to. If omitted, publishes to all connected accounts for the specified platforms |
| `telegram_publication_types` | string[] | No       | Telegram publication types. Default: `["post"]`                                                                 |
| `platform_settings`          | object   | No       | Per-platform settings (see below)                                                                               |

#### Media Source

There are two ways to provide media:

**Option A: Direct upload (recommended for files on disk)**

Upload the file first, then use the returned path:

```bash
# Step 1: Upload the file
curl -X POST https://api.sendapi.labslumen.com/api/v1/upload/file \
  -H "Authorization: Bearer sk_live_your_key_here" \
  -F "data=@video.mp4"

# Response: { "file_path": "/tmp/sendapi/abc123.mp4", ... }

# Step 2: Create the post with the server path
curl -X POST https://api.sendapi.labslumen.com/api/v1/users/42/posts \
  -H "Authorization: Bearer sk_live_your_key_here" \
  -H "Content-Type: application/json" \
  -d '{
    "title": "My video",
    "platforms": ["youtube"],
    "media_type": "video",
    "media_source": "/tmp/sendapi/abc123.mp4"
  }'
```

**Option B: URL-based (recommended for files already on a CDN)**

Provide a publicly accessible URL. The worker downloads it during processing.

```json
{
  "title": "My video",
  "platforms": ["youtube"],
  "media_type": "video",
  "media_source": "https://your-cdn.com/video.mp4"
}
```

#### Platform Settings

For TikTok, you can specify privacy and content settings:

```json
{
  "platform_settings": {
    "tiktok": {
      "privacy_level": "PUBLIC_TO_EVERYONE",
      "allow_comment": true,
      "allow_duet": false,
      "allow_stitch": false,
      "branded_content": false,
      "music_usage_confirmed": true
    }
  }
}
```

TikTok privacy levels: `PUBLIC_TO_EVERYONE`, `MUTUAL_FOLLOW_FRIENDS`, `FOLLOWER_OF_CREATOR`, `SELF_ONLY`

**Errors:**

- `403 Forbidden` — upload quota exceeded

### Post Status Flow

```
pending -> processing -> success
                      -> failed
```

| Status       | Meaning                                              |
| ------------ | ---------------------------------------------------- |
| `pending`    | Queued, waiting to be picked up                      |
| `processing` | Worker is uploading to platforms                     |
| `success`    | Published to all platforms                           |
| `failed`     | One or more platforms failed (check `error_message`) |

### List Posts

```
GET /users/{user_id}/posts?page=1&per_page=20&post_status=success
```

All query parameters are optional. `post_status` filters by status.

### Get Post

```
GET /users/{user_id}/posts/{post_id}
```

---

## Webhooks

Instead of polling for post status, register a webhook to receive notifications when posts complete or fail.

### Register Webhook

```
POST /webhooks
```

| Field    | Type     | Required | Description                              |
| -------- | -------- | -------- | ---------------------------------------- |
| `url`    | string   | Yes      | Your HTTPS callback URL (max 2048 chars) |
| `events` | string[] | Yes      | Events to subscribe to                   |

**Supported events:**

- `post.completed` — post successfully published to all platforms
- `post.failed` — post failed on one or more platforms
- `post.processing` — post picked up by worker
- `account.disconnected` — a platform account token expired
- `subscription.expiring` — a user's subscription is within 3 days of expiry (checked daily; renew via PATCH /users/{user_id})

**Response (201):**

```json
{
  "id": 1,
  "url": "https://your-app.com/webhooks/sendapi",
  "events": ["post.completed", "post.failed"],
  "is_active": true,
  "failure_count": 0,
  "created_at": "2026-04-05T10:00:00Z",
  "secret": "whsec_abc123..."
}
```

**Store the `secret` securely** — it's shown only once and is used to verify webhook signatures.

### Webhook Payload

When an event fires, SendAPI sends a `POST` request to your URL:

```json
{
  "event": "post.completed",
  "timestamp": "2026-04-05T10:05:30Z",
  "data": {
    "post_id": 123,
    "user_id": 42,
    "external_user_id": "user_12345",
    "status": "success"
  }
}
```

### Verifying Signatures

Every webhook request includes an `X-SendAPI-Signature` header with an HMAC-SHA256 signature of the request body:

```python
import hmac
import hashlib

def verify_webhook(body: bytes, signature: str, secret: str) -> bool:
    expected = hmac.new(
        secret.encode(),
        body,
        hashlib.sha256
    ).hexdigest()
    return hmac.compare_digest(expected, signature)
```

```javascript
const crypto = require("crypto");

function verifyWebhook(body, signature, secret) {
  const expected = crypto
    .createHmac("sha256", secret)
    .update(body)
    .digest("hex");
  return crypto.timingSafeEqual(Buffer.from(expected), Buffer.from(signature));
}
```

### Retry Policy

- Non-2xx response: retries 3 times with exponential backoff (10s, 60s, 300s)
- After 10 consecutive failures: webhook is auto-disabled
- Re-enable via `PATCH /webhooks/{id}` with `{"is_active": true}` (resets failure counter)

### List Webhooks

```
GET /webhooks
```

### Update Webhook

```
PATCH /webhooks/{webhook_id}
```

All fields optional:

```json
{
  "url": "https://new-url.com/hook",
  "events": ["post.completed"],
  "is_active": true
}
```

### Delete Webhook

```
DELETE /webhooks/{webhook_id}
```

---

## Organization

### Get Org Info

```
GET /org
```

Returns your organization details, current usage, and per-tier user breakdown:

```json
{
  "id": 1,
  "name": "My App",
  "plan_name": "organization",
  "uploads_used": 142,
  "uploads_limit": 10000,
  "users_count": 85,
  "rate_limit_per_min": 600,
  "tier_breakdown": {
    "free": 60,
    "pro": 20,
    "business": 5
  }
}
```

### Manage API Keys

#### Create Key

```
POST /org/api-keys
```

```json
{ "name": "Production" }
```

**Response:** includes `key` field (full key, shown once).

#### List Keys

```
GET /org/api-keys
```

Returns key metadata only (prefix, name, last used) — never the full key.

#### Revoke Key

```
DELETE /org/api-keys/{key_id}
```

### Branding

```
GET /org/branding
```

Returns URLs for the "Powered by SendAPI" badge:

```json
{
  "badge_url": "https://sendapi.labslumen.com/branding/powered-by-sendapi.svg",
  "landing_url": "https://sendapi.labslumen.com?ref=api"
}
```

Display the badge in your app to give your users a path to discover SendAPI directly.

---

## Error Format

All errors follow a consistent format:

```json
{
  "status_code": 404,
  "detail": "User not found in this organization"
}
```

### Common Error Codes

| Status | Meaning                                                                |
| ------ | ---------------------------------------------------------------------- |
| `400`  | Bad request (invalid input, missing fields)                            |
| `401`  | Missing or invalid API key                                             |
| `403`  | Forbidden (quota exceeded, plan lacks API access, rate limit exceeded) |
| `404`  | Resource not found (user, post, account, webhook)                      |
| `409`  | Conflict (duplicate external_user_id, channel already connected)       |
| `500`  | Internal server error                                                  |

---

## Complete Example: Python

```python
import requests
import time

API_KEY = "sk_live_your_key_here"
BASE_URL = "https://api.sendapi.labslumen.com/api/v1"
HEADERS = {
    "Authorization": f"Bearer {API_KEY}",
    "Content-Type": "application/json",
}


def create_user(external_id: str, email: str = None, plan: str = "free") -> dict:
    """Create a SendAPI user for your end-user."""
    resp = requests.post(
        f"{BASE_URL}/users",
        headers=HEADERS,
        json={"external_user_id": external_id, "email": email, "plan": plan},
    )
    resp.raise_for_status()
    return resp.json()


def get_oauth_url(user_id: int, platform: str) -> str:
    """Get an OAuth URL for connecting a platform account."""
    resp = requests.post(
        f"{BASE_URL}/users/{user_id}/accounts/{platform}/connect",
        headers=HEADERS,
    )
    resp.raise_for_status()
    return resp.json()["oauth_url"]


def connect_telegram(user_id: int, channel_id: str) -> dict:
    """Connect a Telegram channel (bot must be admin first)."""
    resp = requests.post(
        f"{BASE_URL}/users/{user_id}/accounts/telegram/connect",
        headers=HEADERS,
        json={"channel_id": channel_id},
    )
    resp.raise_for_status()
    return resp.json()


def upload_file(file_path: str) -> str:
    """Upload a media file and return the server path."""
    with open(file_path, "rb") as f:
        resp = requests.post(
            f"{BASE_URL}/upload/file",
            headers={"Authorization": f"Bearer {API_KEY}"},
            files={"data": f},
        )
    resp.raise_for_status()
    return resp.json()["file_path"]


def create_post(user_id: int, title: str, media_source: str,
                platforms: list[str], media_type: str = "video") -> dict:
    """Create a post and queue it for publishing."""
    resp = requests.post(
        f"{BASE_URL}/users/{user_id}/posts",
        headers=HEADERS,
        json={
            "title": title,
            "platforms": platforms,
            "media_type": media_type,
            "media_source": media_source,
        },
    )
    resp.raise_for_status()
    return resp.json()


def wait_for_post(user_id: int, post_id: int, timeout: int = 300) -> dict:
    """Poll until the post is done processing."""
    deadline = time.time() + timeout
    while time.time() < deadline:
        resp = requests.get(
            f"{BASE_URL}/users/{user_id}/posts/{post_id}",
            headers=HEADERS,
        )
        resp.raise_for_status()
        post = resp.json()
        if post["status"] in ("success", "failed"):
            return post
        time.sleep(5)
    raise TimeoutError(f"Post {post_id} still processing after {timeout}s")


# --- Usage ---

# 1. Create a user
user = create_user("user_12345", "user@example.com")
print(f"Created user: {user['id']}")

# 2. Connect YouTube (redirect user to this URL in their browser)
oauth_url = get_oauth_url(user["id"], "youtube")
print(f"Send user to: {oauth_url}")

# 2b. Connect Telegram (bot must be admin in the channel)
# tg_account = connect_telegram(user["id"], "@my_channel")

# 3. Upload a file and create a post
server_path = upload_file("video.mp4")
post = create_post(
    user_id=user["id"],
    title="My first cross-platform post",
    media_source=server_path,
    platforms=["youtube", "telegram"],
)
print(f"Post created: {post['id']}, status: {post['status']}")

# 3b. Or use a public URL directly (no upload needed)
# post = create_post(
#     user_id=user["id"],
#     title="From CDN",
#     media_source="https://your-cdn.com/video.mp4",
#     platforms=["youtube"],
# )

# 4. Wait for completion
result = wait_for_post(user["id"], post["id"])
print(f"Post finished: {result['status']}")
```

## Complete Example: Node.js

```javascript
const API_KEY = "sk_live_your_key_here";
const BASE_URL = "https://api.sendapi.labslumen.com/api/v1";

const headers = {
  Authorization: `Bearer ${API_KEY}`,
  "Content-Type": "application/json",
};

async function createUser(externalId, email) {
  const resp = await fetch(`${BASE_URL}/users`, {
    method: "POST",
    headers,
    body: JSON.stringify({ external_user_id: externalId, email }),
  });
  if (!resp.ok) throw new Error(`Failed: ${resp.status}`);
  return resp.json();
}

async function connectTelegram(userId, channelId) {
  const resp = await fetch(
    `${BASE_URL}/users/${userId}/accounts/telegram/connect`,
    {
      method: "POST",
      headers,
      body: JSON.stringify({ channel_id: channelId }),
    },
  );
  if (!resp.ok) throw new Error(`Failed: ${resp.status}`);
  return resp.json();
}

async function createPost(userId, title, mediaSource, platforms) {
  const resp = await fetch(`${BASE_URL}/users/${userId}/posts`, {
    method: "POST",
    headers,
    body: JSON.stringify({
      title,
      platforms,
      media_type: "video",
      media_source: mediaSource,
    }),
  });
  if (!resp.ok) throw new Error(`Failed: ${resp.status}`);
  return resp.json();
}

async function getPost(userId, postId) {
  const resp = await fetch(`${BASE_URL}/users/${userId}/posts/${postId}`, {
    headers: { Authorization: `Bearer ${API_KEY}` },
  });
  if (!resp.ok) throw new Error(`Failed: ${resp.status}`);
  return resp.json();
}

// Webhook handler (Express.js)
const crypto = require("crypto");

app.post("/webhooks/sendapi", (req, res) => {
  const signature = req.headers["x-sendapi-signature"];
  const expected = crypto
    .createHmac("sha256", WEBHOOK_SECRET)
    .update(JSON.stringify(req.body))
    .digest("hex");

  if (!crypto.timingSafeEqual(Buffer.from(expected), Buffer.from(signature))) {
    return res.status(401).send("Invalid signature");
  }

  const { event, data } = req.body;
  console.log(`Received ${event} for post ${data.post_id}: ${data.status}`);

  res.status(200).send("OK");
});
```

---

## Interactive API Docs

Explore all endpoints interactively at:

```
https://api.sendapi.labslumen.com/schema/swagger
```

Click **Authorize** and paste your `sk_live_...` key to test requests directly from the browser.
