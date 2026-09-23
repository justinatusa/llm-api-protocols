# 冲突、未核实与已否定的说法

所有文件里标“冲突”“未核实”的地方都汇总在这里。“冲突”= 两处官方页面说法不一，本文照录两边；“未核实”= 本轮没找到官方原文；“已否定”= 早期调研或常见说法，被现行官方页面推翻。“来源”列的链接全部于 2026-09-23 抓取。

| 类型 | 内容 | 来源 |
|---|---|---|
| 冲突 | OpenAI Chat 的 `store` 默认值：OpenAPI 写 `false`，迁移指南写“新账号默认存储” | [OpenAI OpenAPI 定义][oa-openapi]；[OpenAI 迁移到 Responses 指南][oa-migrate] |
| 冲突 | Responses 的 `encrypted_content`：推理指南和迁移指南说不存对话时默认返回、`include` 不再必需；参考页字段说明写“默认填充”，范围更宽 | [OpenAI 推理指南][oa-reasoning]；[OpenAI 迁移到 Responses 指南][oa-migrate]；[OpenAI Responses 参考][oa-resp] |
| 冲突 | Anthropic 预填返回 400 的起点：一页写 Claude 4.6+，另一页写 Opus 4.7+ | [Anthropic Messages 使用指南][an-working] |
| 冲突 | Anthropic 4.7+ 采样参数：指南写只接受默认值；参考页写 `top_p` ≥ 0.99 可接受、`top_k` 一律拒绝 | [Anthropic Messages 使用指南][an-working]；[Anthropic Messages 参考][an-msg] |
| 冲突 | Anthropic 4.7+ 分词器增幅：一页写 token 数约为原来的 1–1.35 倍（多 0–35%），另一页写约多 30% | [Anthropic token 计数文档][an-count]；[Claude Opus 5 迁移指南][an-opus5-mig] |
| 冲突 | Gemini Python SDK 重试：排查页写默认最多重试 4 次，SDK 源码不传 `HttpRetryOptions` 就不重试 | [Gemini 排查指南][g-trouble]；[Google GenAI Python SDK 源码][g-sdk-client] |
| 冲突 | Gemini 鉴权：API 总览要求 `x-goog-api-key` 头，部分官方 curl 只用 `?key=` | [Gemini API 总览][g-api]；[Gemini generateContent 参考][g-gen] |
| 冲突 | Gemini `Content.role`：写明只能 `user` / `model`，同页函数声明说明又提到 `"function"` | [Gemini generateContent 参考][g-gen] |
| 冲突 | Gemini `responseJsonSchema`：`responseSchema` 标 deprecated，写着关键字清单的 `_responseJsonSchema` 也标 deprecated | [Gemini generateContent 参考][g-gen] |
| 冲突 | Gemini 请求大小：图片页写整请求 ≤ 20 MB，文件页写内联 ≤ 100 MB | [Gemini 图片理解文档][g-image]；[Gemini 文件输入方式文档][g-file] |
| 冲突 | Gemini Interactions：概览说不支持自定义安全设置，参考页仍列出 `safety_settings` | [Gemini Interactions 概览][g-int-ov]；[Gemini Interactions 参考][g-int-api] |
| 冲突 | Claude 在 Bedrock 上的结构化输出：Anthropic 的旧 Bedrock 页笼统写“支持”；AWS 页写 `InvokeModel` 上只有 Sonnet 4.5、Haiku 4.5、Opus 4.5、Opus 4.6 支持，Messages 路由（mantle）不支持、传了 400 | [Anthropic 旧版 Bedrock 说明][an-bedrock]；[Anthropic Bedrock 说明（Messages 路由）][an-bedrock-mantle]；[AWS Bedrock Claude 结构化输出文档][bedrock-so] |
| 冲突 | Bedrock `InvokeModel` 请求上限：Anthropic 页写 20 MB，AWS API 参考写 25000000（没标单位） | [Anthropic 旧版 Bedrock 说明][an-bedrock]；[AWS Bedrock InvokeModel 参考][bedrock-invoke] |
| 冲突 | Vertex 上的 Claude 批处理：Anthropic 页写不支持 Message Batches，Google 另有 Batch predictions | [Anthropic Vertex AI 说明][an-vertex]；[Vertex AI Claude 批量预测文档][vx-claude-batch] |
| 冲突 | Azure Foundry 网页抓取：同一页既说需要 Anthropic 托管，又说 Azure 托管支持 `web_fetch_20250910` | [Azure Foundry Claude 模型说明][az-claude]；[Anthropic 网页抓取工具文档][an-webfetch] |
| 冲突 | 豆包密文被篡改：Chat 参考页写会报 `Invalid signature`，思考页只说无法还原 | [火山方舟 Chat 接口参考][ark-chat]；[火山方舟深度思考文档][ark-think] |
| 冲突 | DeepSeek `reasoning_effort: "none"`：Chat API 页写可关闭思考，思考指南只把 `none` 列在 Responses 下 | [DeepSeek Chat 接口参考][ds-chat]；[DeepSeek 思考模式指南][ds-think] |
| 冲突 | DeepSeek Responses 支持的模型：指南兼容表只写 `deepseek-flash`，参考页和定价页两个模型都支持 | [DeepSeek Responses 指南][ds-resp]；[DeepSeek Responses 接口参考][ds-create-resp]；[DeepSeek 定价页][ds-price] |
| 冲突 | Kimi Anthropic 入口的模型：参考页 `model` 枚举只有 `kimi-k3`，Claude Code 接入指南在同一地址用 `kimi-k2.7-code` | [Kimi Messages 接口参考][kimi-msg]；[Kimi Claude Code 接入指南][kimi-cc] |
| 冲突 | Kimi 流式用量：参考页说 `[DONE]` 前多一块、`choices` 为空；同页示例放在结束块上 | [Kimi Chat 接口参考][kimi-chat]；[Kimi 流式输出指南][kimi-stream] |
| 冲突 | Kimi 非流式超时：错误码页写 900 秒，简介页写 2 小时 | [Kimi 错误码][kimi-err]；[Kimi 平台简介][kimi-intro] |
| 冲突 | xAI 流式用量：计费页说只在最后一块，流式指南示例每块都带 | [xAI 费用追踪文档][xai-cost]；[xAI 流式指南][xai-stream] |
| 冲突 | xAI Chat 的推理内容：对比页说不返回，流式指南的 Chat 示例里有 `delta.reasoning_content` | [xAI 接口对比页][xai-cmp]；[xAI 流式指南][xai-stream] |
| 冲突 | Qwen `tool_choice: "required"`：Chat 参考页、函数调用指南、错误码页三处说法不一 | [阿里云百炼 Chat 兼容接口参考][ali-chat]；[阿里云百炼函数调用指南][ali-fc]；[阿里云百炼错误码][ali-err] |
| 冲突 | Qwen `preserve_thinking` 默认为 true 的型号：Chat 参考页和思考指南列的不一样 | [阿里云百炼 Chat 兼容接口参考][ali-chat]；[阿里云百炼深度思考指南][ali-think] |
| 冲突 | 智谱 `stop`：国内参考页 schema 写最多 4 个，Z.ai 参考页写目前只支持 1 个 | [智谱对话补全参考][zp-chat]；[Z.ai Chat 接口参考][zai-chat] |
| 冲突 | 智谱思考力度映射：编程套餐页、Responses API、思考指南各有一套 | [智谱编程套餐模型说明][zp-coding-model]；[智谱 Responses 接口参考][zp-resp]；[智谱思考模式指南][zp-think] |

| 未核实 | Gemini `generateContent`：不加 `alt=sse` 时流式返回是否是 JSON 数组；`functionCall.args` 会不会跨块增长；服务端是否真的同时接受 camelCase 和 snake_case | [Gemini generateContent 参考][g-gen]；[protobuf JSON 映射规则][proto-json] |
| 未核实 | Gemini Interactions：图片 / 文件 / 音频的大小上限 | [Gemini Interactions 参考][g-int-api] |
| 未核实 | 智谱 Anthropic 兼容入口：除力度映射外，`anthropic-version`、`cache_control`、`tool_choice`、流式事件、错误体等字段支持情况 | [智谱 Claude 兼容说明][zp-claude-compat] |
| 未核实 | DeepSeek：`deepseek-chat` / `deepseek-reasoner` 在 2026-07-24 后还能不能调；Anthropic 入口的流式事件、错误体、`stop_reason`；HTTP 错误的 JSON 键 | [DeepSeek 更新日志][ds-log]；[DeepSeek Anthropic 兼容指南][ds-anth]；[DeepSeek 错误码][ds-err] |
| 未核实 | xAI：错误 JSON 的子字段、限流响应头 | [xAI 调试指南][xai-debug]；[xAI 限流文档][xai-rl] |
| 未核实 | 智谱、Qwen、MiniMax 的限流响应头；Kimi Chat 的剩余额度头 | [智谱限流文档][zp-rl]；[阿里云百炼限流文档][ali-rl]；[MiniMax 错误码][mm-err]；[Kimi 错误码][kimi-err] |
| 未核实 | Kimi Anthropic 入口能否用 `x-api-key`；MiniMax Responses 是否记历史；Vertex 上的 Claude 怎么传 beta 功能 | [Kimi Messages 接口参考][kimi-msg]；[MiniMax Responses 接口参考][mm-resp]；[Anthropic Vertex AI 说明][an-vertex] |
| 未核实 | 豆包推理接口错误的 JSON 外层结构、限流响应头 | [火山方舟错误码][ark-err] |
| 未核实 | 客户端中途断开或取消时，已生成的部分是否计费、生成是否立即停止：各家都没写 | [Kimi 常见问题排查][kimi-trouble] |
| 社区报告（未核实） | 智谱 Anthropic 入口：`messages` 里放 `role:"system"` 返回 422；不传 `max_tokens` 返回 500；`claude-3-opus-*` 被悄悄换成 `glm-4.7` | [社区 issue：GLM-5 #74][zp-gh-74]；[社区 issue：zai-coding-plugins #15][zp-gh-15] |
| 已否定 | “Kimi 没有 Responses API”：早期调研看的是过时的接口列表，现行概览列出了 Responses 和 Messages | [Kimi API 总览][kimi-ov] |
| 已否定 | “DeepSeek 当前模型叫 `deepseek-chat` / `deepseek-reasoner`”：公告 2026-07-24 停用，现行 schema 不再列出 | [DeepSeek 更新日志][ds-log]；[DeepSeek Chat 接口参考][ds-chat] |
| 已否定 | “DeepSeek 中英文文档的模型名、流式用量、函数名长度不一致”：两页 2026-09-19 更新后已一致 | [DeepSeek Chat 接口参考][ds-chat] |
| 已否定 | “vLLM Responses 默认 `store=true` 就会 400”：现行代码是静默当成 false，只有同时 `background` 才 400 | [vLLM Responses 源码文档][vllm-resp] |
| 已否定 | “Anthropic 兼容入口多半只开在编程套餐地址上”：智谱、豆包、DeepSeek、Kimi 都有通用入口 | [智谱 Claude 兼容说明][zp-claude-compat]；[火山方舟 Anthropic 兼容文档][ark-anth]；[DeepSeek Anthropic 兼容指南][ds-anth]；[Kimi Messages 接口参考][kimi-msg] |
| 已否定 | “MiniMax 国内地址是 `api.minimaxi.com`”：现行两处中文站（platform.minimaxi.com、platform.minimax.cn）都写 `api.minimax.cn` | [MiniMax 国内站文本生成指南（minimaxi.com）][mm-cn-tg]；[MiniMax 国内站文本生成指南（minimax.cn）][mm-cn-tg2] |
| 已否定 | “OpenAI SDK 会丢掉未知字段”：Python SDK 会保留在 `model_extra` 里 | [OpenAI Python SDK 参考][oa-py] |

---

[ali-chat]: https://www.alibabacloud.com/help/en/model-studio/qwen-api-via-openai-chat-completions
[ali-err]: https://help.aliyun.com/zh/model-studio/error-code
[ali-fc]: https://www.alibabacloud.com/help/en/model-studio/qwen-function-calling
[ali-rl]: https://www.alibabacloud.com/help/en/model-studio/rate-limit
[ali-think]: https://www.alibabacloud.com/help/en/model-studio/deep-thinking
[an-bedrock]: https://platform.claude.com/docs/en/build-with-claude/claude-on-amazon-bedrock-legacy
[an-bedrock-mantle]: https://platform.claude.com/docs/en/build-with-claude/claude-in-amazon-bedrock
[an-count]: https://platform.claude.com/docs/en/build-with-claude/token-counting
[an-msg]: https://platform.claude.com/docs/en/api/messages
[an-opus5-mig]: https://platform.claude.com/docs/en/models/opus-5/migration-guide
[an-vertex]: https://platform.claude.com/docs/en/build-with-claude/claude-on-vertex-ai
[an-webfetch]: https://platform.claude.com/docs/en/agents-and-tools/tool-use/web-fetch-tool
[an-working]: https://platform.claude.com/docs/en/build-with-claude/working-with-messages
[ark-anth]: https://www.volcengine.com/docs/82379/2160841
[ark-chat]: https://www.volcengine.com/docs/82379/1494384
[ark-err]: https://www.volcengine.com/docs/82379/1099476
[ark-think]: https://www.volcengine.com/docs/82379/1449737
[az-claude]: https://learn.microsoft.com/en-us/azure/foundry/foundry-models/concepts/claude-models
[bedrock-invoke]: https://docs.aws.amazon.com/bedrock/latest/APIReference/API_runtime_InvokeModel.html
[bedrock-so]: https://docs.aws.amazon.com/bedrock/latest/userguide/claude-messages-structured-outputs.html
[ds-anth]: https://api-docs.deepseek.com/guides/anthropic_api
[ds-chat]: https://api-docs.deepseek.com/api/create-chat-completion
[ds-create-resp]: https://api-docs.deepseek.com/api/create-response
[ds-err]: https://api-docs.deepseek.com/quick_start/error_codes
[ds-log]: https://api-docs.deepseek.com/updates
[ds-price]: https://api-docs.deepseek.com/quick_start/pricing
[ds-resp]: https://api-docs.deepseek.com/guides/responses_api
[ds-think]: https://api-docs.deepseek.com/guides/thinking_mode
[g-api]: https://ai.google.dev/api
[g-file]: https://ai.google.dev/gemini-api/docs/generate-content/file-input-methods
[g-gen]: https://ai.google.dev/api/generate-content
[g-image]: https://ai.google.dev/gemini-api/docs/generate-content/image-understanding
[g-int-api]: https://ai.google.dev/api/interactions-api
[g-int-ov]: https://ai.google.dev/gemini-api/docs/interactions-overview
[g-sdk-client]: https://github.com/googleapis/python-genai/blob/main/google/genai/_api_client.py
[g-trouble]: https://ai.google.dev/gemini-api/docs/troubleshooting
[kimi-cc]: https://platform.kimi.ai/docs/guide/claude-code-kimi
[kimi-chat]: https://platform.kimi.ai/docs/api/chat
[kimi-err]: https://platform.kimi.ai/docs/api/errors
[kimi-intro]: https://platform.kimi.ai/docs/introduction
[kimi-msg]: https://platform.kimi.ai/docs/api/messages
[kimi-ov]: https://platform.kimi.ai/docs/api/overview
[kimi-stream]: https://platform.kimi.ai/docs/guide/utilize-the-streaming-output-feature-of-kimi-api
[kimi-trouble]: https://platform.kimi.ai/docs/guide/troubleshooting
[mm-cn-tg]: https://platform.minimaxi.com/docs/guides/text-generation
[mm-cn-tg2]: https://platform.minimax.cn/docs/guides/text-generation
[mm-err]: https://platform.minimax.io/docs/api-reference/errorcode
[mm-resp]: https://platform.minimax.io/docs/api-reference/responses-create
[oa-migrate]: https://developers.openai.com/api/docs/guides/migrate-to-responses
[oa-openapi]: https://platform.openai.com/docs/static/api-definition.yaml
[oa-py]: https://developers.openai.com/api/reference/python/
[oa-reasoning]: https://developers.openai.com/api/docs/guides/reasoning
[oa-resp]: https://developers.openai.com/api/reference/resources/responses/methods/create/
[proto-json]: https://protobuf.dev/programming-guides/json/
[vllm-resp]: https://docs.vllm.ai/en/latest/api/vllm/entrypoints/openai/responses/serving/
[vx-claude-batch]: https://docs.cloud.google.com/gemini-enterprise-agent-platform/models/partner-models/claude/batch
[xai-cmp]: https://docs.x.ai/developers/model-capabilities/text/comparison
[xai-cost]: https://docs.x.ai/developers/cost-tracking
[xai-debug]: https://docs.x.ai/developers/debugging
[xai-rl]: https://docs.x.ai/developers/rate-limits
[xai-stream]: https://docs.x.ai/developers/model-capabilities/text/streaming
[zai-chat]: https://docs.z.ai/api-reference/llm/chat-completion
[zp-chat]: https://docs.bigmodel.cn/api-reference/%E6%A8%A1%E5%9E%8B-api/%E5%AF%B9%E8%AF%9D%E8%A1%A5%E5%85%A8
[zp-claude-compat]: https://docs.bigmodel.cn/cn/guide/develop/claude/introduction
[zp-coding-model]: https://docs.bigmodel.cn/cn/coding-plan/latest-model
[zp-gh-15]: https://github.com/zai-org/zai-coding-plugins/issues/15
[zp-gh-74]: https://github.com/zai-org/GLM-5/issues/74
[zp-resp]: https://docs.bigmodel.cn/api-reference/response/%E5%88%9B%E5%BB%BA-response
[zp-rl]: https://docs.bigmodel.cn/cn/api/rate-limit
[zp-think]: https://docs.bigmodel.cn/cn/guide/capabilities/thinking
