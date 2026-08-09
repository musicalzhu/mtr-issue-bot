## Context

仓库正在为一个基于 Go 的 MTR 分析工具做准备。该变更需要满足提供的 MVP 规格要求，同时保持实现尽量保守，并兼容 GitLab CI 的实际使用场景。

## Goals / Non-Goals

**Goals:**
- 将 MTR 原始产物解析成结构化失败记录。
- 在测试文件满足 CASE marker 规范时提取 case 内容。
- 生成稳定的报告，并在不泄露敏感信息的前提下注册 GitLab Issue。

**Non-Goals:**
- 自动根因分析、MR/Pipeline 自动评论、flaky 检测或更广泛的缺陷系统集成。
- 支持所有可能的 MTR 输出变体；MVP 只针对当前团队使用的真实样例格式。

## Decisions

- 实现将使用单一的 `analyze` CLI 入口，并采用以解析、case 提取、报告生成、去重和 GitLab 集成为中心的 Go 模块化结构。这能保持命令入口简洁，并与 MVP 需求保持一致。
- MVP 将使用 Go 实现，而不是临时脚本。该决策基于与团队现有工具链生态一致、适合 GitLab Runner 的单二进制部署，以及未来可扩展到 `jmeter` 或 `go-randgen` 等测试工具的可维护性。
- MTR 解析将采用“尽力而为、保守落地”的策略；无法完全识别的字段将保留为空，并以 warning 形式记录，而不是伪造数据。
- Issue 身份将基于仅由 `case_ref` 派生的稳定 fingerprint，并使用 `[MTR] <case_ref>` 作为稳定标题。这能避免因错误信息变化而导致标题频繁变更。
- GitLab 集成将使用带超时、简单退避和重试逻辑的客户端，明确处理网络错误、超时、HTTP 429 和 HTTP 5xx；不可重试的 4xx 错误会快速失败并记录清晰日志。
- Markdown 嵌入将优先使用可自动选择长度的 fenced code block；必要时回退到缩进代码块，以避免破坏 Issue 内容结构。

## Risks / Trade-offs

- [解析器可能漏掉极端日志格式] → 缓解方案：依赖真实样例的 golden test，并在无法解析时通过 warning 降级处理。
- [CASE marker 规范化对老旧测试文件可能不完全适用] → 缓解方案：不伪造 case content；当前提不满足时，降级为基于 test file 的处理。
- [GitLab API 可用性可能存在波动] → 缓解方案：显式实现重试、保留 dry-run 支持，并在失败时返回非零退出码，便于 CI 可见。

## Migration Plan

1. 先实现 parser 与 case indexing 模块，并配套 golden test。
2. 再接入报告生成与 Issue 注册流程，并由配置开关控制是否真正调用 GitLab API。
3. 使用样例产物在 CI 中进行验证，确认 `report.json` 和 `report.junit.xml` 能在空运行或 dry-run 场景下正确生成。

## Open Questions

- 仓库是否需要扩展 fixtures 目录位置，除了 MVP 建议的 `testdata/mtr/` 之外；这个问题可以在实现阶段再决定，不会影响当前规格契约。
