# MTR Issue Bot

## Purpose

该能力用于让系统从 MTR 的原始执行产物中分析出结构化失败记录，并将每个唯一失败 case 可靠地登记到 GitLab Issue 中，形成可验证、可维护的处理流程。

## Definitions

- `case_ref`: 失败记录的稳定引用，优先使用可识别的 case_id，回退到 `<suite>/<test_file>` 或 `test_file`。
- `fingerprint`: 基于 `case_ref` 的稳定哈希值，计算方式为 `sha256("mtr|" + case_ref)`。
- `dry-run`: 在生成报告的情况下跳过 GitLab API 调用，仅验证数据处理路径。

## Behavior

### Requirement: 从单一 CLI 入口分析 MTR 产物
系统必须接受 `--input`、`--cases`、`--output` 和 `--config` 这几个参数，并从这些输入中处理 MTR 原始结果和 mysql-test 用例文件。

#### Scenario: 成功执行分析
- **WHEN** 输入目录中包含 MTR 产物，测试用例目录中包含 `.test` 文件
- **THEN** 系统应生成报告输出文件，并在部分文件无法解析时仍继续处理其它内容

#### Scenario: 输入目录缺失
- **WHEN** 提供的输入目录不存在
- **THEN** 系统必须返回清晰错误，并且不得继续执行不完整的分析流程

### Requirement: 解析 MTR 中支持的失败类型
系统必须识别 MTR 输出中的 `failed` 和 `result_mismatch` 两类失败，并尽可能提取如 suite、test file、错误码、错误信息、失败 SQL、行号、日志片段和 diff 等字段；对于无法提取的信息，不能伪造。

#### Scenario: 包含错误信息的 failed 失败
- **WHEN** MTR 日志中出现带 SQL 错误和行号信息的 `failed` 失败
- **THEN** 系统应生成包含这些提取信息的失败记录，并将不可用字段保留为空

#### Scenario: 存在 reject/result 的 result mismatch
- **WHEN** MTR 输出显示结果不匹配，并且 `.reject` / `.result` 产物可用
- **THEN** 系统应生成包含统一 diff 的失败记录，并标记为 `result_mismatch`

## Case Extraction

### Requirement: 基于 CASE marker 安全提取 case content
系统只有在失败记录能够关联到测试文件，并且测试文件满足 CASE marker 规范化前提时，才应提取 `case_content`；否则应保留 `case_content` 为空，并降级为基于 test file 的处理。

#### Scenario: CASE marker 合法且能匹配失败位置
- **WHEN** 测试文件中存在唯一且稳定的 `# CASE: <case_id>` 标记，并且失败记录能映射到对应 case 区间
- **THEN** 系统应提取对应的 case 内容并写入失败记录

#### Scenario: CASE marker 前提不满足
- **WHEN** 测试文件缺少有效 CASE 标记，或者无法可靠定位对应位置
- **THEN** 系统应保留 `case_content` 为空，记录 warning，并且不得伪造 case 内容

### Requirement: 生成稳定的 case reference 和 fingerprint
系统必须在可识别 case_id 时使用其作为 `case_ref`，否则回退为 `<suite>/<test_file>` 或 `test_file`；同时必须使用 `sha256("mtr|" + case_ref)` 计算 `fingerprint`。

#### Scenario: 能识别 case 标识
- **WHEN** 失败记录中包含可识别的 case 标识
- **THEN** 系统应使用该标识作为 `case_ref`，并据此生成 fingerprint

#### Scenario: 无法识别 case 标识
- **WHEN** 失败记录无法关联到 case 标识
- **THEN** 系统应使用基于 suite/test-file 的回退值作为 `case_ref`

## Reporting

### Requirement: 对失败记录去重并生成报告
系统必须按 `case_ref` 对失败记录进行去重；对于每个 case_ref，使用第一条解析出的记录作为主记录，并保留出现次数与失败类型信息；系统还必须为每次运行生成 `report.json` 和 `report.junit.xml`，即使没有任何失败也要生成空报告。

#### Scenario: 同一 case_ref 出现多条失败
- **WHEN** 同一个 `case_ref` 在一次运行中出现多条失败记录
- **THEN** 系统应生成一个 issue 分组，并使用主失败记录以及汇总后的出现次数信息

#### Scenario: 没有发现任何失败
- **WHEN** 分析完成后没有任何失败记录
- **THEN** 系统仍应生成合法的空报告文件，并且不得创建或更新 GitLab Issue

## GitLab Integration

### Requirement: 安全地登记 GitLab Issue
系统必须查询当前项目中已打开、带有配置标签的 Issue；优先按 fingerprint 匹配，若未命中则回退到标题匹配；若没有找到匹配的已打开 Issue，则创建新 Issue；若发现已存在匹配 Issue，则追加评论。

#### Scenario: 发现新的失败 case
- **WHEN** 去重后的 case reference 没有匹配到已打开 Issue
- **THEN** 系统应创建标题为 `[MTR] <case_ref>` 的 GitLab Issue，并附带隐藏的 fingerprint 元数据

#### Scenario: 同一个 case 再次失败
- **WHEN** 已存在匹配 fingerprint 或标题的已打开 Issue
- **THEN** 系统应追加评论，并且不得重复创建 Issue

## Safety and Controls

### Requirement: 支持安全控制与稳定性行为
系统必须支持 `enabled: false` 的 dry-run 模式，遵守 `max_issues_per_run` 限制，从环境变量读取 GitLab token，并对 GitLab API 的临时失败进行超时、清晰日志和重试处理。

#### Scenario: 启用 dry-run 模式
- **WHEN** 配置中 `bot.enabled` 设置为 `false`
- **THEN** 系统应生成报告，跳过所有 GitLab API 调用，并输出估算的唯一 case 数量

#### Scenario: 超出 Issue 数量限制
- **WHEN** `max_issues_per_run` 大于 0，并且去重后的 case 数量超过该限制
- **THEN** 系统仅生成报告，不创建或更新任何 GitLab Issue
