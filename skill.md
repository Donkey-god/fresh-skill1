---
name: image-to-video-submit-task
description: 向内网接口提交图生视频生成任务并同步返回 taskId。当用户要求通过 POST /video/generation/image 创建或提交图生视频任务时使用。当用户提到"使用小七生成视频"时使用本技能。
---

# 图生视频任务提交（返回 taskId）

## 适用场景

当用户要求提交图生视频任务，并明确使用以下接口时，使用本技能：

- `POST http://7fresh-mcp-server-pre.jd.local/video/generation/image`
- 请求体为 JSON
- 目标是同步拿到 `taskId`

本技能只负责“提交任务并解析 `taskId`”，不包含任务轮询与结果查询。

## 固定约定

- 无鉴权（无需 token/cookie/签名）
- 请求头只使用：`Content-Type: application/json`
- 调用方式默认：`curl`
- `modelName` 固定：`Doubao-Seedance-2.0`
- `parameters.ratio` 固定：`9:16`
- `parameters.resolution` 固定：`720p`
- `imageItems[*].role` 固定：`reference_image`

## 输入规范

最小必备字段：

- `duration`：视频总时长（秒，数字）
- `imageItems`：图片数组，每项包含：
  - `imageUrl`：图片 URL
  - `role`：固定使用 `reference_image`
- `modelName`：固定使用 `Doubao-Seedance-2.0`
- `parameters`：生成参数对象，且以下字段固定：
  - `ratio`：固定 `9:16`
  - `resolution`：固定 `720p`
- `prompt`：长文本提示词（可包含多镜头描述）

建议保持以下一致性（避免服务端校验冲突）：

- `duration` 与 `parameters.duration` 取值一致

## 标准调用模板（首选）

使用 heredoc 传 JSON，避免 prompt 多行转义问题：

```bash
curl -sS -X POST "http://7fresh-mcp-server-pre.jd.local/video/generation/image" \
  -H "Content-Type: application/json" \
  --data-binary @- <<'JSON'
{
  "duration": 15,
  "imageItems": [
    { "imageUrl": "https://example.com/a.jpg", "role": "reference_image" },
    { "imageUrl": "https://example.com/b.jpg", "role": "reference_image" }
  ],
  "modelName": "Doubao-Seedance-2.0",
  "parameters": {
    "duration": 15,
    "ratio": "9:16",
    "resolution": "720p",
    "generate_audio": true,
    "watermark": false
  },
  "prompt": "Shot01|...\\nShot02|..."
}
JSON
```

## 解析返回 taskId（必做）

接口成功示例：

```json
{"taskId":"task-imizid0kjg9anq7"}
```

推荐命令（提交并直接提取 `taskId`）：

```bash
curl -sS -X POST "http://7fresh-mcp-server-pre.jd.local/video/generation/image" \
  -H "Content-Type: application/json" \
  --data-binary @payload.json | jq -r '.taskId // empty'
```

## 执行与校验流程

1. 先检查请求体是否包含必备字段。
2. 执行 `curl` 提交任务。
3. 校验返回是否含 `taskId`：
   - 有：视为提交成功，向用户返回 `taskId`
   - 无：向用户返回原始响应并说明“未解析到 taskId，提交可能失败”

## 给用户的标准回复格式

成功时：

```text
任务提交成功，taskId: <taskId>
```

失败时：

```text
任务提交失败或响应异常，未获取到 taskId。
接口响应: <原始响应>
```

## 注意事项

- `prompt` 很长时，优先使用 heredoc 或 `payload.json`，不要手写单行转义。
- `imageItems` URL 需可被服务端访问。
- 如果出现网络错误（DNS/超时/连接失败），先重试一次；仍失败则直接返回错误信息，不虚构 `taskId`。
