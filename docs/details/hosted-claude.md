# Claude 的云托管版本（Bedrock / Vertex / Azure Foundry）

> 链接文字写明来源的厂商和文档，完整列表见 [来源](sources.md)；“冲突”“未核实”汇总在 [冲突与未核实](conflicts.md)。

同样是 Anthropic Messages，放到云厂商上，地址、`model` 位置、版本号、流式格式都会变。

| | Anthropic 官方 | Bedrock `InvokeModel` | Vertex AI | Azure Foundry |
|---|---|---|---|---|
| 地址 | `POST https://api.anthropic.com/v1/messages` | `/model/{modelId}/invoke`（流式 `invoke-with-response-stream`） | `…/publishers/anthropic/models/{MODEL}:rawPredict`（流式 `:streamRawPredict`） | `https://<resource>.services.ai.azure.com/anthropic/v1/messages` |
| `model` 放哪 | body | 路径里，body 不写 | 路径里，body 不能写 | body，填**部署名** |
| 鉴权 | `Authorization: Bearer` 或 `x-api-key` | AWS SigV4（Messages 路由也可用 Bedrock API key） | Google OAuth Bearer | Entra Bearer 或 `x-api-key` |
| 版本 | 头 `anthropic-version: 2023-06-01` | body `"anthropic_version": "bedrock-2023-05-31"` | body `"anthropic_version": "vertex-2023-10-16"` | 头 `anthropic-version` |
| beta 功能 | 头 `anthropic-beta` | body `anthropic_beta`（字符串数组） | 未核实 | 头 `anthropic-beta` |
| 流式 | SSE | AWS event-stream 编码（`application/vnd.amazon.eventstream`） | 走 `:streamRawPredict` | SSE |

来源：[Anthropic Messages 参考][an-msg]；[AWS Bedrock InvokeModel 参考][bedrock-invoke]；[AWS Bedrock Messages API 文档][bedrock-msg]；[AWS Bedrock Claude 请求参数文档][bedrock-params]；[Anthropic 旧版 Bedrock 说明][an-bedrock]；[Vertex AI Claude 使用指南][vx-claude]；[Anthropic Vertex AI 说明][an-vertex]；[Azure Foundry Claude 模型说明][az-claude]；[Azure Foundry Claude 使用指南][az-claude-use]。Bedrock 另有 `/anthropic/v1/messages` 路由：`model` 写在 body，用官方 `anthropic-version` 头和 SSE，不是上表的 eventstream（[AWS Bedrock Messages API 文档][bedrock-msg]）。Bedrock 的 `Converse` 是 AWS 自己的格式：工具块写成 `{toolUse: ...}`，不是 `{type:"tool_use"}`，不算五种格式之一（[AWS Bedrock Converse 文档][bedrock-converse]）。

功能支持差异（✓ 支持，✗ 不支持，— 没写）：

| 功能 | Anthropic 官方 | Bedrock `InvokeModel` | Bedrock Messages 路由（mantle） | Vertex AI | Azure Foundry |
|---|---|---|---|---|---|
| Message Batches | ✓ | ✗ | ✗ | Anthropic 页写 ✗；Google 另有 Batch predictions（冲突，[冲突与未核实](conflicts.md)） | — |
| Files API | ✓ | ✗ | ✗ | ✗ | 只有 Anthropic 托管的部署有 |
| 网页搜索 | ✓ | ✗ | ✗ | ✓（基础版） | ✓（Azure 托管只支持 `web_search_20250305`） |
| 网页抓取 | ✓ | ✗ | ✗ | ✗ | 同页两句说法不一（[冲突与未核实](conflicts.md)） |
| 代码执行 | ✓ | ✗ | ✗ | ✗ | — |
| `count_tokens` | ✓ | AWS 自有 `CountTokens`，部分 Claude 型号没有 | ✓ | ✓（`count-tokens:rawPredict`） | ✓ |
| 结构化输出 | ✓ | Anthropic 旧页写 ✓，AWS 页只写 4 个型号 ✓（冲突，[冲突与未核实](conflicts.md)） | ✗（传 `output_config.format` 会 400） | ✓ | ✓ |
| 单请求上限 | 32 MB | 20 MB 与 API 参考的 25000000 不一致（[冲突与未核实](conflicts.md)） | — | 30 MB | — |

来源：[Anthropic 批处理文档][an-batch]；[Anthropic Files API 参考][an-files]；[Anthropic 网页搜索工具文档][an-websearch]；[Anthropic 网页抓取工具文档][an-webfetch]；[Anthropic 代码执行工具文档][an-codeexec]；[Anthropic token 计数文档][an-count]；[Anthropic 结构化输出文档][an-so]；[Anthropic 旧版 Bedrock 说明][an-bedrock]；[Anthropic Bedrock 说明（Messages 路由）][an-bedrock-mantle]；[AWS Bedrock InvokeModel 参考][bedrock-invoke]；[AWS Bedrock CountTokens 文档][bedrock-count]；[AWS Bedrock Claude 结构化输出文档][bedrock-so]；[Anthropic Vertex AI 说明][an-vertex]；[Vertex AI Claude 批量预测文档][vx-claude-batch]；[Vertex AI Claude token 计数文档][vx-claude-count]；[Azure Foundry Claude 模型说明][az-claude]。

---

[an-batch]: https://platform.claude.com/docs/en/build-with-claude/batch-processing
[an-bedrock]: https://platform.claude.com/docs/en/build-with-claude/claude-on-amazon-bedrock-legacy
[an-bedrock-mantle]: https://platform.claude.com/docs/en/build-with-claude/claude-in-amazon-bedrock
[an-codeexec]: https://platform.claude.com/docs/en/agents-and-tools/tool-use/code-execution-tool
[an-count]: https://platform.claude.com/docs/en/build-with-claude/token-counting
[an-files]: https://platform.claude.com/docs/en/api/files-create
[an-msg]: https://platform.claude.com/docs/en/api/messages
[an-so]: https://platform.claude.com/docs/en/build-with-claude/structured-outputs
[an-vertex]: https://platform.claude.com/docs/en/build-with-claude/claude-on-vertex-ai
[an-webfetch]: https://platform.claude.com/docs/en/agents-and-tools/tool-use/web-fetch-tool
[an-websearch]: https://platform.claude.com/docs/en/agents-and-tools/tool-use/web-search-tool
[az-claude]: https://learn.microsoft.com/en-us/azure/foundry/foundry-models/concepts/claude-models
[az-claude-use]: https://learn.microsoft.com/en-us/azure/foundry/foundry-models/how-to/use-foundry-models-claude
[bedrock-converse]: https://docs.aws.amazon.com/bedrock/latest/userguide/conversation-inference.html
[bedrock-count]: https://docs.aws.amazon.com/bedrock/latest/userguide/count-tokens.html
[bedrock-invoke]: https://docs.aws.amazon.com/bedrock/latest/APIReference/API_runtime_InvokeModel.html
[bedrock-msg]: https://docs.aws.amazon.com/bedrock/latest/userguide/inference-messages-api.html
[bedrock-params]: https://docs.aws.amazon.com/bedrock/latest/userguide/model-parameters-anthropic-claude-messages-request-response.html
[bedrock-so]: https://docs.aws.amazon.com/bedrock/latest/userguide/claude-messages-structured-outputs.html
[vx-claude]: https://docs.cloud.google.com/gemini-enterprise-agent-platform/models/partner-models/claude/use-claude
[vx-claude-batch]: https://docs.cloud.google.com/gemini-enterprise-agent-platform/models/partner-models/claude/batch
[vx-claude-count]: https://docs.cloud.google.com/gemini-enterprise-agent-platform/models/partner-models/claude/count-tokens
