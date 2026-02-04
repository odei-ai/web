# odei-telegram API Reference

Complete API documentation for the odei-telegram MCP server.

## Table of Contents

- [odei.telegram.chats.list](#odeitelegramchatslist)
- [odei.telegram.messages.fetch](#odeitelegrammessagesfetch)
- [odei.telegram.messages.send](#odeitelegrammessagessend)

---

## odei.telegram.chats.list

No description provided

### Parameters

| Name | Type | Description |
|------|------|-------------|
| `refresh` | boolean (optional> | |
| `limit` | number.int(>.min(1>.max(100> (optional> | |
| `includeEmpty` | boolean (optional> | |

### Example

```typescript
// Call odei.telegram.chats.list
const result = await mcp.callTool('odei.telegram.chats.list', {
});
```

---

## odei.telegram.messages.fetch

No description provided

### Parameters

| Name | Type | Description |
|------|------|-------------|
| `chatId` | union([z.string | |
| `limit` | number.int(>.min(1>.max(200> (optional> | |
| `sinceMessageId` | number.int(> (optional> | |
| `sinceTimestamp` | number.int(> (optional> | |
| `refresh` | boolean (optional> | |

### Example

```typescript
// Call odei.telegram.messages.fetch
const result = await mcp.callTool('odei.telegram.messages.fetch', {
  chatId: /* union([z.string */,
});
```

---

## odei.telegram.messages.send

No description provided

### Parameters

| Name | Type | Description |
|------|------|-------------|
| `chatId` | union([z.string | |
| `text` | string.min(1>.max(4096> | |
| `parseMode` | enum(['MarkdownV2' | |
| `disablePreview` | boolean (optional> | |
| `notify` | boolean (optional> | |
| `replyToMessageId` | number.int(> (optional> | |
| `threadId` | number.int(> (optional> | |
| `dryRun` | boolean (optional> | |

### Example

```typescript
// Call odei.telegram.messages.send
const result = await mcp.callTool('odei.telegram.messages.send', {
  chatId: /* union([z.string */,
  text: /* string.min(1>.max(4096> */,
  parseMode: /* enum(['MarkdownV2' */,
});
```

---
