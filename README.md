# LLM API 协议：学习用地图手册

写给要直接调 LLM API、以后要写适配层的工程师。先看懂五种原生请求格式——OpenAI Chat Completions、OpenAI Responses、Anthropic Messages、Gemini `generateContent`、Gemini Interactions——再查 DeepSeek、智谱 GLM、Kimi、xAI Grok、Qwen、MiniMax、豆包、Azure 这些第三方的“兼容”到底兼容了什么。

怎么读：第 0 层 [导读](docs/llm-api-protocols.md)（全局地图，一次读完）→ 第 1 层 [字段对照](docs/field-atlas.md)（写代码时查）→ 第 2 层 `docs/details/` 下的细节页（按需查）。

| 层 | 文件 | 内容 |
|---|---|---|
| 0 | [docs/llm-api-protocols.md](docs/llm-api-protocols.md) | 五种格式、第三方怎么分叉、容易记混的名字、停止/流式/报错/断开的要点、怎么选 |
| 1 | [docs/field-atlas.md](docs/field-atlas.md) | 同一个意思在各家叫什么、放哪、默认值；重点字段 |
| 2 | [docs/details/](docs/details/) | 工具调用示例、原生格式细节、第三方厂商、流式与断开、缓存、Claude 云托管、错误与限流、冲突与未核实、来源 |
| — | [docs/taxonomy.md](docs/taxonomy.md) | 为什么这么分类 |
