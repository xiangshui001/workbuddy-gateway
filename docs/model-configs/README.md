# 国内站模型配置与实测记录

检查日期：2026-10-05。测试通过本机 OpenAI 兼容网关进行，不代表之后模型目录与权限不变。

| 模型 ID | 建议 reasoning_effort | 目录输入上限 | 目录输出上限 |
|---|---|---:|---:|
| glm-5.3 | high | 1000000 | 64000 |
| glm-5.3-flash | high | 1000000 | 131072 |
| kimi-k3-1 | medium | 1000000 | 32000 |
| hy4-preview | high | 960000 | 64000 |
| hy3 | high | 192000 | 64000 |
| deepseek-v4.1-flash | high | 1000000 | 128000 |

## 使用方式

六个以模型 ID 命名的 JSON 文件各自是一份完整的 `/v1/chat/completions` 请求体。替换 `messages` 中的问题后单独发送，不要把多个文件组成的数组直接提交。

- SDK Base URL：`http://127.0.0.1:8317/v1`
- 请求地址：`http://127.0.0.1:8317/v1/chat/completions`
- 请求头：`Content-Type: application/json`、`Authorization: Bearer <你的网关密钥>`
- 模板使用 `max_tokens: 8192`，长输出可调高，但应遵守目录输出上限。推理也会消耗生成预算。
- Responses 接口对应写法是 `reasoning: {"effort": "high"}`，并使用 `max_output_tokens`；Kimi 使用 `medium`。

上下文容量在客户端的模型配置中设置，不是 Chat Completions 请求字段。不要在请求体中加入 `context_window` 或 `contextWindow`。本次国内站 v2 目录为除 HY3 外的五款模型提供了默认 300000 的窗口设置；HY3 的目录输入上限为 192000。需要长上下文时再调整。

## 实测结论与边界

共 84 次真实请求：83 次 HTTP 200，1 次故意使用无效思考值的 DeepSeek 对照测试返回 HTTP 400 / code 11150。推荐配置的六款模型单轮、多轮、Responses 调用均得到预期算术答案，并有思考正文或思考 token 计数。GLM-5.3 的一次短多轮请求只返回思考 token 计数，没有思考正文。

- GLM 两款的 v2 目录列出 `low/high/max`，HY4 列出的档位只有 `high`。
- Kimi K3、HY3 没有明确的可选档位列表，分别采用声明的 `medium`、`high`。
- GLM-5.3 的 v2 默认为 `high`，v3 声明为 `medium`，这里提供已实测通过的 `high`，不宣称它是所有官方客户端的默认。
- DeepSeek 默认、`none/off` 测试没有思考正文或思考 token；显式 `high` 时有。其 `low/medium/high/max/xhigh` 请求也通过，但本测试不是不同档位的效果基准。
- 其他五款模型连故意填写的无效档位也返回 200，所以 HTTP 成功不能证明该档位有意义。它们在 `none/off` 下仍有思考，不能据此保证关闭思考。
- `kimi-k3-1` 与 `kimi-k3-2` 都调用成功；模板选用国内站实时目录中的 `kimi-k3-1`。

输入和输出容量来源于检查当天的官方配置。已使用目录输出上限作为参数提交并得到短答案，未实际生成到输出边界，也未发送百万 token 验证输入边界。短测试成功不能保证所有长会话或工具场景。

## 来源与文件

- 官方实时目录：`https://copilot.tencent.com/v2/enterprises/personal/models`
- 官方配置：`https://copilot.tencent.com/v3/config`
- [官方模型配置说明](https://www.codebuddy.cn/docs/workbuddy/From-Beginner-to-Expert-Guide/Function-Description/Model)
- `model-capabilities.json`：所选模型容量、思考元数据及 v2/v3 差异。
- `test-results.json`：虚构算术问题的实测摘要；保留思考长度与 token 数，不包含思考正文、认证密钥、账号标识或请求标识。

本目录只提供配置与历史测试资料，不改变网关的运行行为。
