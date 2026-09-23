# 工具调用示例

> 五种原生格式在“模型调了一次工具、你把结果带回去”这一轮的最小请求体。字段含义的对照表在 [字段对照 · 工具](../field-atlas.md#2-工具-重点)。
> 链接文字写明来源的厂商和文档，完整列表见 [来源](sources.md)；“冲突”“未核实”汇总在 [冲突与未核实](conflicts.md)。

第二轮请求体最小示例如下（都是合法 JSON）（[OpenAI 函数调用指南][oa-fc]；[OpenAI 迁移到 Responses 指南][oa-migrate]；[Anthropic 定义工具文档][an-tools]；[Anthropic 处理工具调用指南][an-handle]；[Gemini generateContent 函数调用指南][g-fc]；[Gemini Interactions 参考][g-int-api]；[Gemini Interactions 快速入门][g-int-qs]）。

Chat Completions：

```json
{"messages":[{"role":"user","content":"Weather in Paris?"},
 {"role":"assistant","tool_calls":[{"id":"call_123","type":"function","function":{"name":"get_weather","arguments":"{\"location\":\"Paris\"}"}}]},
 {"role":"tool","tool_call_id":"call_123","content":"15C"}]}
```

Responses：

```json
{"input":[{"role":"user","content":"Weather in Paris?"},
 {"type":"function_call","call_id":"call_123","name":"get_weather","arguments":"{\"location\":\"Paris\"}"},
 {"type":"function_call_output","call_id":"call_123","output":"15C"}]}
```

Anthropic Messages：

```json
{"messages":[{"role":"user","content":"Weather in Paris?"},
 {"role":"assistant","content":[{"type":"tool_use","id":"toolu_1","name":"get_weather","input":{"location":"Paris"}}]},
 {"role":"user","content":[{"type":"tool_result","tool_use_id":"toolu_1","content":"15C"}]}]}
```

Gemini `generateContent`（`thoughtSignature` 填模型上一轮返回的值）：

```json
{"contents":[{"role":"user","parts":[{"text":"Weather in Paris?"}]},
 {"role":"model","parts":[{"thoughtSignature":"SIG_FROM_MODEL","functionCall":{"id":"fc1","name":"get_weather","args":{"location":"Paris"}}}]},
 {"role":"user","parts":[{"functionResponse":{"id":"fc1","name":"get_weather","response":{"result":"15C"}}}]}]}
```

Gemini Interactions（有状态：只发工具结果，历史在服务端；`tools` 每轮要重发，这里省略）：

```json
{"model":"gemini-3.5-flash","previous_interaction_id":"PREVIOUS_ID",
 "input":[{"type":"function_result","call_id":"fc1","name":"get_weather","result":[{"type":"text","text":"15C"}]}]}
```

---

[an-handle]: https://platform.claude.com/docs/en/agents-and-tools/tool-use/handle-tool-calls
[an-tools]: https://platform.claude.com/docs/en/agents-and-tools/tool-use/define-tools
[g-fc]: https://ai.google.dev/gemini-api/docs/generate-content/function-calling
[g-int-api]: https://ai.google.dev/api/interactions-api
[g-int-qs]: https://ai.google.dev/gemini-api/docs/interactions/quickstart
[oa-fc]: https://developers.openai.com/api/docs/guides/function-calling
[oa-migrate]: https://developers.openai.com/api/docs/guides/migrate-to-responses
