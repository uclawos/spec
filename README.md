# UCP Spec

> U-ClawOS Protocol（UCP）规范。
> Human-readable: [`PROTOCOL.md`](./PROTOCOL.md) (English) / [`docs/14-UCP协议规范.md`](../docs/14-UCP协议规范.md) (Chinese, authoritative)。

## 内容

- [`PROTOCOL.md`](./PROTOCOL.md) — 协议规范英文版（v0.2 起）
- [`schemas/agent.schema.json`](./schemas/agent.schema.json) — `agent.toml` 的 JSON Schema (v0.2)
- `schemas/job.schema.json` — `job.toml` 的 JSON Schema（v0.2.x 补）

## 当前协议版本

`v0.2` (Draft, 2026-04-30) —— 主要变更：`runtime.type = "standalone"`（"软件管家"模式）+ minor 向后兼容。详见 [`PROTOCOL.md`](./PROTOCOL.md) §4.3。

## 用途

1. **校验 `agent.toml` 的合法性**（CI / SDK / 在线 validator）
2. **生成各语言 SDK 的 model 类**（Rust / TS / Python）
3. **作为 IDE 自动补全数据源**（VS Code 已能识别 JSON Schema for TOML 通过 even-better-toml 插件）

## 与 SDK 的关系

- `sdk-rust` 编译时 `include_str!("../schemas/agent.schema.json")` 嵌入。本 spec 仓是 source-of-truth；sdk-rust 仓内有副本（与 sdk 一起发版），CI 同步保证一致
- `sdk-ts` 通过 npm 引用本仓库（待发布到 `@uclawos/spec`）
- 第三方语言：直接 fetch 文件 URL（`https://uclawos.org/spec/v0.1/schemas/agent.schema.json` — 域名待启用）

## 版本管理

- Schema `$id` URL 含 `v0.1/` —— 新版本不破坏老 URL
- Breaking change 走 v0.x → v1.0 升级
- 每次发布要更 `docs/14-UCP协议规范.md` 的 §12（v0.1 的边界）

## 路线图

| 版本 | 时间 | 主要变化 |
|---|---|---|
| v0.1 | 2026-04 | 初版：agent.toml + job.toml 主结构 |
| **v0.2** | **2026-04** | **`runtime.type = "standalone"`；minor 向后兼容；英文 PROTOCOL.md** |
| v0.3 | 2026-Q4 | 容器隔离、Skill 协议草案、联邦多机 |
| v1.0 | 2027 | 至少 3 家独立厂商 Orchestrator 实现 |
