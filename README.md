# dsh-config-gen

Writes and checks the provider config file for DeepSeek Harness, so you do not have to
hand-edit YAML.

dsh keeps its provider wiring in `~/.dsh/profiles/<profile>/cordis.patch.yml`. That file is
a YAML array of loader patches, not a normal config file. It is expressive and it fails
quietly:

- One wrong indent under `providers:` and the provider does not exist. No error, no
  warning, the model just is not selectable.
- It is also the obvious place to paste an API key, which is how keys end up in issues
  and screenshots.

This tool covers both: generate a correct file from flags or a wizard, and validate an
existing one before you start dsh.

Standard library only. Python 3.8+.

## Generate

```bash
# minimal: provider wiring only
python dsh_config_gen.py --preset minimal \
  --base-url http://127.0.0.1:8080/v1 \
  --api-key-env BRIDGE_API_KEY \
  --model chat-model \
  --print
```

```bash
# full: also turns off the hosted session log
python dsh_config_gen.py --preset full \
  --base-url http://127.0.0.1:8080/v1 \
  --api-key-env BRIDGE_API_KEY \
  --model chat-model:1000000:128000 \
  --model long-context-model:256000:32000
```

`--model` accepts three forms:

| Form | Meaning |
|---|---|
| `id` | 128000 context, 8192 output |
| `id:CONTEXT` | custom context window |
| `id:CONTEXT:MAXOUT` | both |

Write to a file instead of stdout:

```bash
python dsh_config_gen.py --preset full --model chat-model
# wrote ~/.dsh/profiles/desktop/cordis.patch.yml
# next: make sure BRIDGE_API_KEY is set, then fully restart dsh
```

Use `--output PATH` to write elsewhere and `--force` to overwrite. Without `--force` it
refuses to touch an existing file, since losing a working config to a typo is a bad trade.

### Wizard

```bash
python dsh_config_gen.py --wizard
```

Asks for the base URL, the environment variable name, and the model list, including
context window and vision support per model.

### Output

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

The key is not in that file. Only the name of the environment variable that holds it.

## Validate

```bash
python dsh_config_gen.py --validate ~/.dsh/profiles/desktop/cordis.patch.yml
```

```
dsh-config-gen 1.0.0
validating ~/.dsh/profiles/desktop/cordis.patch.yml
==============================================================
[FAIL] secret     plaintext apiKey value line 173
       Use apiKeyEnv instead, then rotate this key if the file
       was ever shared.
[FAIL] provider   provider 'local-bridge' has no baseURL
[WARN] provider   provider 'local-bridge' baseURL does not end with
       /v1: http://127.0.0.1:8080
       The OpenAI-compatible routes live under /v1. Without it
       you get 404s on every call.
[WARN] provider   env var 'BRIDGE_API_KEY' is not set in this shell
--------------------------------------------------------------
2 failure(s), 2 warning(s)
```

What it checks:

| Area | Checks |
|---|---|
| `structure` | Top-level array shape, entries without ids, duplicate ids |
| `secret` | Literal `apiKey:` values |
| `provider` | Missing `baseURL`, non-http scheme, missing `/v1` suffix, `localhost` caveat, missing `apiKeyEnv`, unset env var, empty model list |
| `model` | `agent-default-model` pointing at a model no provider declares |

`--json` for machine consumption. Exit code `1` when there is at least one failure, so it
works as a pre-commit hook or a CI gate:

```bash
python dsh_config_gen.py --validate cordis.patch.yml || exit 1
```

The generator runs the same checks on its own output before writing, so it will not produce
a file that fails them.

## Field reference

[`docs/fields.md`](docs/fields.md) lists every key in the provider block, what it does, and
what breaks when it is wrong.

Two settings account for most "it just does not work" reports:

- `baseURL` must end in `/v1`. The OpenAI-compatible routes are mounted there, and without
  the suffix every request 404s.
- `supportsDeveloperRole: false` is correct for most non-OpenAI upstreams. Some gateways
  reject a `developer` role message and report it as an opaque channel error.

## Related

- [`dsh-doctor`](https://github.com/yangjinbin2006/dsh-doctor) checks a running setup: TCP,
  `/v1/models`, one real chat request, plus the usual upstream error codes.

## License

MIT
