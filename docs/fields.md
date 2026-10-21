# Patch layer field reference

Everything below lives in `~/.dsh/profiles/<profile>/cordis.patch.yml`.

The file is a **top-level YAML array**. Each element is a loader patch entry that targets an
existing layer by `id` and either overrides its config, disables it, or inserts something new.
If the file is a bare mapping instead of an array, the loader ignores the whole thing without
telling you.

---

## Entry shape

```yaml
- id: <loader id being targeted>
  name: "<package name>"
  config:
    <key>: <value>
```

| Key | Required | Notes |
|---|---|---|
| `id` | yes in practice | The identity later layers target. Entries without an id cannot be overridden or disabled from anywhere else. |
| `name` | usually | The package name the entry binds to. Copy it exactly from the layer you are overriding. |
| `config` | per entry | Merged into the target's config. Shape is defined by the package, not by the loader. |

Duplicate `id`s are not an error — the later entry wins and the earlier one is silently
discarded. That is a common way to lose an afternoon: two `llm-pi-ai` blocks, only the second
one in effect.

---

## `agent-default-model`

```yaml
- id: agent-default-model
  name: "@deepseek-ai/dsh-agent-default-model"
  config:
    provider: local-bridge
    model: chat-model
    reasoningEffort: high
```

| Field | Meaning | When wrong |
|---|---|---|
| `provider` | Must match a key under `providers:` in `llm-pi-ai` | Provider not found; no model selectable |
| `model` | Must match an `id` in that provider's `models` list | First request fails with model-not-found |
| `reasoningEffort` | `low` / `medium` / `high` / model-specific extras | Ignored if the model does not declare it; can 400 if the upstream rejects unknown values |

`reasoningEffort` is only meaningful when the model declares a `reasoningEfforts` mapping.
Declaring an effort the model does not list is a common source of otherwise-inexplicable 400s.

---

## `llm-pi-ai`

```yaml
- id: llm-pi-ai
  name: "@deepseek-ai/dsh-llm-pi-ai"
  config:
    providers:
      <provider-id>:
        ...
```

### Provider block

| Field | Meaning | When wrong |
|---|---|---|
| `displayName` | Label shown in the UI | Cosmetic only |
| `apiKeyEnv` | **Name** of the environment variable holding the key | Key not found; 401 |
| `apiKey` | Literal key | **Do not use.** Ends up in screenshots and dotfile repos |
| `api` | Wire protocol; `openai-completions` for OpenAI-compatible endpoints | Wrong protocol means unparseable responses |
| `baseURL` | Endpoint root, **must include `/v1`** | 404 on every route |
| `compat` | Compatibility switches, see below | Varies — usually opaque 400s |

### `compat`

| Field | Values | Meaning | Symptom when wrong |
|---|---|---|---|
| `thinkingFormat` | `openai` | How reasoning/thinking deltas are expressed | Thinking content missing, or garbled into the answer |
| `supportsReasoningEffort` | `true` / `false` | Whether the endpoint accepts a reasoning-effort hint | 400 on requests carrying the field |
| `supportsDeveloperRole` | `true` / `false` | Whether the endpoint accepts `role: "developer"` | 400 that often surfaces as an opaque channel error (see `11128` in dsh-doctor's docs) |

Default `supportsDeveloperRole` to `false` for anything that is not a first-party OpenAI
endpoint. The failure it causes is indirect and hard to attribute.

### Model block

```yaml
        models:
          - id: chat-model
            contextWindow: 1000000
            maxTokens: 128000
            reasoningEfforts:
              low: low
              high: high
            supportsVision: true
```

| Field | Meaning | When wrong |
|---|---|---|
| `id` | Exact id the endpoint serves — case sensitive | 404, or silent remap to a different model |
| `contextWindow` | Input budget advertised to the client | **Too large is worse than too small**: oversize prompts fail upstream with a context error |
| `maxTokens` | Output budget | Too large gets truncated or rejected |
| `reasoningEfforts` | Allowed effort levels, mapped to upstream values | Declaring an unsupported level yields 400s |
| `supportsVision` | Whether images may be attached | Images rejected, or silently dropped |

`contextWindow` is the field people most often inflate. Advertise what the endpoint actually
enforces, not what the model card claims. A model card number is the theoretical maximum; most
deployments enforce something smaller, and the client has no way to discover the real limit
except by failing.

---

## Verify the id, do not trust the docs

Model ids drift. A deployment may serve `foo-preview-f` when the docs say `foo-preview`, or the
bridge may alias one to the other. Always copy ids from the live endpoint:

```bash
curl -s http://127.0.0.1:8080/v1/models -H "Authorization: Bearer $BRIDGE_API_KEY" | jq -r '.data[].id'
```

If a listed id differs from what your patch layer declares, align them. Depending on the alias
table, the mismatch is either harmless or a hard 404 — and the difference between those two
outcomes is not something you want to discover in production.

---

## Where the key belongs

```yaml
        apiKeyEnv: BRIDGE_API_KEY     # right: config names the variable
        apiKey: sk-...                # wrong: config contains the secret
```

Set the variable in the environment that launches dsh. Caveats that catch people:

- **Windows**: apps launched from Explorer inherit Explorer's environment block from login
  time. `setx` writes the registry but running processes keep the old block. Restart Explorer
  or sign out.
- **macOS**: apps launched from the Dock never read your shell profile. Use `launchctl setenv`.
- **Docker**: the variable has to be on the process, not merely in the image or an unwired
  `.env` file.

If a key ever did land in this file: rotate it first, then remove the line. Cleaning up before
rotating assumes the file was never copied anywhere, which is not an assumption worth making.
