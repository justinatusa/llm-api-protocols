# 缓存

> 本页列各入口怎么开缓存、有效期多长、用量里看哪个字段。Claude 云托管的有效期例外见 [Claude 云托管](hosted-claude.md)。
> 方括号里是来源键，见文末；“冲突”“未核实”汇总在 [冲突与未核实](conflicts.md)。

| 入口 | 方式 | 怎么标 / 怎么开 | 有效期 | 用量里看哪 | 来源 |
|---|---|---|---|---|---|
| OpenAI Chat / Responses | 默认自动；GPT-5.6+ 可改成只按显式断点写 | `prompt_cache_key`；5.6+ 另有 `prompt_cache_options` 和内容块上的 `prompt_cache_breakpoint`（最多 4 个） | 5.6+ 至少保留 30 分钟（可能更久）；更早的模型是 `in_memory` 或 `24h` | Chat `prompt_tokens_details.cached_tokens`；Responses `input_tokens_details.cached_tokens` / `cache_write_tokens` | [oa-cache] |
| Anthropic | 两种：自动（顶层一个 `cache_control`）或显式断点 | 顶层 `cache_control`（自动放在最后一个可缓存块）或块上 `cache_control:{type:"ephemeral", ttl}`，最多 4 个；不够最小长度时不报错也不缓存 | 5 分钟（默认）/ 1 小时 | `cache_creation_input_tokens`、`cache_read_input_tokens`；`input_tokens` 不含缓存 | [an-cache]；云托管的 TTL 例外见 [Claude 云托管](hosted-claude.md) |
| Gemini `generateContent` | 2.5+ 默认自动；另可显式建缓存 | 先 `POST /v1beta/cachedContents`，再在请求里传 `cachedContent` | 显式默认 1 小时，可自设 | `usageMetadata.cachedContentTokenCount` | [g-cache][g-caching] |
| Gemini Interactions | 只有自动，不支持显式缓存 | — | 不能设 | `usage.total_cached_tokens` | [g-cache2][g-int-api] |
| DeepSeek | 自动磁盘缓存，尽力而为 | 无（Anthropic 入口忽略 `cache_control`） | 不用后几小时到几天清掉 | `prompt_cache_hit_tokens` / `prompt_cache_miss_tokens` | [ds-cache][ds-anth] |
| 智谱 | 自动 | 无 | 没写 | `prompt_tokens_details.cached_tokens` | [zp-cache] |
| Kimi | Chat / Responses 默认写；Anthropic 入口默认只读不写 | `prompt_cache_options:{mode:"implicit", ttl}`、`prompt_cache_key`；Anthropic 入口只认顶层 `cache_control` | 5 分钟 / 1 小时 | Chat `prompt_tokens_details.cached_tokens`（另有顶层 `cached_tokens`）；Anthropic 入口 `cache_read_input_tokens` 等，且 `input_tokens` 不含缓存读写 | [kimi-cache][kimi-msg] |
| xAI | 自动，不保证命中 | 无断点；Chat 请求头 `x-grok-conv-id`、Responses `prompt_cache_key` 可提高命中 | 没写 | `*_tokens_details.cached_tokens` | [xai-cache] |
| Qwen | 默认自动（不能关）；可另加 `cache_control` 显式标记；Responses 另有会话缓存，开启后另两种关闭 | `cache_control:{type:"ephemeral"}`（最多 4 个）；请求头 `x-dashscope-session-cache: enable` | 显式和会话缓存 5 分钟，命中后刷新 | `prompt_tokens_details.cached_tokens` | [ali-cache][ali-resp] |
| MiniMax | OpenAI 入口自动；Anthropic 入口显式 | `cache_control:{type:"ephemeral"}`，最多 4 个 | 显式 5 分钟，命中后刷新 | OpenAI 入口 `prompt_tokens_details.cached_tokens`；Anthropic 入口 `cache_*_input_tokens` | [mm-cache][mm-anth-cache] |
| 豆包 | 显式，要先在控制台开启 | Responses `caching:{type:"enabled"}`（加 `prefix:true` 为前缀缓存）；老 Context API 标题已写“已下线” | Responses 缓存是绝对过期时间，最长 7 天，用了也不延长；Context API 用后会重置有效期 | Responses `input_tokens_details.cached_tokens`；Context API `prompt_tokens_details.cached_tokens` | [ark-cache][ark-stream] |

OpenAI Responses 的 `truncation: "auto"` 和压缩（compaction）会改写历史开头，之前的缓存可能就不命中了 [oa-cache][oa-compact]。

---

[ali-cache]: https://www.alibabacloud.com/help/en/model-studio/context-cache
[ali-resp]: https://www.alibabacloud.com/help/en/model-studio/qwen-api-via-openai-responses
[an-cache]: https://platform.claude.com/docs/en/build-with-claude/prompt-caching
[ark-cache]: https://www.volcengine.com/docs/82379/1396491
[ark-stream]: https://docs.volcengine.com/docs/82379/2123275
[ds-anth]: https://api-docs.deepseek.com/guides/anthropic_api
[ds-cache]: https://api-docs.deepseek.com/guides/kv_cache
[g-cache]: https://ai.google.dev/gemini-api/docs/generate-content/caching
[g-cache2]: https://ai.google.dev/gemini-api/docs/caching
[g-caching]: https://ai.google.dev/api/caching
[g-int-api]: https://ai.google.dev/api/interactions-api
[kimi-cache]: https://platform.kimi.ai/docs/guide/use-context-caching-feature-of-kimi-api
[kimi-msg]: https://platform.kimi.ai/docs/api/messages
[mm-anth-cache]: https://platform.minimax.io/docs/api-reference/anthropic-api-compatible-cache
[mm-cache]: https://platform.minimax.io/docs/api-reference/text-prompt-caching
[oa-cache]: https://developers.openai.com/api/docs/guides/prompt-caching
[oa-compact]: https://developers.openai.com/api/docs/guides/compaction
[xai-cache]: https://docs.x.ai/developers/advanced-api-usage/prompt-caching
[zp-cache]: https://docs.bigmodel.cn/cn/guide/capabilities/cache
