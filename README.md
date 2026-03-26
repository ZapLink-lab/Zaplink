# 📚 Zaplink WhatsApp API Documentation

**Version:** 1.0.0  
**Base URL:** `https://api.zaplink.co.ke/v1`  
**Support:** [zaplink.info@gmail.com]

---

## 📖 Table of Contents

1. [Getting Started](#getting-started)
2. [Authentication](#authentication)
3. [Rate Limits](#rate-limits)
4. [Error Handling](#error-handling)
5. [Sessions API](#sessions-api)
6. [Messaging API](#messaging-api)
7. [Templates API](#templates-api)
8. [Bulk Messaging API](#bulk-messaging-api)
9. [Scheduled Messages API](#scheduled-messages-api)
10. [Groups API](#groups-api)
11. [Webhooks](#webhooks)
12. [Number Validation](#number-validation)
13. [Analytics](#analytics)
14. [Code Examples](#code-examples)

---

## Getting Started

### Quick Start Guide

1. **Sign up** at [zaplink.co.ke](https://zaplink.co.ke)
2. **Connect your WhatsApp** via QR code or pairing code
3. **Generate an API key** from your dashboard
4. **Start sending messages** using the API

### Requirements

- Active WhatsApp account
- Valid phone number (Kenyan format: +254XXXXXXXXX)
- API key from  Zaplink dashboard

---

## Authentication

Zaplink uses **API keys** for authentication. Include your API key in the request header:

```http
X-API-Key: zaplink_live_your_api_key_here
```

Or as a Bearer token:

```http
Authorization: Bearer zaplink_live_your_api_key_here
```

### Getting Your API Key

1. Log in to [zaplink.co.ke](https://zaplink.co.ke)
2. Navigate to **API Keys**
3. Click **Generate New Key**
4. Copy and store securely (you can always reveal or rotate keys from your dashboard)

### Security Features

- **IP Whitelisting**: Restrict API key usage to specific IP addresses.
- **Permissions (Scopes)**: Limit what an API key can do (e.g., `messages:send`, `sessions:read`).
- **Rotation**: Rotate your API keys with a custom overlap window (default 60 minutes) to ensure zero downtime during key updates.

⚠️ **Never share your API key publicly or commit it to version control**

---

## Rate Limits

Rate limits vary by plan:

| Plan | Requests/Minute | Messages/Day | Sessions |
|------|----------------|--------------|----------|
| **Free** | 5 | 50 | 1 |
| **Starter** | 50 | 600 | 1 |
| **Professional** | 100 | 6,000 | 5 |
| **Enterprise** | Custom | Custom | Unlimited |

### Rate Limit Headers

Every API response includes rate limit information:

```http
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 847
X-RateLimit-Reset: 1708012345
```

### Handling Rate Limits

When you exceed the rate limit, you'll receive a `429` error:

```json
{
  "success": false,
  "error": "Rate limit exceeded",
  "retryAfter": 60
}
```

**Best Practice:** Implement exponential backoff when receiving 429 errors.

---

## Error Handling

### Standard Error Response

```json
{
  "success": false,
  "error": "Error message description",
  "code": "ERROR_CODE",
  "details": {
    "field": "additional context"
  }
}
```

### Common Error Codes

| Code | HTTP Status | Description |
|------|-------------|-------------|
| `INVALID_API_KEY` | 401 | API key is invalid or expired |
| `RATE_LIMIT_EXCEEDED` | 429 | Too many requests |
| `SESSION_NOT_FOUND` | 404 | WhatsApp session doesn't exist |
| `SESSION_DISCONNECTED` | 503 | WhatsApp session is not connected |
| `INVALID_PHONE_NUMBER` | 400 | Phone number format is invalid |
| `INSUFFICIENT_QUOTA` | 402 | Message quota exceeded |
| `VALIDATION_ERROR` | 400 | Request validation failed |

---

## Sessions API

Sessions represent your connected WhatsApp accounts.

### List Sessions

Get all your WhatsApp sessions.

**Endpoint:** `GET /sessions`

**Headers:**
```http
X-API-Key: your_api_key
```

**Response:**
```json
{
  "success": true,
  "sessions": [
    {
      "sessionId": "session_abc123",
      "phoneNumber": "+254712345678",
      "status": "connected",
      "name": "Customer Support Line",
      "lastConnected": "2026-02-18T10:30:00Z",
      "messagesSentToday": 247,
      "dailyLimit": 1000
    }
  ]
}
```

### Get Session Details

**Endpoint:** `GET /sessions/:sessionId`

**Response:**
```json
{
  "success": true,
  "session": {
    "sessionId": "session_abc123",
    "phoneNumber": "+254712345678",
    "status": "connected",
    "name": "Customer Support Line",
    "qrCode": null,
    "health": {
      "score": 95,
      "connectionState": "connected",
      "lastDisconnect": null
    },
    "limits": {
      "messagesPerMinute": 15,
      "messagesPerDay": 1000,
      "currentDayUsage": 247
    }
  }
}
```

### Session Status Values

- `pending` - Session created, waiting for QR scan
- `connecting` - Connecting to WhatsApp
- `connected` - Active and ready to send messages
- `disconnected` - Temporarily disconnected
- `banned` - Account has been banned
- `failed` - Connection failed

---

## Messaging API

Send WhatsApp messages programmatically.

### Send Text Message

**Endpoint:** `POST /send/message`

**Headers:**
```http
X-API-Key: your_api_key
Content-Type: application/json
```

**Request Body:**
```json
{
  "sessionId": "session_abc123",
  "to": "254712345678",
  "message": {
    "text": "Hello! This is a test message from Zaplink."
  }
}
```

**Response:**
```json
{
  "success": true,
  "messageId": "msg_xyz789",
  "timestamp": 1708012345678,
  "status": "sent"
}
```

### Send Image

**Endpoint:** `POST /send/message`

The `url` must be a **public URL** (your own storage, S3, CDN, etc.).

**Request Body:**
```json
{
  "sessionId": "session_abc123",
  "to": "254712345678",
  "message": {
    "image": {
      "url": "https://example.com/image.jpg",
      "caption": "Check out this image!"
    }
  }
}
```

### Send Document

**Request Body:**
```json
{
  "sessionId": "session_abc123",
  "to": "254712345678",
  "message": {
    "document": {
      "url": "https://example.com/invoice.pdf",
      "filename": "Invoice_12345.pdf",
      "caption": "Your invoice is attached"
    }
  }
}
```

### Send Video

**Request Body:**
```json
{
  "sessionId": "session_abc123",
  "to": "254712345678",
  "message": {
    "video": {
      "url": "https://example.com/video.mp4",
      "caption": "Watch this video"
    }
  }
}
```

### Media: use your own URL

Zaplink fetches the file from your public URL and deliver to the customer. This is the fastest and recommended method.

- **Flow:** You pass `message: { image: { url: "https://your-cdn.com/photo.jpg" } }`. We download from your URL and deliver to the customer.
- **Constraints:** Max file size **16 MB**. Supported: images, video, audio, documents (PDF, Office, etc.).

| **Media Support** | Send any media by providing a public URL in the `message` object. Maximum file size is **16 MB**. Supported types include `image`, `video`, `document`, and `audio`. |

### Send Location

**Request Body:**
```json
{
  "sessionId": "session_abc123",
  "to": "254712345678",
  "message": {
    "location": {
      "latitude": -1.286389,
      "longitude": 36.817223,
      "name": "Nairobi City Center",
      "address": "Nairobi, Kenya"
    }
  }
}
```

### Send Buttons (Interactive)

**Request Body:**
```json
{
  "sessionId": "session_abc123",
  "to": "254712345678",
  "message": {
    "buttons": {
      "text": "Choose an option:",
      "buttons": [
        {
          "id": "option_1",
          "text": "View Order"
        },
        {
          "id": "option_2",
          "text": "Contact Support"
        },
        {
          "id": "option_3",
          "text": "Cancel"
        }
      ]
    }
  }
}
```

### Phone Number Format

✅ **Correct formats:**
- `254712345678` (without +)
- `+254712345678` (with +)
- `254712345678@s.whatsapp.net` (full JID)

❌ **Incorrect formats:**
- `0712345678` (local format)
- `712345678` (missing country code)

---

## Templates API

Create reusable message templates with variables.

### Create Template

**Endpoint:** `POST /templates`

**Request Body:**
```json
{
  "name": "Order Confirmation",
  "category": "transactional",
  "messageType": "text",
  "templateData": {
    "type": "text",
    "text": "Hello {{name}}, your order #{{order_id}} totaling KES {{amount}} has been confirmed. Estimated delivery: {{delivery_date}}."
  }
}
```

**Response:**
```json
{
  "success": true,
  "templateId": "template_def456",
  "message": "Template created successfully"
}
```

### List Templates

**Endpoint:** `GET /templates`

**Query Parameters:**
- `category` (optional) - Filter by category
- `limit` (optional) - Results per page (default: 20)
- `offset` (optional) - Pagination offset

**Response:**
```json
{
  "success": true,
  "templates": [
    {
      "id": "template_def456",
      "name": "Order Confirmation",
      "category": "transactional",
      "messageType": "text",
      "variables": ["name", "order_id", "amount", "delivery_date"],
      "usageCount": 1247,
      "createdAt": "2026-01-15T08:00:00Z"
    }
  ],
  "total": 15
}
```

### Send Using Template

**Endpoint:** `POST /send/template`

**Request Body (Custom Template):**
```json
{
  "sessionId": "session_abc123",
  "templateId": "template_def456",
  "to": "254712345678",
  "variables": {
    "name": "John Doe",
    "order_id": "ORD-12345"
  }
}
```

**Request Body (Library Template):**
```json
{
  "sessionId": "session_abc123",
  "template_name": "Welcome Message",
  "to": "254712345678",
  "variables": {
    "name": "Jane"
  }
}
```

**Response:**
```json
{
  "success": true,
  "messageId": "msg_xyz789"
}
```

### Template Categories

- `marketing` - Promotional messages
- `transactional` - Order confirmations, receipts
- `notification` - Alerts, reminders
- `support` - Customer service responses

---

## Bulk Messaging API

Send messages to multiple recipients efficiently.

### Send Bulk Messages

**Endpoint:** `POST /send/bulk`

**Request Body:**
```json
{
  "sessionId": "session_abc123",
  "messages": [
    {
      "to": "254712345678",
      "message": {
        "text": "Hello John, your appointment is tomorrow at 10 AM."
      }
    },
    {
      "to": "254787654321",
      "message": {
        "text": "Hello Jane, your order has been shipped."
      }
    }
  ],
  "settings": {
    "delayBetweenMessages": 5000,
    "maxRetries": 3
  }
}
```

**Response:**
```json
{
  "success": true,
  "campaignId": "campaign_hij789",
  "totalRecipients": 2,
  "message": "Bulk campaign started"
}
```

⚠️ **Important:** Bulk messages are automatically spread over time to prevent bans:
- 100 messages → 30 minutes
- 500 messages → 2 hours
- 2,000 messages → 4 hours
- 10,000+ messages → 6 hours

### Send Bulk with Template

**Endpoint:** `POST /send/template-bulk`

**Request Body:**
```json
{
  "sessionId": "session_abc123",
  "templateId": "template_def456",
  "recipients": [
    {
      "phone": "254712345678",
      "variables": {
        "name": "John",
        "order_id": "12345",
        "amount": "5000",
        "delivery_date": "Feb 20"
      }
    },
    {
      "phone": "254787654321",
      "variables": {
        "name": "Jane",
        "order_id": "67890",
        "amount": "7500",
        "delivery_date": "Feb 21"
      }
    }
  ]
}
```

**Response:**
```json
{
  "success": true,
  "campaignId": "campaign_klm012",
  "totalRecipients": 2
}
```

### Get Campaign Status

**Endpoint:** `GET /bulk/campaigns/:campaignId`

**Response:**
```json
{
  "success": true,
  "campaign": {
    "id": "campaign_hij789",
    "name": "Monthly Newsletter",
    "status": "processing",
    "totalRecipients": 1000,
    "sentCount": 647,
    "deliveredCount": 612,
    "failedCount": 8,
    "progress": 64.7,
    "estimatedCompletion": "2026-02-18T16:30:00Z"
  }
}
```

### Campaign Status Values

- `draft` - Campaign created but not started
- `queued` - Waiting to start
- `processing` - Currently sending
- `paused` - Temporarily paused
- `completed` - All messages sent
- `failed` - Campaign failed

---

## Scheduled Messages API

Schedule messages for future delivery.

### Schedule a Message

**Endpoint:** `POST /scheduler/messages`

**Request Body:**
```json
{
  "sessionId": "session_abc123",
  "recipientPhone": "254712345678",
  "messageType": "text",
  "messageData": {
    "text": "Reminder: Your appointment is in 1 hour!"
  },
  "scheduledAt": "2026-02-20T09:00:00Z"
}
```

**Response:**
```json
{
  "success": true,
  "messageId": "scheduled_nop345",
  "scheduledAt": "2026-02-20T09:00:00Z",
  "message": "Message scheduled successfully"
}
```

### List Scheduled Messages

**Endpoint:** `GET /scheduler/messages`

**Query Parameters:**
- `status` - Filter by status (scheduled, sent, failed, cancelled)
- `sessionId` - Filter by session

**Response:**
```json
{
  "success": true,
  "messages": [
    {
      "id": "scheduled_nop345",
      "recipientPhone": "254712345678",
      "messageType": "text",
      "scheduledAt": "2026-02-20T09:00:00Z",
      "status": "scheduled",
      "createdAt": "2026-02-18T10:00:00Z"
    }
  ],
  "total": 5
}
```

### Update Scheduled Message

**Endpoint:** `PUT /scheduler/messages/:messageId`

**Request Body:**
```json
{
  "scheduledAt": "2026-02-20T10:00:00Z",
  "messageData": {
    "text": "Updated reminder text"
  }
}
```

### Cancel Scheduled Message

**Endpoint:** `DELETE /scheduler/messages/:messageId`

**Response:**
```json
{
  "success": true,
  "message": "Scheduled message cancelled"
}
```

---

## Groups API

Manage WhatsApp groups programmatically.

### Create Group

**Endpoint:** `POST /groups`

**Request Body:**
```json
{
  "sessionId": "session_abc123",
  "name": "Team Updates",
  "participants": [
    "254712345678",
    "254787654321"
  ]
}
```

**Response:**
```json
{
  "success": true,
  "groupId": "group_qrs678",
  "groupJid": "120363123456789@g.us"
}
```

### List Groups

**Endpoint:** `GET /groups?sessionId=session_abc123`

**Response:**
```json
{
  "success": true,
  "groups": [
    {
      "id": "group_qrs678",
      "groupJid": "120363123456789@g.us",
      "name": "Team Updates",
      "participantCount": 15,
      "isAdmin": true,
      "createdAt": "2026-01-10T12:00:00Z"
    }
  ]
}
```

### Get Group Details

**Endpoint:** `GET /groups/:groupJid?sessionId=session_abc123`

**Response:**
```json
{
  "success": true,
  "group": {
    "groupJid": "120363123456789@g.us",
    "name": "Team Updates",
    "description": "Daily team communications",
    "participantCount": 15,
    "isAdmin": true,
    "participants": [
      {
        "jid": "254712345678@s.whatsapp.net",
        "phone": "254712345678",
        "role": "admin"
      },
      {
        "jid": "254787654321@s.whatsapp.net",
        "phone": "254787654321",
        "role": "member"
      }
    ]
  }
}
```

### Send Message to Group

**Endpoint:** `POST /groups/:groupJid/messages`

**Request Body:**
```json
{
  "sessionId": "session_abc123",
  "message": {
    "text": "Hello team! Meeting at 3 PM today."
  }
}
```

### Add Participants

**Endpoint:** `POST /groups/:groupJid/participants`

**Request Body:**
```json
{
  "sessionId": "session_abc123",
  "participants": [
    "254798765432",
    "254756789012"
  ]
}
```

### Remove Participants

**Endpoint:** `DELETE /groups/:groupJid/participants`

**Request Body:**
```json
{
  "sessionId": "session_abc123",
  "participants": [
    "254798765432"
  ]
}
```

### Get Group Invite Link

**Endpoint:** `GET /groups/:groupJid/invite?sessionId=session_abc123`

**Response:**
```json
{
  "success": true,
  "inviteLink": "https://chat.whatsapp.com/AbCdEfGhIjKlMnOp"
}
```

---

## Webhooks

Receive real-time notifications about WhatsApp events.

### Setting Up Webhooks

1. Go to **Dashboard → Webhooks**
2. Click **Create Webhook**
3. Enter your endpoint URL
4. Select events to subscribe to
5. Copy the webhook secret for verification

### Webhook Events

| Event | Description |
|-------|-------------|
| `message:received` | Incoming message received |
| `message:sent` | Outgoing message sent successfully |
| `message:delivered` | Message delivered to recipient |
| `message:read` | Message read by recipient |
| `message:failed` | Message failed to send |
| `session:connected` | WhatsApp session connected |
| `session:disconnected` | WhatsApp session disconnected |
| `session:qr` | New QR code generated |

### Webhook Payload Structure

All webhooks follow this format:

```json
{
  "event": "message:received",
  "timestamp": 1708012345678,
  "data": {
    "sessionId": "session_abc123",
    "messageId": "msg_xyz789",
    "from": "254712345678@s.whatsapp.net",
    "text": "Hello! I need help with my order.",
    "type": "text",
    "message": {
      "conversation": "Hello! I need help with my order."
    },
    "timestamp": 1708012345678
  }
}
```

### Webhook Headers

Every webhook request includes these headers:

```http
Content-Type: application/json
X-Zaplink-Signature: sha256_hmac_signature
X-Zaplink-Event: message:received
X-Zaplink-Timestamp: 1708012345678
User-Agent: Zaplink-Webhooks/1.0
```

### Verifying Webhook Signatures

**Always verify webhook signatures** to ensure requests are from Zaplink:

**Node.js Example:**
```javascript
const crypto = require('crypto');

function verifyWebhook(payload, signature, secret) {
  const hmac = crypto
    .createHmac('sha256', secret)
    .update(JSON.stringify(payload))
    .digest('hex');
  
  return crypto.timingSafeEqual(
    Buffer.from(signature),
    Buffer.from(hmac)
  );
}

// Usage in Express
app.post('/webhooks/zaplink', (req, res) => {
  const signature = req.headers['x-zaplink-signature'];
  const isValid = verifyWebhook(req.body, signature, YOUR_WEBHOOK_SECRET);
  
  if (!isValid) {
    return res.status(401).send('Invalid signature');
  }
  
  // Process webhook
  const { event, data } = req.body;
  
  switch (event) {
    case 'message:received':
      handleIncomingMessage(data);
      break;
    case 'message:delivered':
      updateMessageStatus(data);
      break;
  }
  
  res.status(200).send('OK');
});
```

**Python Example:**
```python
import hmac
import hashlib
import json

def verify_webhook(payload, signature, secret):
    computed = hmac.new(
        secret.encode(),
        json.dumps(payload).encode(),
        hashlib.sha256
    ).hexdigest()
    
    return hmac.compare_digest(signature, computed)

# Usage in Flask
@app.route('/webhooks/zaplink', methods=['POST'])
def webhook():
    signature = request.headers.get('X-Zaplink-Signature')
    
    if not verify_webhook(request.json, signature, WEBHOOK_SECRET):
        return 'Invalid signature', 401
    
    event = request.json.get('event')
    data = request.json.get('data')
    
    if event == 'message:received':
        handle_incoming_message(data)
    
    return 'OK', 200
```

### Webhook Event Examples

#### Message Received

```json
{
  "event": "message:received",
  "timestamp": 1708012345678,
  "data": {
    "sessionId": "session_abc123",
    "messageId": "msg_abc123",
    "from": "254712345678@s.whatsapp.net",
    "text": "Hello! I need help.",
    "type": "text",
    "message": {
      "conversation": "Hello! I need help."
    },
    "timestamp": 1708012345678
  }
}
```

#### Message Delivered

```json
{
  "event": "message:delivered",
  "timestamp": 1708012345678,
  "data": {
    "sessionId": "session_abc123",
    "messageId": "msg_xyz789",
    "to": "254712345678@s.whatsapp.net",
    "deliveredAt": 1708012345678
  }
}
```

#### Message Read

```json
{
  "event": "message:read",
  "timestamp": 1708012345678,
  "data": {
    "sessionId": "session_abc123",
    "messageId": "msg_xyz789",
    "to": "254712345678@s.whatsapp.net",
    "readAt": 1708012345678
  }
}
```

#### Session Disconnected

```json
{
  "event": "session:disconnected",
  "timestamp": 1708012345678,
  "data": {
    "sessionId": "session_abc123",
    "reason": "Connection lost",
    "reconnecting": true
  }
}
```

### Webhook Retry Policy

If your endpoint fails, Zaplink will retry:

- **Attempt 1:** Immediately
- **Attempt 2:** After 1 second
- **Attempt 3:** After 5 seconds
- **Attempt 4:** After 15 seconds

After 4 failed attempts, the webhook is marked as failed.

### Webhook Best Practices

1. **Respond Quickly:** Return `200 OK` within 5 seconds
2. **Process Async:** Queue webhook processing for later
3. **Verify Signatures:** Always validate webhook authenticity
4. **Handle Idempotency:** Use `messageId` to prevent duplicate processing
5. **Log Failures:** Track failed webhooks for debugging

**Example Async Processing:**
```javascript
app.post('/webhooks/zaplink', async (req, res) => {
  // Verify signature
  if (!verifyWebhook(req.body, req.headers['x-zaplink-signature'], SECRET)) {
    return res.status(401).send('Invalid signature');
  }
  
  // Respond immediately
  res.status(200).send('OK');
  
  // Process asynchronously
  processWebhookAsync(req.body).catch(err => {
    console.error('Webhook processing error:', err);
  });
});

async function processWebhookAsync(payload) {
  // Add to queue, database, etc.
  await queue.add('webhook', payload);
}
```

---

## Number Validation

Verify if a phone number is registered on WhatsApp.

### Validate Single Number

**Endpoint:** `POST /validation/number`

**Request Body:**
```json
{
  "sessionId": "session_abc123",
  "phoneNumber": "254712345678"
}
```

**Response:**
```json
{
  "success": true,
  "valid": true,
  "exists": true,
  "jid": "254712345678@s.whatsapp.net"
}
```

### Validate Multiple Numbers

**Endpoint:** `POST /validation/numbers`

**Request Body:**
```json
{
  "sessionId": "session_abc123",
  "phoneNumbers": [
    "254712345678",
    "254787654321",
    "254798765432"
  ]
}
```

**Response:**
```json
{
  "success": true,
  "results": [
    {
      "phone": "254712345678",
      "valid": true,
      "exists": true,
      "jid": "254712345678@s.whatsapp.net"
    },
    {
      "phone": "254787654321",
      "valid": true,
      "exists": false
    },
    {
      "phone": "254798765432",
      "valid": false,
      "error": "Invalid phone number format"
    }
  ]
}
```

---

## Analytics

Get insights about your messaging activity.

### Get Overview

**Endpoint:** `GET /analytics/overview?sessionId=session_abc123&days=30`

**Response:**
```json
{
  "success": true,
  "analytics": {
    "totalMessages": 24567,
    "deliveryRate": 98.7,
    "readRate": 76.4,
    "failedMessages": 124,
    "averageResponseTime": 180,
    "period": {
      "from": "2026-01-18T00:00:00Z",
      "to": "2026-02-18T23:59:59Z"
    }
  }
}
```

### Get Message Trends

**Endpoint:** `GET /analytics/messages?sessionId=session_abc123&days=7`

**Response:**
```json
{
  "success": true,
  "trends": [
    {
      "date": "2026-02-18",
      "sent": 847,
      "delivered": 832,
      "read": 641,
      "failed": 5
    },
    {
      "date": "2026-02-17",
      "sent": 923,
      "delivered": 915,
      "read": 712,
      "failed": 3
    }
  ]
}
```

---

## Code Examples

### Node.js SDK

```javascript
const ZaplinkClient = require('@zaplink/sdk');

const zaplink = new ZaplinkClient({
  apiKey: 'zaplink_live_your_api_key'
});

// Send a message
async function sendMessage() {
  try {
    const result = await zaplink.messages.send({
      sessionId: 'session_abc123',
      to: '254712345678',
      message: {
        text: 'Hello from Node.js!'
      }
    });
    
    console.log('Message sent:', result.messageId);
  } catch (error) {
    console.error('Error:', error.message);
  }
}

// Send bulk messages
async function sendBulk() {
  const messages = [
    { to: '254712345678', message: { text: 'Hi John!' } },
    { to: '254787654321', message: { text: 'Hi Jane!' } }
  ];
  
  const result = await zaplink.messages.sendBulk({
    sessionId: 'session_abc123',
    messages
  });
  
  console.log('Campaign started:', result.campaignId);
}

// Listen to webhooks
const express = require('express');
const app = express();

app.post('/webhooks/zaplink', express.json(), (req, res) => {
  if (!zaplink.webhooks.verify(req.body, req.headers['x-zaplink-signature'])) {
    return res.status(401).send('Invalid signature');
  }
  
  const { event, data } = req.body;
  
  if (event === 'message:received') {
    console.log('New message from:', data.from);
    console.log('Message:', data.message.conversation);
  }
  
  res.status(200).send('OK');
});

app.listen(3000);
```

### Python SDK

```python
from zaplink import ZaplinkClient

zaplink = ZaplinkClient(api_key='zaplink_live_your_api_key')

# Send a message
def send_message():
    result = zaplink.messages.send(
        session_id='session_abc123',
        to='254712345678',
        message={'text': 'Hello from Python!'}
    )
    print(f'Message sent: {result["messageId"]}')

# Send with template
def send_template():
    result = zaplink.templates.send(
        session_id='session_abc123',
        template_id='template_def456',
        to='254712345678',
        variables={
            'name': 'John',
            'order_id': '12345'
        }
    )
    print(f'Message sent: {result["messageId"]}')

# Handle webhooks (Flask)
from flask import Flask, request
app = Flask(__name__)

@app.route('/webhooks/zaplink', methods=['POST'])
def webhook():
    if not zaplink.webhooks.verify(request.json, request.headers.get('X-Zaplink-Signature')):
        return 'Invalid signature', 401
    
    event = request.json.get('event')
    data = request.json.get('data')
    
    if event == 'message:received':
        print(f'New message from: {data["from"]}')
        print(f'Message: {data["message"]["conversation"]}')
    
    return 'OK', 200

if __name__ == '__main__':
    app.run(port=3000)
```

### PHP Example

```php
<?php

require 'vendor/autoload.php';

use Zaplink\ZaplinkClient;

$zaplink = new ZaplinkClient([
    'apiKey' => 'zaplink_live_your_api_key'
]);

// Send a message
function sendMessage($zaplink) {
    $result = $zaplink->messages->send([
        'sessionId' => 'session_abc123',
        'to' => '254712345678',
        'message' => [
            'text' => 'Hello from PHP!'
        ]
    ]);
    
    echo "Message sent: " . $result['messageId'];
}

// Handle webhook
$payload = json_decode(file_get_contents('php://input'), true);
$signature = $_SERVER['HTTP_X_ZAPLINK_SIGNATURE'];

if (!$zaplink->webhooks->verify($payload, $signature)) {
    http_response_code(401);
    exit('Invalid signature');
}

$event = $payload['event'];
$data = $payload['data'];

if ($event === 'message:received') {
    echo "New message from: " . $data['from'];
}

http_response_code(200);
echo 'OK';
```

### cURL Examples

**Send a message:**
```bash
curl -X POST https://api.zaplink.co.ke/v1/send/message \
  -H "X-API-Key: zaplink_live_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "sessionId": "session_abc123",
    "to": "254712345678",
    "message": {
      "text": "Hello from cURL!"
    }
  }'
```

**Send with template:**
```bash
curl -X POST https://api.zaplink.co.ke/v1/send/template \
  -H "X-API-Key: zaplink_live_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "sessionId": "session_abc123",
    "templateId": "template_def456",
    "to": "254712345678",
    "variables": {
      "name": "John",
      "order_id": "12345"
    }
  }'
```

**Validate number:**
```bash
curl -X POST https://api.zaplink.co.ke/v1/validation/number \
  -H "X-API-Key: zaplink_live_your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "sessionId": "session_abc123",
    "phoneNumber": "254712345678"
  }'
```

---

## Testing

### Sandbox Environment

Test your integration without sending real messages:

**Base URL:** `https://sandbox.zaplink.co.ke/v1`

### Test API Key

Generate a test API key from your dashboard (Settings → API Keys → Create Test Key).

Test keys are prefixed with `zaplink_test_`.

### Test Phone Numbers

Use these numbers for testing (they won't actually receive messages):

- `254700000001` - Always succeeds
- `254700000002` - Always fails (simulate errors)
- `254700000003` - Slow delivery (30s delay)

---

## Support

### Documentation
- API Docs: [docs.zaplink.co.ke](https://docs.zaplink.co.ke)
- Guides: [zaplink.co.ke/guides](https://zaplink.co.ke/guides)

### Community
- GitHub: [github.com/zaplink-api](https://github.com/zaplink-api)

### Contact
- Email: support@zaplink.co.ke
- Phone: +254 783 123 456 (Mon-Fri, 9AM-5PM EAT)

### Status Page
Check system status: [status.zaplink.co.ke](https://status.zaplink.co.ke)

---

## Changelog

### v1.0.0 (2026-02-18)
- Initial API release
- Sessions management
- Message sending (text, images, documents, videos)
- Templates API
- Bulk messaging
- Scheduled messages
- Groups API
- Webhooks
- Number validation
- Analytics

---

## Legal

### Terms of Service
By using the Zaplink API, you agree to our [Terms of Service](https://zaplink.co.ke/terms).

### Privacy Policy
Read our [Privacy Policy](https://zaplink.co.ke/privacy).

### WhatsApp Terms
You must comply with [WhatsApp's Terms of Service](https://www.whatsapp.com/legal/terms-of-service).

**Important:** Do not use Zaplink for:
- Spam or unsolicited messages
- Harassment or abuse
- Illegal activities
- Violating WhatsApp's policies

---

**Happy Coding! 🚀**

Need help? Email us at zaplink.info@gmail.com.
