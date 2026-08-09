## Why

当前的 MTR/mysqltest 工作流仍然依赖人工查看原始日志、汇总失败 case、分析 failed 或 result mismatch，并手工登记 GitLab Issue。该变更引入一个自动化的 MVP，让 MTR 产物能够直接转成结构化报告和 GitLab Issue，从而显著减少人工整理和登记成本。

## What Changes

- 新增 CLI 命令 `mtr-issue-bot analyze`，用于读取 MTR 原始结果和 mysql-test 用例文件。
- 解析 `failed` 和 `result_mismatch` 两类失败，提取可用上下文信息并生成稳定的 case reference。
- 当测试文件满足 CASE marker 规范化前提时，提取对应 case 内容；否则安全降级。
- 生成 `report.json` 和 `report.junit.xml`，按 `case_ref` 去重失败记录，并支持 dry-run 与 Issue 数量保护模式。
- 创建或更新 GitLab Issue，使用稳定标题、隐藏 fingerprint 标记、Markdown 安全嵌入以及重试感知的 API 处理。
- 确认 MVP 的实现语言决策：推荐使用 Go 实现，因为它与团队现有 DevOps/运维工具生态一致，适合 GitLab Runner 单二进制部署，并且便于后续接入更多测试工具。

## Capabilities

### New Capabilities
- `mtr-issue-bot`: 面向 MTR 原始结果的端到端 MVP 分析能力，覆盖 case 提取、报告生成与 GitLab Issue 登记。

### Modified Capabilities
- 无。

## Impact

- 在代码仓库中新增 Go 语言 CLI 入口。
- 新增基于 YAML 的配置加载与 GitLab / bot 参数处理。
- 新增报告生成、GitLab 集成以及 parser golden test 相关模块。
