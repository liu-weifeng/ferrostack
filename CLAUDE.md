<!-- cairn:begin -->
## Cairn 工程协同与上下文规范（项目：ferrostack）

本工作区使用 **Cairn** 连接本地开发工具（Claude Code / Codex / Antigravity `agy`）与团队 Web 控制台。请遵循以下轻量工作流（无需配置 MCP）：

1. **开工前同步上下文与最新人工指示**
   - 执行 `cairn brief`（或 `cairn brief "<模块或关键词>"`）读取当前能力承诺边界、活跃设计修订、进行中工作项与待处理问题。
   - 若提示 Web 端有已修改文档（`web_edited`），请先执行 `cairn docs pull` 安全同步到本地（若本地存在未提交冲突改动，会自动保存为 `.cairn-web.md` 旁路文件，绝不覆盖你的本地修改）。
2. **读文档与写文档**
   - 检索文档章节：`cairn docs search "<关键词>"`；读取精确文档或指定章节：`cairn docs read <路径或文档ID> [--section "<章节标题>"]`。
   - 新建或修改本地 Markdown 文档后，执行 `cairn docs push <文件.md>` 立即推送到平台供人类在 Web 端审阅；完成 Git 提交后执行 `cairn docs refresh` 即可升级为 `repo_verified`。
3. **提报、查询和修复 Bug**
- 提 Bug：用 `cairn issue add --kind bug -t "标题" -m "复现步骤、实际与预期结果" --json` 保存发现，记下返回的 ID。多行正文可用 `--stdin`。
- 找 Bug：用 `cairn issue list --kind bug -q "关键词" --json` 搜索待处理问题；查历史时加 `--status all`。用 `cairn issue show OBS-ID --json` 读取现象、关联文件和版本号。
- 修 Bug：按问题描述在当前仓库复现、修改代码并验证。修复后执行 `cairn issue fix OBS-ID --version N -m "实际改动和验证结果"`，将 OBS-ID 和 N 替换为刚才读取的 ID 和 version。验证失败时保持原问题待处理。
- 返回 `pending_local` 时，重连执行 `cairn sync`。遇到版本冲突，重新读取问题并核对最新内容。
4. **单独跟踪工作项时**
   - 开始编码任务：`cairn work start "<任务标题>" --agent claude-code|codex|agy [--component <组件ID>]`。
   - 关键节点、受阻或完成待验证时：`cairn work update <WI-ID> --status in_progress|blocked|pending_verification -m "<阶段进展说明>"`。
5. **需求管理与自动化测试闭环**
   - 查询需求与验收标准：`cairn req list [--module <模块>] [--priority P0] [--json]`、`cairn req show <REQ-ID>`；拆解工作项：`cairn req to-work <REQ-ID>`；勾选验收项：`cairn req ac <REQ-ID> --id AC-1 --done`。
   - 运行或上报自动化测试：`cairn test suites`、`cairn test run <SUITE-ID> --wait`、`cairn test report --suite <SUITE-ID> --junit report.xml`；查看逐条结果或失败转缺陷：`cairn test run-show <TRUN-ID>`、`cairn test to-bug <TRUN-ID> <TRES-ID>`。
   - 完整参数与详细流程请按需加载已安装的 **`cairn` Skill**（`cairn setup-agents --global` 可一键安装到 Claude Code / Codex / Antigravity）。
<!-- cairn:end -->
