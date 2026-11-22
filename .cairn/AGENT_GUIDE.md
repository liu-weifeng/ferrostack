# Cairn 工作区协作入口说明（面向 Claude Code / Codex / agy）

本工作区已接入 Cairn 工程上下文与协同工具。本地 Markdown 仍是正式工作材料，请直接阅读与编辑。

常用命令（在工作区任意子目录均可执行）：
- 开工简报（能力边界 + 活跃文档 + Web 新改动提醒 + 问题与进度）：`cairn brief [关键词] [--component 组件]`
- 安全拉取人在 Web 端修改或新建的文档：`cairn docs pull`（本地有未提交修改时自动存为 `.cairn-web.md` 绝不覆盖）
- 写完或更新设计/交接文档后立即推送共享：`cairn docs push <文件.md>`
- 搜索或读取精确文档章节：`cairn docs search "<关键词>"` / `cairn docs read <路径或文档ID>`
- 提 Bug：用 `cairn issue add --kind bug -t "标题" -m "复现步骤、实际与预期结果" --json` 保存发现，记下返回的 ID。多行正文可用 `--stdin`。
- 找 Bug：用 `cairn issue list --kind bug -q "关键词" --json` 搜索待处理问题；查历史时加 `--status all`。用 `cairn issue show OBS-ID --json` 读取现象、关联文件和版本号。
- 修 Bug：按问题描述在当前仓库复现、修改代码并验证。修复后执行 `cairn issue fix OBS-ID --version N -m "实际改动和验证结果"`，将 OBS-ID 和 N 替换为刚才读取的 ID 和 version。验证失败时保持原问题待处理。
- 返回 `pending_local` 时，重连执行 `cairn sync`。遇到版本冲突，重新读取问题并核对最新内容。
- 单独跟踪工作项时：`cairn work start "<任务标题>" --agent claude-code` / `cairn work update <WI-ID> --status in_progress -m "..."`
- 扫描全量 Markdown 修订 / 离线重连同步：`cairn docs refresh` / `cairn sync`
