## 1. 项目脚手架

- [ ] 1.1 创建 Go CLI 入口以及配置、解析、报告和 GitLab 集成所需的包结构
- [ ] 1.2 增加 YAML 配置加载、基于环境变量的 token 读取，以及清晰的配置校验错误

## 2. MTR 解析与 case 处理

- [ ] 2.1 实现对 MTR 输入产物的递归扫描，以及对 `failed` 和 `result_mismatch` 失败的解析支持
- [ ] 2.2 实现 CASE marker 解析和安全的 case content 提取，并支持降级处理
- [ ] 2.3 实现 case reference 生成、fingerprint 计算，以及按 `case_ref` 的去重逻辑

## 3. 报告生成与验证

- [ ] 3.1 生成包含失败详情、去重分组和运行元数据的 `report.json`
- [ ] 3.2 生成空或非空的 `report.junit.xml`，并保证正确转义和合法 JUnit 输出
- [ ] 3.3 为 parser 样例和 CASE marker 边界情况补充 golden test

## 4. GitLab Issue 注册

- [ ] 4.1 实现 GitLab Issue 的查询、匹配、创建和评论追加，使用稳定标题和隐藏标记
- [ ] 4.2 实现重试逻辑、超时处理以及对临时 API 错误的清晰日志输出
- [ ] 4.3 实现 dry-run、Issue 数量保护以及 Markdown 安全嵌入

## 5. 端到端验证

- [ ] 5.1 使用样例产物执行 CLI，验证报告生成和 Issue 动作是否符合预期
- [ ] 5.2 使用 OpenSpec 工作流验证变更，并确认规划产物完整
