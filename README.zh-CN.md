# dsh-config-gen

生成和校验 DeepSeek Harness 的 provider 配置，省掉手写 YAML。

dsh 把 provider 配置放在 `~/.dsh/profiles/<profile>/cordis.patch.yml`。这不是普通配置文件，
而是顶层为数组的 loader 补丁层，表达力强，但出错时没有任何反馈：

- `providers:` 下面缩进错一格，provider 就不存在了。没有报错，没有警告，只是模型列表里
  少了一项。
- 它同时也是大家贴 API Key 的地方，所以 key 很容易跟着截图和 issue 一起公开。

这个工具两头都管：用参数或用向导生成一份正确的文件，以及在启动 dsh 之前校验已有的文件。

只用 Python 标准库，3.8 以上。

## 生成

```bash
# 最小版：只生成 provider 接线
python dsh_config_gen.py --preset minimal \
  --base-url http://127.0.0.1:8080/v1 \
  --api-key-env BRIDGE_API_KEY \
  --model chat-model \
  --print
```

```bash
# 完整版：额外关闭托管会话日志
python dsh_config_gen.py --preset full \
  --base-url http://127.0.0.1:8080/v1 \
  --api-key-env BRIDGE_API_KEY \
  --model chat-model:1000000:128000 \
  --model long-context-model:256000:32000
```

`--model` 支持三种写法：

| 写法 | 含义 |
|---|---|
| `id` | 128000 上下文，8192 输出 |
| `id:CONTEXT` | 自定义上下文窗口 |
| `id:CONTEXT:MAXOUT` | 两个都自定义 |

写到文件而不是 stdout：

```bash
python dsh_config_gen.py --preset full --model chat-model
# wrote ~/.dsh/profiles/desktop/cordis.patch.yml
# next: make sure BRIDGE_API_KEY is set, then fully restart dsh
```

`--output PATH` 指定别的路径，`--force` 允许覆盖。不加 `--force` 时不会动已有文件，因为
手滑丢掉一份能用的配置不划算。

### 向导模式

```bash
python dsh_config_gen.py --wizard
```

依次问 base URL、环境变量名和模型列表，每个模型再问上下文窗口和是否支持图片。

### 产物

```yaml
- id: agent-default-model
  name: "@deepseek-ai/dsh-agent-default-model"
  config:
    provider: local-bridge
    model: chat-model
    reasoningEffort: high

- id: llm-pi-ai
  name: "@deepseek-ai/dsh-llm-pi-ai"
  config:
    providers:
      local-bridge:
        displayName: Local Bridge
        apiKeyEnv: BRIDGE_API_KEY
        api: openai-completions
        baseURL: http://127.0.0.1:8080/v1
        compat:
          thinkingFormat: openai
          supportsReasoningEffort: true
          supportsDeveloperRole: false
        models:
          - id: chat-model
            contextWindow: 1000000
            maxTokens: 128000
            supportsVision: true
```

文件里没有 key，只有保存 key 的环境变量名。

## 校验

```bash
python dsh_config_gen.py --validate ~/.dsh/profiles/desktop/cordis.patch.yml
```

检查项：

| 分类 | 内容 |
|---|---|
| `structure` | 顶层是否为数组、有无缺 id 的条目、id 是否重复 |
| `secret` | 是否有明文 `apiKey:` 值 |
| `provider` | 缺 `baseURL`、协议不是 http(s)、结尾缺 `/v1`、`localhost` 提示、缺 `apiKeyEnv`、环境变量未设置、模型列表为空 |
| `model` | `agent-default-model` 指向了没有 provider 声明的模型 |

`--json` 输出机器可读结果。有 failure 时退出码为 `1`，可以当 CI 或 pre-commit 门禁：

```bash
python dsh_config_gen.py --validate cordis.patch.yml || exit 1
```

生成器写文件前会先对自己产出的内容跑一遍检查，所以不会生成通不过检查的配置。

## 字段说明

每个 provider 字段的含义，以及写错之后的具体症状，见 [`docs/fields.md`](docs/fields.md)。

最常见的两个坑：

- `baseURL` 必须以 `/v1` 结尾。OpenAI 兼容路由挂在 `/v1` 下，漏了就是全 404。
- `supportsDeveloperRole: false` 对绝大多数非 OpenAI 官方端点都该打开。有些网关拒收
  `developer` 角色消息，然后把它报成一个看不出原因的渠道错误。

## 相关项目

- [`dsh-doctor`](https://github.com/yangjinbin2006/dsh-doctor)：对配好的环境做运行时诊断，
  包括 TCP、`/v1/models`、一次真实对话请求，以及常见上游错误码。

## 许可

MIT
