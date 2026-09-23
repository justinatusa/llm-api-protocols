# 错误、重试、限流、批处理

> 本页是各入口的错误体、状态码、重试规则、限流响应头、鉴权和批处理。
> 方括号里是来源键，见文末；“冲突”“未核实”汇总在 [冲突与未核实](conflicts.md)。

## 1. 错误与重试

| 入口 | 错误体 | 特别的状态码 | 限流响应头 | 重试 | 请求 ID | 来源 |
|---|---|---|---|---|---|---|
| OpenAI | `{error:{type, message, param, code}}` | 额度用完也是 429；503 `server_is_overloaded` | `x-ratelimit-{limit,remaining,reset}-{requests,tokens}`，`Retry-After` | Python SDK 默认重试 2 次（408、409、429、≥500），先读 `retry-after-ms` 再读 `retry-after`，要等超过 120 秒就不重试 | 响应头 `x-request-id`；可自带 `X-Client-Request-Id` | [oa-errors][oa-rl][oa-py-client] |
| Anthropic | `{type:"error", error:{type, message}, request_id}` | 402 计费、413 过大、**529** 过载、504 超时 | `anthropic-ratelimit-*`，`retry-after`（花费上限触发的 429 不带） | SDK 默认重试 2 次（408、409、429、≥500），先读 `retry-after-ms` 再读 `retry-after`，没有 120 秒上限 | 响应头 `request-id` = 错误体 `request_id` | [an-errors][an-rl][an-sdk-client] |
| Gemini `generateContent` | `{error:{code, message, status, details[]}}` | 402 预付费用完（别重试）、429、503 | 没写 | 排查页写 Python SDK 最多重试 4 次；SDK 源码默认不重试，要传 `HttpRetryOptions`（冲突） | 没有请求 ID 头；响应体有 `responseId` | [g-err][g-trouble][g-sdk-client] |
| Gemini Interactions | `{error:{code:"invalid_request", message}}`；流里是 `error` 事件 | 402（别重试）、416、429、**499** 取消、503、504 | 没写 | 文档建议 429、503 退避重试 | 没写 | [g-int-err] |
| DeepSeek | JSON 结构没写 | **402** 余额不足、**422** 参数错 | 没写 | 没写 | 没写 | [ds-err] |
| 智谱 | `{error:{code:"1302", message}}`，`code` 是数字字符串 | 429 分 `1302` 超限、`1305` 过载（不用 529）、`1113` 欠费 | 没写 | 没写 | 请求体可带 `request_id`，会原样返回 | [zp-code][zp-rl] |
| Kimi | `{error:{type, message}}` | 余额不足也是 429；499 客户端断开；超时返回 HTML 504 | 只有过载时给 `Retry-After` | 按 `Retry-After` 退避 | Anthropic 入口错误体有 `request_id` | [kimi-err] |
| xAI | 子字段没写 | 422；deferred 未完成返回 202 | 没写 | gRPC SDK 只对 `UNAVAILABLE` 重试 | 没写 | [xai-debug][xai-rl][xai-sdk] |
| Qwen | 与 OpenAI 相同的 `{error:{message, type, param, code}}` | 跨地域 key 返回 401；429；503 | 没写 | 没写 | 响应头或体里有，头名没写 | [ali-compat][ali-err] |
| MiniMax | 业务码表（如 1002 限流、1008 余额不足）；OpenAI 入口响应里有 `base_resp.status_code`；Anthropic 入口用 Anthropic 错误格式 | Anthropic 入口列了 413、429、**529** | 没写 | 业务码表只写“稍后重试” | Anthropic 入口错误体有 `request_id` | [mm-err][mm-oai-chat][mm-anth-chat] |
| 豆包 | 错误码表只列 Type / Code / Message，没有 JSON 示例；官方 SDK 读 `error.code` / `param` / `type` | 欠费是 403 不是 402；429 为 `RateLimitExceeded.*` | 没写 | 官方 SDK 默认重试 2 次，不读 `Retry-After` | SDK 读响应头 `X-Client-Request-Id` | [ark-err][ark-sdk] |
| Azure OpenAI | `code`、`message`、`param`、`type`、`inner_error` | 429 | `x-ratelimit-*`；等待时间在 `retry-after-ms`（毫秒） | 按 `retry-after-ms` 等 | `apim-request-id` | [az-quota][az-chat-ref] |

**幂等**：所有入口的推理接口都没写 `Idempotency-Key`，重试要自己防重复。OpenAI 只在 Agents 的一个接口上定义了它；豆包的批处理任务按对象路径去重，属于控制面 [oa-openapi-gh][ark-ctrl]。

## 2. 鉴权、限流方式、批处理

| 入口 | 鉴权 | 限流方式 | 批处理 / 实时 | 来源 |
|---|---|---|---|---|
| OpenAI | 见 [字段对照 · 请求骨架](../field-atlas.md) | 每分钟请求数（RPM）/ 每分钟 token（TPM） | `POST /v1/batches`，24 小时，5 折；实时语音用 Realtime（WebRTC / WebSocket 会话，单次最长 60 分钟） | [oa-rl][oa-batch][oa-realtime] |
| Anthropic | 见 [字段对照 · 请求骨架](../field-atlas.md) | RPM + 输入 TPM + 输出 TPM | `POST /v1/messages/batches`，5 折 | [an-rl][an-batch] |
| Gemini | 见 [字段对照 · 请求骨架](../field-atlas.md) | 没写剩余额度头 | `generateContent` 用 `:batchGenerateContent`，5 折；Interactions 暂无；实时语音用 Live API（WebSocket，仅 v1beta） | [g-batch][g-int-mig][g-live] |
| DeepSeek | Bearer；Anthropic 入口用 `x-api-key` | 按**并发数**（Flash 2500 / Pro 500） | 没写 | [ds-rate] |
| 智谱 | Bearer（key 形如 `id.secret`）或 JWT | 按并发 | `/api/paas/v4/batches`，5 折 | [zp-http][zp-rl][zp-batch] |
| Kimi | Bearer | 并发 + RPM + TPM + 每日 token（TPD），按充值档位；TPM 按“输入 + `max_completion_tokens`”算，不按实际输出 | `/v1/batches`，6 折（`kimi-k3` 不支持） | [kimi-limits][kimi-intro][kimi-batch] |
| xAI | Bearer | 每个模型按 RPS + TPM | `/v1/batches`，部分文本模型 8 折，不承诺 24 小时 | [xai-rl][xai-batch] |
| Qwen | Bearer；key 跨地域用会 401 | RPM / TPM，另按秒限制（突发可能在分钟额度内就 429） | 文件批处理 5 折 | [ali-compat][ali-rl][ali-batch] |
| MiniMax | Bearer（Anthropic 入口也可用 `x-api-key`，同时传时 Bearer 优先） | RPM / TPM | 没写 | [mm-anth] |
| 豆包 | Bearer API key（AK/SK 可经 `GetApiKey` 换临时 key） | 有 RPM / TPM（从错误码可见），额度表没写 | 控制面 `CreateBatchInferenceJob`，按在线价 5 折 | [ark-auth][ark-api][ark-err][ark-batch] |
| Azure OpenAI | `api-key` 或 Entra Bearer | 不是按整分钟放行，而是每 1 秒 / 10 秒各卡一次，突发会提前 429 | `/openai/v1/batches`，24 小时，比 Global Standard 便宜 50% | [az-quota][az-v1][az-batch] |

---

[ali-batch]: https://www.alibabacloud.com/help/en/model-studio/batch-interfaces-compatible-with-openai/
[ali-compat]: https://www.alibabacloud.com/help/en/model-studio/compatibility-of-openai-with-dashscope
[ali-err]: https://help.aliyun.com/zh/model-studio/error-code
[ali-rl]: https://www.alibabacloud.com/help/en/model-studio/rate-limit
[an-batch]: https://platform.claude.com/docs/en/build-with-claude/batch-processing
[an-errors]: https://platform.claude.com/docs/en/api/errors
[an-rl]: https://platform.claude.com/docs/en/api/rate-limits
[an-sdk-client]: https://github.com/anthropics/anthropic-sdk-python/blob/main/src/anthropic/_base_client.py
[ark-api]: https://api.volcengine.com/api-docs/75110
[ark-auth]: https://www.volcengine.com/docs/82379/1298459
[ark-batch]: https://www.volcengine.com/docs/82379/1399517
[ark-ctrl]: https://api.volcengine.com/api-docs/?serviceCode=ark&version=2024-01-01
[ark-err]: https://www.volcengine.com/docs/82379/1099476
[ark-sdk]: https://github.com/volcengine/volcengine-python-sdk
[az-batch]: https://learn.microsoft.com/en-us/azure/foundry/openai/how-to/batch
[az-chat-ref]: https://learn.microsoft.com/en-us/rest/api/microsoft-foundry/azureopenai/chat
[az-quota]: https://learn.microsoft.com/en-us/azure/foundry/openai/how-to/quota
[az-v1]: https://learn.microsoft.com/en-us/azure/foundry/openai/api-version-lifecycle
[ds-err]: https://api-docs.deepseek.com/quick_start/error_codes
[ds-rate]: https://api-docs.deepseek.com/quick_start/rate_limit
[g-batch]: https://ai.google.dev/gemini-api/docs/batch-api
[g-err]: https://ai.google.dev/gemini-api/docs/generate-content/api-errors
[g-int-err]: https://ai.google.dev/gemini-api/docs/api-errors
[g-int-mig]: https://ai.google.dev/gemini-api/docs/migrate-to-interactions
[g-live]: https://ai.google.dev/api/live
[g-sdk-client]: https://github.com/googleapis/python-genai/blob/main/google/genai/_api_client.py
[g-trouble]: https://ai.google.dev/gemini-api/docs/troubleshooting
[kimi-batch]: https://platform.kimi.ai/docs/guide/use-batch-api
[kimi-err]: https://platform.kimi.ai/docs/api/errors
[kimi-intro]: https://platform.kimi.ai/docs/introduction
[kimi-limits]: https://platform.kimi.ai/docs/pricing/limits
[mm-anth]: https://platform.minimax.io/docs/api-reference/text-anthropic-api
[mm-anth-chat]: https://platform.minimax.io/docs/api-reference/text-chat-anthropic
[mm-err]: https://platform.minimax.io/docs/api-reference/errorcode
[mm-oai-chat]: https://platform.minimax.io/docs/api-reference/text-chat-openai
[oa-batch]: https://developers.openai.com/api/docs/guides/batch
[oa-errors]: https://developers.openai.com/api/docs/guides/error-codes
[oa-openapi-gh]: https://github.com/openai/openai-openapi/blob/master/openapi.yaml
[oa-py-client]: https://github.com/openai/openai-python/blob/main/src/openai/_base_client.py
[oa-realtime]: https://developers.openai.com/api/docs/guides/realtime-conversations
[oa-rl]: https://developers.openai.com/api/docs/guides/rate-limits
[xai-batch]: https://docs.x.ai/developers/advanced-api-usage/batch-api
[xai-debug]: https://docs.x.ai/developers/debugging
[xai-rl]: https://docs.x.ai/developers/rate-limits
[xai-sdk]: https://github.com/xai-org/xai-sdk-python/blob/main/src/xai_sdk/client.py
[zp-batch]: https://docs.bigmodel.cn/cn/guide/tools/batch
[zp-code]: https://docs.bigmodel.cn/cn/api/api-code
[zp-http]: https://docs.bigmodel.cn/cn/guide/develop/http/introduction
[zp-rl]: https://docs.bigmodel.cn/cn/api/rate-limit
