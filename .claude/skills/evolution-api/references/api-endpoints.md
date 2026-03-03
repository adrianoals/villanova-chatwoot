# Evolution API — API Endpoints Reference

Base URL: `http://<server>:8080`

Authentication header: `apikey: YOUR_GLOBAL_API_KEY`

## Table of Contents
1. [Instance Controller](#instance-controller)
2. [Message Controller](#message-controller)
3. [Chat Controller](#chat-controller)
4. [Group Controller](#group-controller)
5. [Label Controller](#label-controller)
6. [Settings Controller](#settings-controller)
7. [Webhook Controller](#webhook-controller)
8. [Chatwoot Controller](#chatwoot-controller)
9. [Webhook Events](#webhook-events)

---

## Instance Controller

### Create Instance
```
POST /instance/create
```
```json
{
  "instanceName": "my-whatsapp",
  "integration": "WHATSAPP-BAILEYS",
  "token": "optional-custom-token",
  "qrcode": true,
  "number": "5511999999999",
  "rejectCall": false,
  "msgCall": "I am busy",
  "groupsIgnore": false,
  "alwaysOnline": false,
  "readMessages": false,
  "readStatus": false,
  "syncFullHistory": false
}
```

`integration` options:
- `WHATSAPP-BAILEYS` — Free, QR code pairing (WhatsApp Web)
- `WHATSAPP-BUSINESS` — Official Meta API (requires `businessId`, `number`, `token`)

### Connect (Get QR Code)
```
GET /instance/connect/{instanceName}
```
Returns QR code as base64 image.

### Connection State
```
GET /instance/connectionState/{instanceName}
```
Response: `{ "state": "open" }` — possible states: `open`, `close`, `connecting`

### Fetch All Instances
```
GET /instance/fetchInstances
```

### Restart Instance
```
POST /instance/restart/{instanceName}
```

### Logout Instance
```
DELETE /instance/logout/{instanceName}
```
Disconnects WhatsApp but keeps the instance configuration.

### Delete Instance
```
DELETE /instance/delete/{instanceName}
```
Permanently removes the instance and all its data.

### Set Presence
```
POST /instance/setPresence/{instanceName}
```
```json
{
  "presence": "available"
}
```
Options: `available`, `unavailable`

---

## Message Controller

### Send Plain Text
```
POST /message/sendText/{instanceName}
```

Format v2 (with options object):
```json
{
  "number": "5511999999999",
  "options": {
    "delay": 1200,
    "presence": "composing",
    "linkPreview": false
  },
  "textMessage": {
    "text": "Hello!"
  }
}
```

Legacy format (also supported):
```json
{
  "number": "5511999999999",
  "text": "Hello!",
  "delay": 1200,
  "linkPreview": true,
  "mentionsEveryOne": false,
  "mentioned": ["5511888888888"]
}
```

### Send Media (image/video/document)
```
POST /message/sendMedia/{instanceName}
```
```json
{
  "number": "5511999999999",
  "mediatype": "image",
  "mimetype": "image/png",
  "caption": "Check this out",
  "fileName": "photo.png",
  "media": "https://example.com/image.png",
  "delay": 1200
}
```
`mediatype`: `image`, `video`, `document`

Media can be a URL or base64 string.

### Send Audio
```
POST /message/sendWhatsAppAudio/{instanceName}
```

Format v2 (with options object):
```json
{
  "number": "5511999999999",
  "options": {
    "delay": 1200,
    "presence": "recording",
    "encoding": true
  },
  "audioMessage": {
    "audio": "https://example.com/audio.ogg"
  }
}
```

Legacy format (also supported):
```json
{
  "number": "5511999999999",
  "audio": "https://example.com/audio.mp3",
  "delay": 1200
}
```

### Send Location
```
POST /message/sendLocation/{instanceName}
```
```json
{
  "number": "5511999999999",
  "latitude": -23.5505,
  "longitude": -46.6333,
  "name": "São Paulo",
  "address": "Av Paulista, 1000"
}
```

### Send Contact
```
POST /message/sendContact/{instanceName}
```
```json
{
  "number": "5511999999999",
  "contact": [
    {
      "fullName": "John Doe",
      "wuid": "5511999999999",
      "phoneNumber": "+5511999999999",
      "organization": "ACME",
      "email": "john@example.com",
      "url": "https://john.com"
    }
  ]
}
```

### Send Reaction
```
POST /message/sendReaction/{instanceName}
```
```json
{
  "key": {
    "id": "message-id",
    "fromMe": false,
    "remoteJid": "5511999999999@s.whatsapp.net"
  },
  "reaction": "👍"
}
```

### Send Poll
```
POST /message/sendPoll/{instanceName}
```
```json
{
  "number": "5511999999999",
  "name": "Best color?",
  "selectableCount": 1,
  "values": ["Red", "Blue", "Green"]
}
```

### Send List
```
POST /message/sendList/{instanceName}
```
```json
{
  "number": "5511999999999",
  "title": "Menu",
  "description": "Choose an option",
  "buttonText": "Options",
  "footerText": "Footer",
  "sections": [
    {
      "title": "Section 1",
      "rows": [
        { "title": "Option A", "description": "Description A", "rowId": "opt_a" },
        { "title": "Option B", "description": "Description B", "rowId": "opt_b" }
      ]
    }
  ]
}
```

### Send Buttons
```
POST /message/sendButtons/{instanceName}
```
```json
{
  "number": "5511999999999",
  "title": "Choose",
  "description": "Description text",
  "footer": "Footer text",
  "buttons": [
    { "type": "reply", "displayText": "Option 1", "id": "btn_1" },
    { "type": "url", "displayText": "Visit site", "url": "https://example.com" },
    { "type": "call", "displayText": "Call us", "phoneNumber": "+5511999999999" },
    { "type": "copy", "displayText": "Copy code", "copyCode": "PROMO123" }
  ]
}
```

### Send Sticker
```
POST /message/sendSticker/{instanceName}
```
```json
{
  "number": "5511999999999",
  "sticker": "https://example.com/sticker.webp"
}
```

### Send Status/Story
```
POST /message/sendStatus/{instanceName}
```

### Send Template (WhatsApp Business only)
```
POST /message/sendTemplate/{instanceName}
```

### Reply to a Message (quoted)
Any send endpoint supports quoting a previous message:
```json
{
  "number": "5511999999999",
  "text": "Replying to your message",
  "quoted": {
    "key": {
      "id": "original-message-id",
      "fromMe": false,
      "remoteJid": "5511999999999@s.whatsapp.net"
    },
    "message": {}
  }
}
```

---

## Chat Controller

| Method | Endpoint | Description |
|--------|----------|-------------|
| `POST` | `/chat/whatsappNumbers/{instanceName}` | Check if numbers are on WhatsApp |
| `POST` | `/chat/markMessageAsRead/{instanceName}` | Mark messages as read |
| `POST` | `/chat/markChatUnread/{instanceName}` | Mark chat as unread |
| `POST` | `/chat/archiveChat/{instanceName}` | Archive/unarchive chat |
| `DELETE` | `/chat/deleteMessageForEveryone/{instanceName}` | Delete message for everyone |
| `POST` | `/chat/fetchProfilePictureUrl/{instanceName}` | Get profile picture URL |
| `POST` | `/chat/getBase64FromMediaMessage/{instanceName}` | Get base64 from media message |
| `POST` | `/chat/updateMessage/{instanceName}` | Edit a sent message |
| `POST` | `/chat/sendPresence/{instanceName}` | Send typing/recording presence |
| `POST` | `/chat/updateBlockStatus/{instanceName}` | Block/unblock user |
| `POST` | `/chat/findContacts/{instanceName}` | Search contacts |
| `POST` | `/chat/findMessages/{instanceName}` | Search messages |
| `POST` | `/chat/findChats/{instanceName}` | List chats |

### Check WhatsApp Numbers
```json
{
  "numbers": ["5511999999999", "5511888888888"]
}
```

### Send Presence (typing indicator)
```json
{
  "number": "5511999999999",
  "presence": "composing"
}
```
Options: `composing`, `recording`, `paused`

---

## Group Controller

| Method | Endpoint | Description |
|--------|----------|-------------|
| `POST` | `/group/create/{instanceName}` | Create a group |
| `POST` | `/group/updateGroupSubject/{instanceName}` | Update group name |
| `POST` | `/group/updateGroupPicture/{instanceName}` | Update group picture |
| `POST` | `/group/updateGroupDescription/{instanceName}` | Update description |
| `GET` | `/group/findGroupInfos/{instanceName}` | Get group info |
| `GET` | `/group/fetchAllGroups/{instanceName}` | List all groups |
| `GET` | `/group/participants/{instanceName}` | Get group participants |
| `GET` | `/group/inviteCode/{instanceName}` | Get invite code |
| `POST` | `/group/updateParticipant/{instanceName}` | Add/remove/promote/demote |
| `POST` | `/group/updateSetting/{instanceName}` | Update group settings |
| `POST` | `/group/toggleEphemeral/{instanceName}` | Toggle disappearing messages |
| `DELETE` | `/group/leaveGroup/{instanceName}` | Leave group |

### Create Group
```json
{
  "subject": "Group Name",
  "participants": ["5511999999999", "5511888888888"],
  "description": "Group description",
  "promoteParticipants": false
}
```

### Update Participant
```json
{
  "groupJid": "120363000000000000@g.us",
  "action": "add",
  "participants": ["5511999999999"]
}
```
`action`: `add`, `remove`, `promote`, `demote`

---

## Label Controller

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/label/findLabels/{instanceName}` | List all labels |
| `POST` | `/label/handleLabel/{instanceName}` | Add/remove label from chat |

---

## Settings Controller

| Method | Endpoint | Description |
|--------|----------|-------------|
| `POST` | `/settings/set/{instanceName}` | Update instance settings |
| `GET` | `/settings/find/{instanceName}` | Get current settings |

```json
{
  "rejectCall": false,
  "msgCall": "I am busy right now",
  "groupsIgnore": false,
  "alwaysOnline": false,
  "readMessages": false,
  "readStatus": false,
  "syncFullHistory": false
}
```

---

## Webhook Controller

### Set Webhook
```
POST /webhook/set/{instanceName}
```
```json
{
  "enabled": true,
  "url": "https://your-server.com/webhook",
  "byEvents": false,
  "base64": false,
  "events": [
    "MESSAGES_UPSERT",
    "CONNECTION_UPDATE",
    "QRCODE_UPDATED",
    "SEND_MESSAGE"
  ]
}
```

### Get Webhook
```
GET /webhook/find/{instanceName}
```

---

## Chatwoot Controller

### Set Chatwoot
```
POST /chatwoot/set/{instanceName}
```
```json
{
  "enabled": true,
  "accountId": "1",
  "token": "chatwoot-admin-access-token",
  "url": "https://chatwoot.example.com",
  "nameInbox": "WhatsApp",
  "signMsg": true,
  "signDelimiter": "\n",
  "reopenConversation": true,
  "conversationPending": true,
  "mergeBrazilContacts": true,
  "importContacts": true,
  "importMessages": true,
  "daysLimitImportMessages": 30,
  "autoCreate": true,
  "organization": "My Company",
  "logo": "https://example.com/logo.png",
  "ignoreJids": []
}
```

### Find Chatwoot
```
GET /chatwoot/find/{instanceName}
```

### Chatwoot Webhook (internal)
```
POST /chatwoot/webhook/{instanceName}
```
This is the endpoint that Chatwoot calls to send agent replies back to WhatsApp. It's configured automatically when `autoCreate: true`.

---

## Webhook Events

### Webhook Payload Structure (MESSAGES_UPSERT example)

```json
{
  "event": "messages.upsert",
  "instance": "my-whatsapp",
  "timestamp": 1633456789,
  "data": {
    "key": {
      "id": "BAE5A5C9B4F3B6D2",
      "fromMe": false,
      "remoteJid": "5511999999999@s.whatsapp.net"
    },
    "pushName": "Customer Name",
    "message": {
      "conversation": "Hello, I have a question"
    },
    "messageType": "conversation",
    "messageTimestamp": 1633456789
  }
}
```

The `data.message` object varies by type:
- Plain text: `{"conversation": "text"}`
- Extended text: `{"extendedTextMessage": {"text": "..."}}`
- Image: `{"imageMessage": {"url": "...", "caption": "..."}}`
- Document: `{"documentMessage": {"url": "...", "fileName": "..."}}`
- Audio: `{"audioMessage": {"url": "..."}}`

### When `webhookByEvents` is enabled

The event name is appended to the webhook URL:
- Base: `https://your-server.com/webhook`
- MESSAGES_UPSERT → `https://your-server.com/webhook/messages-upsert`
- CONNECTION_UPDATE → `https://your-server.com/webhook/connection-update`

### Complete list of events that can be received via webhooks:

| Event | Description |
|-------|-------------|
| `APPLICATION_STARTUP` | API started |
| `INSTANCE_CREATE` | Instance created |
| `INSTANCE_DELETE` | Instance deleted |
| `QRCODE_UPDATED` | QR code generated/updated |
| `MESSAGES_SET` | Messages loaded |
| `MESSAGES_UPSERT` | New message received |
| `MESSAGES_EDITED` | Message edited |
| `MESSAGES_UPDATE` | Message status updated (delivered, read) |
| `MESSAGES_DELETE` | Message deleted |
| `SEND_MESSAGE` | Message sent successfully |
| `SEND_MESSAGE_UPDATE` | Sent message status updated |
| `CONTACTS_SET` | Contacts loaded |
| `CONTACTS_UPSERT` | Contact created/updated |
| `CONTACTS_UPDATE` | Contact updated |
| `PRESENCE_UPDATE` | Online/offline/typing status changed |
| `CHATS_SET` | Chats loaded |
| `CHATS_UPSERT` | Chat created/updated |
| `CHATS_UPDATE` | Chat updated |
| `CHATS_DELETE` | Chat deleted |
| `GROUPS_UPSERT` | Group created/updated |
| `GROUP_UPDATE` | Group settings updated |
| `GROUP_PARTICIPANTS_UPDATE` | Group members changed |
| `CONNECTION_UPDATE` | Connection state changed |
| `LABELS_EDIT` | Label edited |
| `LABELS_ASSOCIATION` | Label associated/removed |
| `CALL` | Call received |
| `REMOVE_INSTANCE` | Instance removed |
| `LOGOUT_INSTANCE` | Instance logged out |
| `ERRORS` | Error occurred |
