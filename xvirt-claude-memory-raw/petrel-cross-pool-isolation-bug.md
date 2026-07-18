---
name: petrel-cross-pool-isolation-bug
description: RESOLVED (branch multipool-tiering) — petrel OSD was single-pool; multi-pool data-path fix VALIDATED (same-name-cross-pool isolated + real tier placement). Phase B (recovery multi-pool) remains.
metadata: 
  node_type: memory
  type: project
  originSessionId: 3e93f53f-4faf-4927-8318-9562d4a000a4
---

## ✅ 已修复 + 验证(2026-06-14,petrel 分支 `multipool-tiering` f059714)
**Phase A 多 pool 数据路径**:`ObjReq` 各 op 带 `pool`;`NetworkObjectAccess` 发请求带 self.pool;OSD dispatch `*_in(resolve(pool))`;`PgManager` 加 `pg_num_of(pool)`/`route_in(pool,oid)` + `write_in/read_in/remove_in/clone_object_in/stat_in/list_oids_in`(worker 用 pg(含 pool)算 collection/复制/EC)。单 pool 路径 `route_in(self.pool)==route` 字节级不变(安全)。
**验证(全新干净 2 节点集群,osd1/osd3 改 ssd,fastpool=ssd)**:同名卷跨 pool 隔离 = pool1(hdd)写A/pool2(ssd)写B 各读回自己 ✓✓✓;真·tier 落盘 = 100MB 写 ssd 池 → 物理落 osd1(60MB)+osd3(40MB) ssd OSD,hdd OSD 0 数据字节 ✓;42 单测过。
**测试方法陷阱(致之前一堆假"损坏")**:`dd conv=fsync` 只刷到 ublk 设备,petrel-ublk 网关异步写缓存还没排到慢 OSD 就被 `systemctl stop` 杀 → 重开 serve 读到丢的写。**测 petrel 持久性 write 后须 sleep ~8s 再 stop**。也是真 durability 点:网关强杀丢未刷缓存(VM 连续运行影响小)。
**坑**:bincode 加字段=线协议不兼容(全集群同时换二进制);`pkill -f "petrel-..."` 匹配自己 shell 自杀(用 `pkill -x petrel-osd`/`-x petrel-mon`);`start_all` 重启不自动重起 OSD(手工 systemd-run 或全新重建)。
**✅ Phase B 已完成 + 验证(2026-06-14,commit fc3c937)**:`recover()` 去掉 `pg.pool != self.pool` 过滤 → 本 OSD 物理持有的每个 collection(任意 pool)都 load 进 worker 参与 recovery/peering/pg_temp(下游已按 pg.pool 算 acting set/复制/scrub)。**durability 实测(size=2 ssd tier 池)**:写 64MB → 杀 osd1(ssd@123)从 osd3 副本读回 SURVIVED → 重启 osd1 等 recover_loop 重新复制 → 杀 osd3(ssd@106)仍从 osd1 读回 RECOVERED(证明 tier 池被多 pool 恢复重新复制到回归 OSD)。37 单测过。
**多 pool 存储基础(Phase A+B)现已完整可用**。剩余小项(非 durability):per-pool usage 心跳上报(监控精度,OSD 总量打 cfg.pool 标签);tier 池 PG split(autoscale)未多 pool(tier 池固定 pg_num)。
**已完成并 push 到 main**:多 pool(merge fc3c937)+ P1 migrate 对象级启用(hyper-agent f713c62,实测 API E2E hdd→ssd md5 一致)+ P2 基础 `TieredObjectAccess`(petrel 6848bbc)+ P2 并发安全(7ce61ae,per-object 锁防 mover-vs-写丢写,20 轮并发测试过)。

**P2 已做(core,已测)**:`crates/petrel-client` `TieredObjectAccess<H,F>`:home+fast 后端 + per-object tier map(promoted 集合)+ 热度 EWMA;read/write 按对象当前层路由;promote/demote=对象级跨池拷贝,**per-object 异步锁**与 write 串行化防丢写;decay/heat_snapshot/promoted() 给 mover。3 单测(路由/promote/demote/热度/并发防丢写)。

**P2 剩余(集成,未做——需真 VM 数据完整性验证,勿仓促进活 I/O 路径)**:
1. tier map 持久化:把 promoted 集合存成小对象 `<vol>.tiermap`(serve 启动 read 载入 with_promoted,mover 变更时写回)。
2. serve 接入:petrel-ublk/vhost `serve` 加 `--fast-pool/--priority`,有 fast-pool 时走 no-cache 风格直连路径(避免缓存与 mover 跨池搬运的一致性问题),构造 `Arc<TieredObjectAccess>`(home=NetworkObjectAccess(home_pool),fast=同 fast_pool),Volume::open(arc),spawn mover。
3. mover 循环:周期 decay → 按 `promotion_score=heat×优先级权重` 排序 → 快层配额内 promote top、demote bottom(优先级 critical/high/normal/low 权重 ∞/4/1/0.25;竞争时高优先级独占)。
4. 编排:ferrovirt/agent 按卷开 tiering(下发 home/fast 池 + 优先级);P1 已存 vm_config.storage_priority/vm_disk.tier_policy。
5. distribution API 报 per-object 层分布(从 promoted 集合)。
6. **真 VM 验证**:guest 跑负载,热块上浮/冷块回落,迁移中并发 IO md5 一致(用 sleep-before-stop 避免网关缓存假象)。
路线建议:先 1+2 让 serve 能分层 + 持久;3 mover;cluster 级并发-dd 完整性测;再 4/5;最后真 VM。

---
## (历史)初次发现记录
petrel 存在一个数据隔离 BUG:**同一个卷名同时存在于两个 pool 时会损坏数据**,即便 collection key 名义上含 pool(`collection_of = (pool<<32)|ps`,见 `crates/petrel-pg/src/lib.rs:410`)。

**实测(123+106,2026-06-14)**:create iso2 在 pool1+pool2 → serve pool1 写 32MB 'A'(停)→ serve pool2 写 32MB 'B'(停)→ 重读 pool1 = **既非 A 也非 B 的乱码**(损坏,非简单别名)。对照:单 pool 内 write/stop/reread 持久正确(durtest PASS),不同卷名跨 pool 完全隔离(P0 ftest@pool4 / stest@pool5 OK)。

**影响**:这直接**阻断分层存储 P2**(设计 v3 §12)——
- 整卷跨层**迁移**(P1 的 migrate)不能用"同名跨 pool 拷贝"(ublk-dd 或对象级 `petrel-ublk migrate-pool` 都中招)。已在 ferrovirt/agent 侧禁用 migrate 数据通路。
- 每对象**热度提升/降级**(promote/demote)本质要让对象在保持身份(同名)的同时换 pool/层 → 必然踩此 bug。

**仍有效的**:P0(device-class 影子树放置)+ P1 放置(VM 盘按优先级建在某 tier pool,**唯一卷名、终生在一个 pool**)——不踩此 bug,已 shipped。

**根因已定位(2026-06-14,代码级)**:**petrel OSD 与线协议是单 pool 的**。
- bluestore 键是 `(coll, oid)`、coll=`(pool<<32)|ps`(`crates/petrel-bluestore/src/lib.rs:81` hkey),存储层本身按 pool 隔离 ✓;块分配器全局扫 extents 重建 free,也不会跨 coll 重叠 ✓。
- **但**:网络 `ObjReq::{Write,Read,...}` **不带 pool**(只有 epoch/oid/offset/data);`petrel-osd demo` 启动时 `--pool-name` 绑**一个** `cfg.pool`,`PgManager::spawn_replicated_with_log_keep(store, map, osd_id, cfg.pool, ...)` 把 manager 钉死单 pool(`PgManager.pool/pg_num` 是字段,`crates/petrel-pg/src/lib.rs:716`)。
- 于是不管客户端为哪个 pool 路由来的对象,OSD 都按 **自己那个 pool** 算 `collection_of(PgId{cfg.pool, ps})` 落盘、并按 **cfg.pool 的 acting set** 复制。→ 同名对象不同 pool 落进同一 coll/同一 key → 串扰损坏。也意味着 **P0 的"按 class 放置"在 map 层正确、但 OSD 数据层没真按 tier 落盘**(唯一卷名只是掩盖了串扰)。

**P2 前置修复 = 让 petrel 多 pool(OSD+协议)**:
1. 线协议 `ObjReq`(objectstore-trait / client_data 的网络枚举)各变体加 `pool: PoolId`;`NetworkObjectAccess` 发请求时带 `self.pool`。
2. `PgManager` 去掉单 `pool` 假设:`write/read/clone/remove/exists` 按**请求带的 pool** 算 `PgId{pool, ps}`(ps 用该 pool 的 pg_num)+ coll;复制 fan-out 用**该 pool** 的 acting set;worker 分片 `shard_of` 用 PgId(已含 pool)。
3. OSD 服务**所有**其 acting 含本 OSD 的 pool 的 PG:recovery/backfill/scrub/peering/心跳 usage 跨 pool 枚举。
4. 验证:同名卷跨 pool 隔离测试通过(本备忘开头那个测试)→ 再恢复 P0 真·tier 落盘验证 → 再做 P2。
这是 petrel **核心数据/控制面重构**(petrel-pg + objectstore-trait + petrel-osd + client_data + 复制/恢复/scrub/心跳),量大、风险高(数据路径),须分步 + 每步完整性校验。修好前不要在 pool-based 迁移/搬运上继续搭 P2。

**进展(2026-06-14,petrel 分支 `multipool-tiering` commit f059714):Phase A 已做(数据路径多 pool),Phase B 未做(恢复/peering),活集群验证被搅乱。**
- 已做(Phase A):网络 `ObjReq` 各 op 带 `pool`(`#[serde(default)]`,但 **bincode 是定位编码、非自描述——加字段=线协议不兼容,新旧混跑必坏**,须全集群同时换新二进制);`NetworkObjectAccess` 发请求带 self.pool;OSD dispatch 用 `*_in(resolve(pool))`;`PgManager` 加 `pg_num_of(pool)`/`route_in(pool,oid)` + `write_in/read_in/remove_in/clone_object_in/stat_in/list_oids_in`(原单 pool 方法委托,本地后端/测试/REPL 不变)。worker 用 pg(含 pool)算 collection/复制/EC,故路由对了下游就对。**42 单测过;单 pool I/O 行为与 main 完全相同(route_in(self.pool)==route)故对单 pool 安全。**
- 未做(Phase B):recovery/backfill/peering/scrub/心跳 usage 仍按 `self.pool` 枚举 —— 多 pool 的 PG 不会被恢复/对账(OSD 故障后非默认 pool 数据durability 不保证)。
- **活集群验证未果(被搅乱)**:在 123/106 部署新二进制时,先撞到混版(部分 OSD 还跑被删的旧二进制 → bincode 解码报错),force-kill 后**手工拉起 123 的 osd0/4/6**(start_all reconcile 不自动重起 OSD,是既有 reconcile 脆弱性 [[petrel-phase3-plan]])。这通暴力 OSD churn 后,**连单 pool 持久读都变 flaky**(rel1 OK/rel2 FAIL),= 副本经 violent kill/re-add 后 peering 没干净收敛(与本次代码无关,是 churn + Phase B 缺失)。故隔离测试结论不可信。**活集群当前处于退化/flaky 态(只测试数据,无重要数据)。**
- **下一步**:① 用新二进制**全新重建**petrel 集群(destroy+wipe+recreate,避免 churn 残留)→ settle 到 HEALTH_OK → 再跑同名跨 pool 隔离测试得干净结论;② 做 Phase B(恢复/peering/scrub 多 pool);③ 通过后恢复 P0 真·tier 落盘验证 + P2。main 分支 petrel 仍是干净单 pool 版,可回滚活集群到它。

**未提交的实验代码**:`petrel/bins/petrel-ublk/src/main.rs` 有 `migrate-pool` 子命令(对象/卷级跨池拷贝),因本 bug 不可靠,**留在工作树未 commit**。相关:[[petrel-phase3-plan]] / 设计 `distributed_storage_designv3.md` §12。
