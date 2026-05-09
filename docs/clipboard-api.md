# 云剪切板后端接口文档

## Base URL

```
/api/clipboard
```

> **前端配置**: 在 `src/config.ts` 中的 `clipboardConfig.apiBaseUrl` 可修改后端 API 地址（构建时配置）。

## 接口列表

---

### 1. 获取内容列表

```
GET /api/clipboard/items
```

**Query Parameters:**

| 参数  | 类型   | 必填 | 说明                   |
|-------|--------|------|------------------------|
| seed  | string | 是   | 种子名称，默认 "public" |
| limit | number | 否   | 返回数量限制，默认 50   |
| offset | number | 否   | 分页偏移，默认 0        |

**Response:**

```json
{
  "success": true,
  "data": [
    {
      "id": "uuid",
      "seed": "public",
      "type": "text",
      "content": "文本内容",
      "createdAt": "2024-01-01T00:00:00Z"
    },
    {
      "id": "uuid",
      "seed": "public",
      "type": "image",
      "content": "https://example.com/image.jpg",
      "filename": "photo.png",
      "createdAt": "2024-01-01T00:00:00Z"
    },
    {
      "id": "uuid",
      "seed": "public",
      "type": "file",
      "content": "https://example.com/file.pdf",
      "filename": "document.pdf",
      "createdAt": "2024-01-01T00:00:00Z"
    }
  ],
  "total": 100
}
```

---

### 2. 创建内容

```
POST /api/clipboard/items
```

**Content-Type: application/json (文本) 或 multipart/form-data (文件)**

**Request (文本):**

```json
{
  "seed": "public",
  "type": "text",
  "content": "文本内容"
}
```

**Request (图片/文件):**

| 参数  | 类型   | 说明         |
|-------|--------|--------------|
| seed  | string | 种子名称     |
| type  | string | "image" 或 "file" |
| file  | File   | 文件数据     |

**Response:**

```json
{
  "success": true,
  "data": {
    "id": "uuid",
    "seed": "public",
    "type": "text",
    "content": "文本内容",
    "createdAt": "2024-01-01T00:00:00Z"
  }
}
```

---

### 3. 删除内容

```
DELETE /api/clipboard/items/:id
```

**Response:**

```json
{
  "success": true
}
```

---

## 字段说明

| 字段      | 类型   | 说明                         |
|-----------|--------|------------------------------|
| id        | string | 唯一标识符 (UUID)           |
| seed      | string | 种子名称，用于内容隔离       |
| type      | string | 内容类型: `text` / `image` / `file` |
| content   | string | 文本内容 或 文件URL          |
| filename  | string | 文件名 (仅 image/file 类型)  |
| createdAt | string | 创建时间 (ISO 8601 格式)    |

---

## 前端配置

修改 `src/config.ts` 中的 `clipboardConfig.apiBaseUrl`:

```typescript
export const clipboardConfig: ClipboardConfig = {
    // 后端 API 基础 URL（构建时配置）
    apiBaseUrl: "/api/clipboard",  // 可改为其他域名，如 "https://api.example.com"
};
```

---

## 存储建议

- **数据库**: 使用 MySQL 存储文本记录
- **文件存储**: 图片/文件可存储到本地磁盘或对象存储 (OSS/S3)，返回访问 URL
- **种子隔离**: 相同 seed 的用户可以看到相同的内容
