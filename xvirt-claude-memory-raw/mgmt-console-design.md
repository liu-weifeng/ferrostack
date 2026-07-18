---
name: mgmt-console-design
description: "The claude.ai design project the xvirt management console is ported from (xvirt-mgmt), + how to access it via the claude_design MCP"
metadata: 
  node_type: memory
  type: reference
  originSessionId: dca19e25-4a8c-4fc3-928b-cb4ba9d1f85e
---

The management console (`xvirt-client/src/components/management/`) is ported from the
user's **claude.ai/design** project **"xvirt-mgmt"** — projectId `66e849b9-4b98-4b07-a706-9d1123c84387`
(type PROJECT, owner Liu Weifeng). Main file: `xvirt Management Console.html` (babel-standalone
prototype loading ~20 jsx modules).

**Access it** via the `DesignSync` MCP tool (claude_design connector). On first call it
auto-grants design-system scopes to the claude.ai login (no `/design-login` needed in this
session). `DesignSync get_project/list_files/get_file --projectId 66e849b9-...` reads the design
(this project is type PROJECT not DESIGN_SYSTEM, so writes/finalize_plan don't apply — read-only port).

**2026-06-19 port (xvirt-client `a297a00`, pushed):** the design's completed per-resource tabs —
`resource-tabs.jsx` (MonitorPaneV2 + PermissionsPaneV2) + `resource-config.jsx` (ConfigurePaneV2) —
were ported into `components/management/ResourceTabs.jsx`, swapping `window.XV_DATA`→`useInventory()`
(same `{root,flat,allHosts,tasks,events,alarms}` shape) and global Icon/LineChart→imports. Two-pane
CSS appended to `styles/management/mgmt.css`. Wired into `Console.jsx` 监控/配置/权限 dispatch
(replaced the basic MonitorPane/ConfigurePane/PermissionsPane in Tabs.jsx). Summary tabs were already
in Tabs.jsx.

**2026-06-19 follow-up (xvirt-client `cc49b02`, pushed + deployed 121/123/106):** the Monitor pane is
now wired to real API — `useInventory.buildInventory` maps `GET /api/cis/tasks` → design task shape
(newest-first cap 200; status success→completed, failed/error/canceled→failed), `GET /api/vcenter/alerts`
(active) + `/alerts/history` → alarm shape (deduped by uuid, critical→error else warning, acked =
status acknowledged|resolved). **Events has no backend API** — synthesized as a merged timeline of
terminal tasks + alert lifecycle (cap 300). refresh()'s Promise.all gained the 3 fetches; WS
`subscribeAllTasks`+`subscribeAlerts` added so panels stay live. Field names verified against center
.121 real data (task: create_at/finish_at/kind/message/object{name,type,uuid}/progress/status/task/user;
alert: uuid/title/severity/resource_name/host_ip/status/first_seen_at/last_seen_at/acked_by). Note:
network-task `object.name` is null backend-side → that task's Target column shows empty (harmless).

**Remaining caveat**: several Configure leaf panels are static vSphere-style descriptions reworded to
xvirt reality (no backing API). Design files NOT snapshotted into the repo — the claude.ai project is
canonical. Related: [[autotest-framework]].
