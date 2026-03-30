# Zaplink API Reference

This document provides a comprehensive guide to the Zaplink API and webhook system. Zaplink is a multi-tenant WhatsApp API gateway that allows you to manage sessions, send/receive messages, and receive real-time notifications.

---

## Authentication

Zaplink supports two methods of authentication. Both methods attach user context to the request. You should choose the method that best fits your implementation environment.

### 1. API Key Authentication (Recommended for Server-to-Server)
Use an API key for backend integrations. API keys support IP whitelisting and are designed for long-term server-side use.

**Header Format:**
```http
X-API-Key: zap_live_abcdef123.secret_part
```
*   **Format:** `prefix.secret`
*   **Lookup:** The prefix is used for database lookup.
*   **Verification:** The secret part is hashed (SHA-256) and compared in constant time.

### 2. Bearer JWT Authentication (Required for Frontend/Dashboard)
Use a Supabase-issued Bearer token for client-side applications.

**Header Format:**
```http
Authorization: Bearer <your_jwt_token>
```

### Authentication Logic
The gateway executes `apiKeyMiddleware` globally. If a valid `X-API-Key` is found, the request is authenticated. Downstream routes using `authMiddleware` will skip their checks if an API key has already provided the user context.

---

## Base URL

| Environment | URL |
| :--- | :--- |
| **Production** | `http://api.zaplink.co.ke/v1` |
| **Local Development** | `http://localhost:3000/v1` |

---

## Sending Messages

Send messages to individual phone numbers or WhatsApp groups using an active session.

### Base Endpoint
`POST /sessions/:id/messages`

**Authentication:** Bearer JWT or API Key
**Content-Type:** `application/json`

### JID Formats
*   **Individual:** `[phonenumber]@s.whatsapp.net` (e.g., `254700000000@s.whatsapp.net`)
*   **Group:** `[group_id]@g.us` (e.g., `120363000000000000@g.us`)

---

### Text Message
*   **Body Schema:**
    ```json
    {
      "to": "254700000000@s.whatsapp.net",
      "message": {
        "type": "text",
        "text": "Hello from Zaplink!"
      }
    }
    ```
*   **Curl:**
    ```bash
    curl -X POST "http://api.zaplink.co.ke/v1/sessions/zap_123/messages" \
         -H "X-API-Key: YOUR_API_KEY" \
         -H "Content-Type: application/json" \
         -d '{"to": "254700000000@s.whatsapp.net", "message": {"type": "text", "text": "Hello!"}}'
    ```

---

### Image Message
*   **Body Schema:**
    ```json
    {
      "to": "254700000000@s.whatsapp.net",
      "message": {
        "type": "image",
        "url": "https://example.com/image.jpg",
        "caption": "Check this out!",
        "mimetype": "image/jpeg"
      }
    }
    ```
*   **Fields:** `url` or `base64` is required. `caption` and `mimetype` are optional.
*   **Curl:**
    ```bash
    curl -X POST "http://api.zaplink.co.ke/v1/sessions/zap_123/messages" \
         -H "X-API-Key: YOUR_API_KEY" \
         -H "Content-Type: application/json" \
         -d '{"to": "254700000000@s.whatsapp.net", "message": {"type": "image", "url": "https://example.com/img.png"}}'
    ```

---

### Video Message
*   **Body Schema:**
    ```json
    {
      "to": "254700000000@s.whatsapp.net",
      "message": {
        "type": "video",
        "url": "https://example.com/video.mp4",
        "caption": "Watch this!",
        "gif": false
      }
    }
    ```
*   **Fields:** `gif` (optional) triggers GIF-style autoplay playback.

---

### Audio Message (Voice Note)
*   **Body Schema:**
    ```json
    {
      "to": "254700000000@s.whatsapp.net",
      "message": {
        "type": "audio",
        "url": "https://example.com/audio.mp3",
        "ptt": true
      }
    }
    ```
*   **Fields:** `ptt` (optional) if `true`, sends as a "Push-To-Talk" voice note.

---

### Document Message
*   **Body Schema:**
    ```json
    {
      "to": "254700000000@s.whatsapp.net",
      "message": {
        "type": "document",
        "url": "https://example.com/invoice.pdf",
        "filename": "Invoice_Oct.pdf",
        "mimetype": "application/pdf"
      }
    }
    ```

---

### Sticker Message
*   **Body Schema:**
    ```json
    {
      "to": "254700000000@s.whatsapp.net",
      "message": {
        "type": "sticker",
        "url": "https://example.com/sticker.webp"
      }
    }
    ```

---

### Location Message
*   **Body Schema:**
    ```json
    {
      "to": "254700000000@s.whatsapp.net",
      "message": {
        "type": "location",
        "latitude": -1.286389,
        "longitude": 36.817223,
        "location_name": "Nairobi City",
        "location_address": "City Square, Nairobi"
      }
    }
    ```

---

### Contact Card Message
*   **Body Schema:**
    ```json
    {
      "to": "254700000000@s.whatsapp.net",
      "message": {
        "type": "contact",
        "contact_name": "John Doe",
        "contact_phone": "254700111222"
      }
    }
    ```

---

### Reaction Message
React to an existing message.
*   **Body Schema:**
    ```json
    {
      "to": "254700000000@s.whatsapp.net",
      "message": {
        "type": "reaction",
        "reaction_emoji": "👍",
        "reaction_message_id": "ABC123XYZ",
        "reaction_from_me": false
      }
    }
    ```

---

### Response Shape (Success)
All sending endpoints return a standardized result:
```json
{
  "success": true,
  "data": {
    "message_id": "3EB0ABC123...",
    "to": "254700000000@s.whatsapp.net",
    "type": "text",
    "status": "sent",
    "timestamp": "2023-10-27T10:05:00.000Z"
  }
}
```

---

### Media Upload
You can upload media directly to Zaplink to get a URL for the messaging endpoints.
*   **Endpoint:** `POST /sessions/:id/upload`
*   **Form-Data:** `file` (Binary)
*   **Response:** `{ "success": true, "data": { "url": "...", "type": "image", ... } }`

---

### Known Limitations
*   **File Size:** Maximum 50MB per file.
*   **Mimetypes:** Standard images (jpeg, png, webp, gif), video (mp4, 3gpp), audio (mpeg, ogg, wav), and documents (pdf, docx, xlsx, pptx, zip, txt) are supported.
*   **Rate Limits:** Default is 200 requests / 15 minutes unless otherwise configured for your API key.

---

## Sessions

Manage WhatsApp session lifecycles, including creation, connection (QR/Pairing), and termination.

### List Sessions
Retrieve all sessions owned by the authenticated user, including their live connection status.
*   **Method:** `GET`
*   **Path:** `/sessions`
*   **Query Params:** `limit` (default 50), `offset` (default 0)
*   **Auth:** Bearer JWT or API Key
*   **Response:**
    ```json
    {
      "success": true,
      "data": [
        {
          "id": "uuid",
          "session_id": "zap_s12345",
          "status": "authenticated",
          "isOnline": true
        }
      ]
    }
    ```
*   **Example:**
    ```bash
    curl -X GET "http://localhost:3000/v1/sessions?limit=10" \
         -H "X-API-Key: YOUR_API_KEY"
    ```

### Get Session
Retrieve a single session by its `session_id`.
*   **Method:** `GET`
*   **Path:** `/sessions/:id`
*   **Auth:** Bearer JWT or API Key
*   **Response:**
    ```json
    {
      "success": true,
      "data": {
        "id": "uuid",
        "session_id": "zap_s12345",
        "status": "authenticated",
        "isOnline": true,
        "created_at": "..."
      }
    }
    ```

### Get Session Status
Lightweight endpoint for polling session status without SSE.
*   **Method:** `GET`
*   **Path:** `/sessions/:id/status`
*   **Auth:** Bearer JWT or API Key
*   **Response:**
    ```json
    {
      "success": true,
      "data": {
        "session_id": "zap_s12345",
        "status": "authenticated",
        "isOnline": true
      }
    }
    ```

### Create Session
Initialize a new WhatsApp session socket.
*   **Method:** `POST`
*   **Path:** `/sessions`
*   **Auth:** Bearer JWT or API Key
*   **Body:**
    ```json
    { "name": "Marketing-Phone-1" }
    ```
*   **Response:**
    ```json
    {
      "success": true,
      "data": {
        "session_id": "zap_abc123",
        "status": "pending",
        "created_at": "2023-10-27T10:00:00Z"
      }
    }
    ```
*   **Example:**
    ```bash
    curl -X POST "http://localhost:3000/v1/sessions" \
         -H "X-API-Key: YOUR_API_KEY" \
         -H "Content-Type: application/json" \
         -d '{"name": "MySession"}'
    ```

### Get QR Code (Stream)
Subscribe to a Server-Sent Events (SSE) stream for real-time QR code updates and connection status.
*   **Method:** `GET`
*   **Path:** `/sessions/:id/qr`
*   **Auth:** Query Parameter `token` (JWT) is typically used for EventSource, but headers work for curl.
*   **Events:** `qr`, `connected`, `disconnected`
*   **Example:**
    ```bash
    curl -X GET "http://localhost:3000/v1/sessions/zap_abc123/qr" \
         -H "X-API-Key: YOUR_API_KEY" \
         -H "Accept: text/event-stream"
    ```

### Request Pairing Code
Connect via phone number instead of QR code.
*   **Method:** `POST`
*   **Path:** `/sessions/:id/pairing-code`
*   **Auth:** Bearer JWT or API Key
*   **Body:**
    ```json
    { "phone_number": "254700000000" }
    ```
*   **Response:**
    ```json
    { "success": true, "data": { "code": "ABC-123-DEF" } }
    ```
*   **Example:**
    ```bash
    curl -X POST "http://localhost:3000/v1/sessions/zap_abc123/pairing-code" \
         -H "X-API-Key: YOUR_API_KEY" \
         -H "Content-Type: application/json" \
         -d '{"phone_number": "254712345678"}'
    ```

### Delete Session
Terminate the socket and wipe all session data.
*   **Method:** `DELETE`
*   **Path:** `/sessions/:id`
*   **Auth:** Bearer JWT or API Key
*   **Example:**
    ```bash
    curl -X DELETE "http://localhost:3000/v1/sessions/zap_abc123" \
         -H "X-API-Key: YOUR_API_KEY"
    ```

---

## API Keys

### Create API Key
*   **Method:** `POST`
*   **Path:** `/api-keys`
*   **Auth:** Bearer JWT (usually created via Dashboard)
*   **Body:**
    ```json
    {
      "key_name": "Prod Server",
      "ip_whitelist": ["1.2.3.4/32"],
      "permissions": ["*"],
      "expires_at": "2024-12-31T23:59:59Z"
    }
    ```
*   **Fields:** `expires_at` is an optional ISO 8601 datetime.
*   **Response:**
    ```json
    {
      "success": true,
      "data": {
        "id": "uuid",
        "key_prefix": "zap_abcd1234",
        "apiKey": "zap_abcd1234.FullRawSecretKeyExample",
        "rate_limit": 1000
      }
    }
    ```
*   **Example:**
    ```bash
    curl -X POST "http://localhost:3000/v1/api-keys" \
         -H "X-API-Key: YOUR_API_KEY" \
         -H "Content-Type: application/json" \
         -d '{"key_name": "New Key"}'
    ```

### List API Keys
*   **Method:** `GET`
*   **Path:** `/api-keys`
*   **Query Params:** `limit` (default 50), `offset` (default 0)
*   **Auth:** Bearer JWT or API Key
*   **Example:**
    ```bash
    curl -X GET "http://localhost:3000/v1/api-keys?limit=20" \
         -H "X-API-Key: YOUR_API_KEY"
    ```

### Delete API Key
*   **Method:** `DELETE`
*   **Path:** `/api-keys/:id`
*   **Auth:** Bearer JWT or API Key
*   **Example:**
    ```bash
    curl -X DELETE "http://localhost:3000/v1/api-keys/uuid-here" \
         -H "X-API-Key: YOUR_API_KEY"
    ```

---

## Webhooks

Manage endpoints that receive real-time updates from Zaplink.

### List Webhooks
*   **Method:** `GET`
*   **Path:** `/webhooks`
*   **Auth:** Bearer JWT or API Key
*   **Example:**
    ```bash
    curl -X GET "http://localhost:3000/v1/webhooks" \
         -H "X-API-Key: YOUR_API_KEY"
    ```

### Create Webhook
*   **Method:** `POST`
*   **Path:** `/webhooks`
*   **Auth:** Bearer JWT or API Key
*   **Body:**
    ```json
    {
      "url": "https://yourserver.com/webhook",
      "events": ["message.received", "message.sent"],
      "secret": "my_shared_secret"
    }
    ```
*   **Example:**
    ```bash
    curl -X POST "http://localhost:3000/v1/webhooks" \
         -H "X-API-Key: YOUR_API_KEY" \
         -H "Content-Type: application/json" \
         -d '{"url":"https://example.com/hook", "events":["message.received"]}'
    ```

### Get Webhook
Retrieve a single webhook configuration.
*   **Method:** `GET`
*   **Path:** `/webhooks/:id`
*   **Auth:** Bearer JWT or API Key
*   **Response:**
    ```json
    {
      "success": true,
      "data": {
        "id": "...",
        "url": "...",
        "events": ["..."],
        "is_active": true
      }
    }
    ```

### Update Webhook
*   **Method:** `PATCH`
*   **Path:** `/webhooks/:id`
*   **Auth:** Bearer JWT or API Key
*   **Body:** `{ "url": "...", "events": [...], "is_active": true }`
*   **Example:**
    ```bash
    curl -X PATCH "http://localhost:3000/v1/webhooks/uuid" \
         -H "X-API-Key: YOUR_API_KEY" \
         -H "Content-Type: application/json" \
         -d '{"is_active": false}'
    ```

### Delete Webhook
*   **Method:** `DELETE`
*   **Path:** `/webhooks/:id`
*   **Auth:** Bearer JWT or API Key
*   **Example:**
    ```bash
    curl -X DELETE "http://localhost:3000/v1/webhooks/uuid" \
         -H "X-API-Key: YOUR_API_KEY"
    ```

### Test Webhook
Dispatches a test event to the specified webhook.
*   **Method:** `POST`
*   **Path:** `/webhooks/:id/test`
*   **Auth:** Bearer JWT or API Key
*   **Example:**
    ```bash
    curl -X POST "http://localhost:3000/v1/webhooks/uuid/test" \
         -H "X-API-Key: YOUR_API_KEY"
    ```

### Webhook Logs
Retrieve delivery audit logs.
*   **Method:** `GET`
*   **Path:** `/webhooks/logs`
*   **Query Params:** `limit` (default 50), `offset` (default 0)
*   **Auth:** Bearer JWT or API Key
*   **Example:**
    ```bash
    curl -X GET "http://localhost:3000/v1/webhooks/logs?limit=10" \
         -H "X-API-Key: YOUR_API_KEY"
    ```

---

## Webhook Events

| Event Type | Trigger | Payload Description |
| :--- | :--- | :--- |
| `message.received` | Inbound message to a connected session. | Contains message content and sender details. |
| `message.sent` | Outbound message sent from a connected session. | Confirmation of message sent via Zaplink or phone. |
| `webhook.test` | Triggered by the `/webhooks/:id/test` endpoint. | Test message with timestamp. |

---

## Webhook Delivery

### Signature Verification
To ensure payloads originate from Zaplink, verify the `X-WA-Signature` header using HMAC-SHA256 with your webhook's `secret`.

**Headers:**
*   `X-WA-Signature`: HMAC-SHA256 hash of the JSON body using the webhook secret.
*   `X-WA-Event`: The type of event being delivered.

**Node.js Verification Example:**
```javascript
const crypto = require('crypto');

function verifyWebhook(payload, signature, secret) {
  const hmac = crypto.createHmac('sha256', secret);
  const digest = hmac.update(JSON.stringify(payload)).digest('hex');
  return crypto.timingSafeEqual(Buffer.from(signature), Buffer.from(digest));
}
```

### Retry Behavior
Zaplink uses BullMQ for resilient delivery:
*   **Attempts:** Max 5 delivery attempts.
*   **Backoff:** Exponential backoff strategy.
*   **Timeout:** Webhook endpoints must respond within 10 seconds.

---

## Payload Field Reference

All message events use a consistent payload structure:

| Field | Type | Description |
| :--- | :--- | :--- |
| `sessionId` | `string` | The ID of the Zaplink session (e.g., `zap_...`). |
| `messageId` | `string` | The unique WhatsApp message ID (`msg.key.id`). |
| `from` | `string` | The sender's phone number JID (normalised). |
| `fromLid` | `string` | The sender's raw WhatsApp LID (if available). |
| `fromMe` | `boolean` | `true` if the message was sent from the connected device. |
| `type` | `string` | The categorized message type (see "Message Types" below). |
| `text` | `string` | The message body or caption (null for non-text media without captions). |
| `timestamp` | `number` | Unix timestamp in seconds. |
| `raw` | `object` | The full original Baileys `proto.IWebMessageInfo` object. |

> [!NOTE]
> Zaplink increasingly uses `resolveFromJid` logic to ensure the `from` field contains the phone-number JID (`@s.whatsapp.net`) even when the primary ID is an LID. The raw LID is preserved in `fromLid`.

---

## Message Types

| Type | Description |
| :--- | :--- |
| `text` | Standard conversation or extended text message. |
| `image` | Image media message. |
| `audio` | Audio/Voice note message. |
| `video` | Video media message. |
| `document` | Document or file message. |
| `sticker` | Sticker message. |
| `location` | Location sharing message. |
| `contact` | Contact (VCard) sharing message. |
| `reaction` | Message reaction event. |
| `unknown` | Unsupported or system-level message. |

---

## Error Codes

| Code | HTTP Status | Description |
| :--- | :--- | :--- |
| `MISSING_TOKEN` | 401 | Authorization header is missing or malformed. |
| `INVALID_TOKEN` | 401 | The provided Bearer token or API key is invalid/not found. |
| `API_KEY_EXPIRED` | 401 | The API key has passed its expiration date. |
| `API_KEY_IP_BLOCKED` | 403 | Client IP address is not in the key's whitelist. |
| `INTERNAL_ERROR` | 500 | An unexpected server-side error occurred. |
| `NOT_FOUND` | 404 | The requested resource (webhook, session, key) does not exist. |
| `VALIDATION_ERROR` | 400 | Request body failed schema validation. |
| `DATABASE_ERROR` | 500 | Failed to communicate with the primary database. |
| `SESSION_NOT_FOUND` | 404 | No active socket/session found for the provided ID. |
| `SESSION_PAIRING_FAILED` | 500 | Failed to generate a pairing code via Baileys. |
| `SESSION_ALREADY_EXISTS` | 400 | Attempted to connect an already authenticated session. |
| `SESSION_QR_EXPIRED` | 404 | The requested QR code has expired or is not yet ready. |
| `WEBHOOK_INACTIVE` | 400 | The operation cannot be performed on an inactive webhook. |
| `RATE_LIMIT_EXCEEDED` | 429 | Too many requests. Check `Retry-After` header. |
