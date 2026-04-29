# UCP JSON Schemas

> 机器可读的 UCP 协议规范。配套人类可读的 [`docs/14-UCP协议规范.md`](../../docs/14-UCP协议规范.md)。

## 文件清单

| 文件 | 内容 | 协议章节 |
|---|---|---|
| `agent.schema.json` | `agent.toml` 校验 schema | §4 |
| `job.schema.json` | `job.toml` 校验 schema（v0.1.1 补） | §5 |

## 怎么用

### Rust（u-claw-os-sdk）

```rust
use u_claw_os_sdk::{parse_agent_manifest, validate::validate_agent};

let m = parse_agent_manifest(&toml_text)?;
validate_agent(&m)?;
```

### 命令行（其他语言）

```bash
# 用任意 jsonschema 工具
ajv validate -s agent.schema.json -d openclaw.agent.json
```

把 `agent.toml` 转 JSON：

```bash
yj -t < openclaw/agent.toml > openclaw.agent.json
```

### Web

`https://uclawos.org/validator` 提供在线校验。

## 版本策略

- Schema `$id` URL 含 `v0.1/`，新版本会换 URL（不破坏老 schema）
- `agent.toml` 里的 `meta.ucp_version` 字段必须和 schema 版本对得上

## 贡献新 Agent

1. 写 `agents/<id>/agent.toml`
2. 用上面的 SDK / ajv 跑通校验
3. 提 PR 到 `github.com/u-claw-os/agents`
