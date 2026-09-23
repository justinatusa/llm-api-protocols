# 第三方厂商：开了哪些入口，差在哪

> 同一个意思在各家叫什么，见 [字段对照 · 第三方同义字段](../field-atlas.md)；这里是入口、同名不同义和各家要点。
> 方括号里是来源键，见文末；“冲突”“未核实”汇总在 [冲突与未核实](conflicts.md)。

## 1. 厂商 × 协议入口

| 厂商 | Chat 兼容 | Responses | Anthropic 兼容 | 其他 / 官方推荐 | 来源 |
|---|---|---|---|---|---|
| DeepSeek | `https://api.deepseek.com/chat/completions`；beta 功能改用 `/beta` | 有，**不存对话**（参考页和定价页两个模型都支持，指南的兼容表只写 `deepseek-flash`） | `https://api.deepseek.com/anthropic` | 按前缀 + 后缀补全（FIM）的接口在 `/beta/completions` | [ds-home][ds-resp][ds-price][ds-anth] |
| 智谱 GLM | 原生 API 本身就是 Chat 格式：`/api/paas/v4/chat/completions`（国内 `open.bigmodel.cn`，海外 `api.z.ai`） | 有：`https://open.bigmodel.cn/api/v1`；`store` 默认 false，设为 true 后可用 `previous_response_id`（7 天） | `https://open.bigmodel.cn/api/anthropic` | 海外对应 `api.z.ai/api/v1`、`api.z.ai/api/anthropic`；编程套餐 Chat 另有 `…/api/coding/paas/v4`，用错地址就用不上套餐额度 | [zp-chat][zp-resp][zp-claude-compat][zp-tools][zai-tools] |
| Kimi | `https://api.moonshot.ai/v1`（国内 `.cn`，key 不能跨区用） | 有，只支持 `kimi-k3`，响应始终 `store:false` | `https://api.moonshot.ai/anthropic`；参考页 `model` 枚举只有 `kimi-k3`，但 Claude Code 接入指南在同一地址用 `kimi-k2.7-code`（冲突） | Kimi Code 是另一个产品，地址和 key 都不同：国内 `https://api.kimi.com/coding/v1`（OpenAI）、`https://api.kimi.com/coding/`（Anthropic），海外把域名换成 `api.kimi.ai` | [kimi-ov][kimi-cn][kimi-resp][kimi-msg][kimi-cc][kimi-code] |
| xAI Grok | `https://api.x.ai/v1/chat/completions`，对比页标为 Deprecated | **推荐**；存对话（`store` 默认开，保留 30 天）；不支持 `background` | `/v1/messages` 已完全废弃 | 另有 gRPC（`xai-sdk`）和 WebSocket | [xai-cmp][xai-resp][xai-legacy][xai-grpc] |
| Qwen（阿里云百炼） | `{host}/compatible-mode/v1`，host 按地域和工作空间区分 | 有，存对话（`previous_response_id` 保留 7 天） | `{host}/apps/anthropic/v1/messages`，只支持文档列出的模型；没有 `/v1/models`，Claude Code 探测模型列表会 404 | 另有 DashScope 原生格式（`input.messages` / `output.choices`）：外壳不同，但也是每次重发历史、工具参数为 JSON 字符串，归 Chat Completions 那一格；服务端记历史要用另一套应用 API | [ali-tg][ali-resp][ali-anth][ali-native][ali-app] |
| MiniMax | `https://api.minimax.io/v1` | 有：`POST /v1/responses`（请求字段里没有 `previous_response_id`） | `https://api.minimax.io/anthropic`，**官方推荐**，只支持 M3 和 M2.x 系列 | 国内地址：`https://api.minimax.cn`（`/v1`、`/anthropic`） | [mm-tg][mm-resp][mm-anth][mm-fc][mm-cn-anth][mm-cn-tg][mm-cn-tg2] |
| 豆包（火山方舟） | `https://ark.cn-beijing.volces.com/api/v3/chat/completions` | 有：`/api/v3/responses`，支持 `previous_response_id` | 按量付费：base 为 `https://ark.cn-beijing.volces.com/api/compatible`；Agent Plan（`…/api/plan`）、Coding Plan（`…/api/coding`）另有地址 | `model` 可以填接入点 ID | [ark-chat][ark-resp][ark-anth][ark-plan][ark-coding] |
| Azure（Foundry） | 老版 `/openai/deployments/{部署名}/…?api-version=`；新版 `/openai/v1/` | 有 | Claude 模型另有 `https://<resource>.services.ai.azure.com/anthropic/v1/messages` | `model` 填的是部署名，不是模型名 | [az-ref][az-v1][az-claude] |
| 大厂把自家模型包成 Chat Completions | Anthropic `https://api.anthropic.com/v1/`；Gemini `/v1beta/openai/`（beta） | — | — | Anthropic 这层静默忽略一批参数（同名不同义）；Gemini 这层只写明图片**生成**接口忽略未列参数，`reasoning_effort` 会映射到思考 | [an-oai][g-oai] |
| 聚合（OpenRouter） | 有 | 有，但不记历史：传 `previous_response_id` 或 `store:true` 直接 400 | 有 `/messages` | 默认忽略目标模型不支持的参数（同名不同义） | [or-err][or-resp][or-msg] |
| 自托管（vLLM、Ollama） | 有 | vLLM：默认不存，`store=true` 被静默当成 false，只有同时 `background` 且没设 `VLLM_ENABLE_RESPONSES_API_STORE=1` 时才 400；Ollama：不记历史 | — | 大量参数不支持或要走 `extra_body` | [vllm][vllm-resp][ollama] |

## 2. 名字一样、行为不一样

| 项目 | 各家实际行为 | 来源 |
|---|---|---|
| 采样参数 | DeepSeek 的 penalty 在 Chat 参考页标为已废弃、传了不生效，思考模式下 `temperature` 也不生效（都不报错）；Kimi 这些值是固定的，传别的值**报错**；xAI 推理模型传 penalty 或 `stop` **报错**；MiniMax 忽略 penalty 和 `logit_bias`；智谱 `temperature` 上限是 1 | [ds-chat][ds-think][kimi-models][xai-reason][mm-oai][zp-chat] |
| 长度上限 | DeepSeek Chat 只认 `max_tokens`（Responses 用 `max_output_tokens`）；xAI Chat 的 `max_completion_tokens` **不含**推理和函数调用 token，Responses 的 `max_output_tokens` 含推理；豆包多数模型的 `max_tokens` 只限制最终回答，不限制思考 token | [ds-chat][ds-resp][xai-models][xai-resp][ark-chat] |
| Responses 不记历史时怎么表现 | DeepSeek：`previous_response_id` 等参数静默忽略；Kimi：响应里恒为 `previous_response_id: null`；OpenRouter：传了直接 400 | [ds-resp][kimi-resp][or-resp] |
| 被静默忽略 | Anthropic 的 OpenAI 兼容层忽略 `response_format` `strict` `reasoning_effort`；OpenRouter 默认忽略目标模型不支持的参数（`provider.require_parameters` 可改）；DeepSeek Responses 明说“不支持的参数静默忽略”；xAI Responses 的 `metadata` `truncation` 只为兼容保留、不生效 | [an-oai][or-err][ds-resp][xai-resp] |
| Anthropic 兼容入口 | DeepSeek 忽略 `anthropic-version` `cache_control` `budget_tokens`；MiniMax 忽略 `top_k` `stop_sequences`；Kimi 只认顶层 `cache_control`；Kimi 和 Qwen 命中 `stop_sequences` 时 `stop_reason` 仍是 `end_turn`；Qwen 的 thinking `signature` 永远是空字符串 | [ds-anth][mm-anth][kimi-msg][ali-anth] |
| 联网搜索 | Kimi 用 `builtin_function` 的 `$web_search`（2026-10-20 退役）；xAI 用 `web_search` / `x_search` 工具，老的 `search_parameters` 仍在 schema 里；智谱 `web_search` 工具的结果放在响应顶层 `web_search[]` | [kimi-chat][xai-tools][zp-chat] |
| `stop` 上限 | OpenAI Chat 最多 4 个（`o3` / `o4-mini` 不支持）；Gemini 最多 5 个；DeepSeek 最多 16 个；Kimi 最多 5 个、每个 ≤ 32 字节；智谱国内参考页写最多 4 个、Z.ai 参考页写目前只支持 1 个（冲突）；xAI 推理模型传了报错；Anthropic 没写上限；MiniMax 的 Anthropic 入口直接忽略 | [oa-chat][g-gen][ds-chat][kimi-chat][zp-chat][zai-chat][xai-reason][an-msg][mm-anth] |

## 3. 各家要点

**DeepSeek**
- 当前模型名 `deepseek-flash` / `deepseek-v4-pro`（中英文档已一致）；`deepseek-chat` / `deepseek-reasoner` 公告 2026-07-24 停用，之后还能不能调没写 [ds-chat][ds-log]。
- `finish_reason` 多出 `insufficient_system_resource` 和 `aborted` [ds-chat]。
- 排队时非流式会发空行（流式的保活行见 [流式、停止与断开](streaming.md)）；10 分钟还没开始推理就断开。解析器要能跳过空行 [ds-rate]。
- Anthropic 入口把 `claude-opus*` 映射到 `deepseek-v4-pro`，其他（含未知名字）映射到 `deepseek-flash`；不支持 `document`、`redacted_thinking` 块；流式事件名、错误体、`stop_reason` 取值都没写 [ds-anth]。

**智谱 GLM**
- Anthropic 兼容入口官方只有一句“某些场景下仍存在差异，但不影响整体兼容性”，没有字段清单；`anthropic-version`、`cache_control`、`tool_choice`、服务端工具、图片/文档块、流式事件、错误体全都没写 [zp-claude-compat]。
- 力度映射有好几张表，互相对不上：编程套餐页写 Claude Code 的 `thinking.type` / `output_config.effort` 怎么映射（关掉思考变成 `low`，仍会轻量思考）；Responses API 里 `none`/`minimal` 才是不思考；思考指南又分“API 请求”和“编程套餐请求”两套 [zp-coding-model][zp-resp][zp-think]。
- 社区报告（未复测，未核实）：在 `messages` 里放 `role:"system"` 返回 422；不传 `max_tokens` 返回 500；传 `claude-3-opus-*` 会被悄悄换成 `glm-4.7` [zp-gh-74][zp-gh-15]。
- 原生 API：支持 JWT 鉴权；`tool_stream` 流式输出工具参数；`do_sample:false` 时忽略 `temperature` / `top_p` [zp-http][zp-stream-tool][zp-chat]。

**Kimi**
- 续写用 `partial: true`，放在最后一条 assistant 消息上（官方建议别和 `json_object` 同时用，结果可能不符合预期）[kimi-chat]。

**xAI Grok**
- Chat 独有 `deferred: true`：先拿 `request_id`，之后轮询（未完成返回 202 空响应），24 小时内只能取一次；Responses 不支持 `background` [xai-deferred]。
- 用量里多出费用字段 `cost_in_usd_ticks` 和各服务端工具的调用次数 [xai-cost]。

---

[ali-anth]: https://www.alibabacloud.com/help/en/model-studio/anthropic-api-messages
[ali-app]: https://help.aliyun.com/en/model-studio/agent-and-workflow-application-api-reference
[ali-native]: https://www.alibabacloud.com/help/en/model-studio/qwen-api-via-dashscope
[ali-resp]: https://www.alibabacloud.com/help/en/model-studio/qwen-api-via-openai-responses
[ali-tg]: https://www.alibabacloud.com/help/en/model-studio/text-generation
[an-msg]: https://platform.claude.com/docs/en/api/messages
[an-oai]: https://platform.claude.com/docs/en/cli-sdks-libraries/libraries/openai-sdk
[ark-anth]: https://www.volcengine.com/docs/82379/2160841
[ark-chat]: https://www.volcengine.com/docs/82379/1494384
[ark-coding]: https://docs.volcengine.com/docs/ark/coding-plan-personal-faq
[ark-plan]: https://www.volcengine.com/docs/82379/2628970
[ark-resp]: https://docs.volcengine.com/docs/ark/create-model-responses-api
[az-claude]: https://learn.microsoft.com/en-us/azure/foundry/foundry-models/concepts/claude-models
[az-ref]: https://learn.microsoft.com/en-us/azure/foundry/openai/reference
[az-v1]: https://learn.microsoft.com/en-us/azure/foundry/openai/api-version-lifecycle
[ds-anth]: https://api-docs.deepseek.com/guides/anthropic_api
[ds-chat]: https://api-docs.deepseek.com/api/create-chat-completion
[ds-home]: https://api-docs.deepseek.com/
[ds-log]: https://api-docs.deepseek.com/updates
[ds-price]: https://api-docs.deepseek.com/quick_start/pricing
[ds-rate]: https://api-docs.deepseek.com/quick_start/rate_limit
[ds-resp]: https://api-docs.deepseek.com/guides/responses_api
[ds-think]: https://api-docs.deepseek.com/guides/thinking_mode
[g-gen]: https://ai.google.dev/api/generate-content
[g-oai]: https://ai.google.dev/gemini-api/docs/openai
[kimi-cc]: https://platform.kimi.ai/docs/guide/claude-code-kimi
[kimi-chat]: https://platform.kimi.ai/docs/api/chat
[kimi-cn]: https://platform.moonshot.cn/docs/api/overview
[kimi-code]: https://www.kimi.com/code/docs/en/
[kimi-models]: https://platform.kimi.ai/docs/api/models-overview
[kimi-msg]: https://platform.kimi.ai/docs/api/messages
[kimi-ov]: https://platform.kimi.ai/docs/api/overview
[kimi-resp]: https://platform.kimi.ai/docs/api/responses
[mm-anth]: https://platform.minimax.io/docs/api-reference/text-anthropic-api
[mm-cn-anth]: https://platform.minimaxi.com/docs/api-reference/text-chat-anthropic
[mm-cn-tg]: https://platform.minimaxi.com/docs/guides/text-generation
[mm-cn-tg2]: https://platform.minimax.cn/docs/guides/text-generation
[mm-fc]: https://platform.minimax.io/docs/guides/text-m3-function-call
[mm-oai]: https://platform.minimax.io/docs/api-reference/text-openai-api
[mm-resp]: https://platform.minimax.io/docs/api-reference/responses-create
[mm-tg]: https://platform.minimax.io/docs/guides/text-generation
[oa-chat]: https://developers.openai.com/api/reference/resources/chat/subresources/completions/methods/create/
[ollama]: https://docs.ollama.com/api/openai-compatibility
[or-err]: https://openrouter.ai/docs/api/reference/errors-and-debugging
[or-msg]: https://openrouter.ai/docs/api/api-reference/anthropic-messages/create-messages
[or-resp]: https://openrouter.ai/docs/api/reference/responses/error-handling
[vllm]: https://docs.vllm.ai/en/latest/serving/online_serving/openai_compatible_server/
[vllm-resp]: https://docs.vllm.ai/en/latest/api/vllm/entrypoints/openai/responses/serving/
[xai-cmp]: https://docs.x.ai/developers/model-capabilities/text/comparison
[xai-cost]: https://docs.x.ai/developers/cost-tracking
[xai-deferred]: https://docs.x.ai/developers/advanced-api-usage/deferred-chat-completions
[xai-grpc]: https://docs.x.ai/developers/grpc-api-reference
[xai-legacy]: https://docs.x.ai/developers/rest-api-reference/inference/legacy
[xai-models]: https://docs.x.ai/developers/models
[xai-reason]: https://docs.x.ai/developers/model-capabilities/text/reasoning
[xai-resp]: https://docs.x.ai/developers/rest-api-reference/inference/responses
[xai-tools]: https://docs.x.ai/developers/tools/overview
[zai-chat]: https://docs.z.ai/api-reference/llm/chat-completion
[zai-tools]: https://docs.z.ai/devpack/tool/others
[zp-chat]: https://docs.bigmodel.cn/api-reference/%E6%A8%A1%E5%9E%8B-api/%E5%AF%B9%E8%AF%9D%E8%A1%A5%E5%85%A8
[zp-claude-compat]: https://docs.bigmodel.cn/cn/guide/develop/claude/introduction
[zp-coding-model]: https://docs.bigmodel.cn/cn/coding-plan/latest-model
[zp-gh-15]: https://github.com/zai-org/zai-coding-plugins/issues/15
[zp-gh-74]: https://github.com/zai-org/GLM-5/issues/74
[zp-http]: https://docs.bigmodel.cn/cn/guide/develop/http/introduction
[zp-resp]: https://docs.bigmodel.cn/api-reference/response/%E5%88%9B%E5%BB%BA-response
[zp-stream-tool]: https://docs.bigmodel.cn/cn/guide/capabilities/stream-tool
[zp-think]: https://docs.bigmodel.cn/cn/guide/capabilities/thinking
[zp-tools]: https://docs.bigmodel.cn/cn/coding-plan/tool/others
