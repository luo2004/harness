---
name: 'claude-vision'
description: '当当前模型没有原生识图能力，需要分析、描述或识别本地图片或网络图片，或消息中出现 Saved attachments 时，使用内置 vision.js 通过 OpenAI 兼容的视觉模型返回文字描述。'
---

# Claude Vision

这个技能用于给没有原生识图能力的模型提供识图能力。核心脚本是同目录下的 `vision.js`：它读取图片，将图片发送到视觉模型 API，再返回文字描述。

## 触发场景

- 用户分享本地图片路径或网络图片 URL
- 消息中出现 `Saved attachments:` 并列出图片
- 用户要求分析、描述或识别图片内容

## 执行规则

1. 仅在当前模型没有原生识图能力时使用本技能；如果模型具备原生识图能力，优先使用原生能力。
2. 当前模型没有原生识图能力时，不要用 `Read` 读取图片内容。对每张图片依次执行脚本，并在所有图片处理完成后再回复用户。
3. 本地图片使用：

   ```text
   node "${CLAUDE_SKILL_DIR}/vision.js" "<图片绝对路径>" "请用中文描述这张图片的内容"
   ```

4. 网络图片使用：

   ```text
   node "${CLAUDE_SKILL_DIR}/vision.js" --url "<图片链接>" "请用中文描述这张图片的内容"
   ```

5. 将脚本输出作为图片理解结果，再结合用户的问题作答；不要在脚本尚未处理完所有图片前提前回复。

## 配置说明

- `vision.js` 读取 `DASHSCOPE_API_KEY`、`VISION_MODEL`，以及可选的 `DASHSCOPE_BASE_URL` 环境变量，也支持同目录 `.env` 文件。
- 本次安装保留了 `vision.js` 中的 API Key 和模型占位符，未填写任何密钥、模型名或自定义 Base URL。需要使用时由用户自行配置。
- 不要在回复、日志或提交中输出 API Key。

## 随附文件

- `vision.js`：识图脚本
- `README.md`：上游项目说明
- `CLAUDE.md`：上游项目原始规则