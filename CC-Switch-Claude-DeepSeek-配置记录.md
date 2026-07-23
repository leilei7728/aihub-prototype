# CC Switch + Claude Desktop + DeepSeek 配置记录

更新时间：2026-07-03

## 1. 目标链路
Claude Desktop -> CC Switch 本地路由 -> DeepSeek API

## 2. 最终可用的关键配置
### 2.1 Claude 侧（Gateway）
- Gateway base URL: http://127.0.0.1:15721/claude-desktop
- Gateway auth scheme: bearer
- Gateway API key: 使用 CC Switch 网关要求的 key（若未启用鉴权可先留空测试）

### 2.2 CC Switch 侧（DeepSeek 供应商）
- 请求地址: https://api.deepseek.com/anthropic
- API 格式: Anthropic Messages (原生)
- 需要模型映射: 开启
- 模型映射建议:
  - Sonnet -> deepseek-v4-pro 或 deepseek-v4-flash
  - Opus -> deepseek-v4-pro
  - Haiku -> deepseek-v4-flash

### 2.3 CC Switch 路由页状态
- 本地路由: 运行中
- 路由总开关: 开
- 路由应用中的 Claude: 开
- 服务地址: http://127.0.0.1:15721

## 3. 本次故障与修复
- 故障: 切换路由时提示读取 Claude 配置失败，JSON 解析错误。
- 原因: C:\Users\cehua01\.claude\settings.json 为 UTF-8 BOM，CC Switch 严格解析失败。
- 修复: 将 settings.json 改为无 BOM UTF-8 的标准 JSON。

## 4. Claude settings.json 参考模板
```json
{
  "includeCoAuthoredBy": false,
  "env": {
    "ANTHROPIC_BASE_URL": "https://api.deepseek.com/anthropic",
    "ANTHROPIC_AUTH_TOKEN": "你的 DeepSeek Key"
  }
}
```

## 5. 成功判定
- Claude 界面可正常回复。
- 输入框模型可见 deepseek-v4-flash（或你映射的模型）。
- CC Switch 路由页中“当前 Provider”不再是“等待首次请求”。
- 总请求数从 0 开始增加。

## 6. 10 秒自检清单（下次排错）
1. CC Switch 本地路由是否运行中
2. Claude 路由开关是否开启
3. Gateway URL 是否完整为 http://127.0.0.1:15721/claude-desktop
4. DeepSeek 请求地址是否为 https://api.deepseek.com/anthropic
5. 模型映射是否已填写 Sonnet/Opus/Haiku
6. settings.json 是否为无 BOM UTF-8

## 7. 不同模型平台配置方式（补充）
先看平台提供的是哪种接口，再决定 API 格式。

### 7.1 Anthropic 兼容接口
适用：DeepSeek（Anthropic 路径）、部分中转平台。
- API 格式: Anthropic Messages
- 请求地址: 平台给的 Anthropic 兼容地址
- 在 CC Switch 中开启模型映射（Sonnet/Opus/Haiku）

### 7.2 OpenAI 兼容接口
适用：Qwen、GLM、Kimi、硅基流动、OpenAI、多数本地推理服务。
- API 格式: OpenAI Chat Completions
- 请求地址: 平台给的 OpenAI 兼容 base URL
- 直接填平台模型名（通常不需要 Sonnet/Opus/Haiku 映射）

### 7.3 常见平台速查
| 平台 | 常见接口类型 | 常见 base URL | CC Switch 建议 |
|---|---|---|---|
| DeepSeek | Anthropic / OpenAI 都可 | https://api.deepseek.com/anthropic 或 https://api.deepseek.com | Claude 场景优先用 Anthropic Messages |
| 通义千问（DashScope） | OpenAI 兼容 | https://dashscope.aliyuncs.com/compatible-mode/v1 | 选 OpenAI Chat Completions |
| 智谱 GLM | OpenAI 兼容 | https://open.bigmodel.cn/api/paas/v4 | 选 OpenAI Chat Completions |
| Kimi（Moonshot） | OpenAI 兼容 | https://api.moonshot.cn/v1 | 选 OpenAI Chat Completions |
| 硅基流动 | OpenAI 兼容 | https://api.siliconflow.cn/v1 | 选 OpenAI Chat Completions |
| 本地 Ollama / LM Studio | OpenAI 兼容（常见） | 本机地址（如 127.0.0.1） | 选 OpenAI Chat Completions |

## 8. 安全提醒
你本次会话中曾暴露过 DeepSeek Key，建议：
- 立即在 DeepSeek 控制台作废旧 key
- 新建 key 后同步更新到 CC Switch 与 Claude 配置
