# virt-control ↔ hyper-agent 异步调用链路待改进项

记录 virt-control（Java/DEALER）与 hyper-agent（Python eventlet/ROUTER）之间的异步调用、进度上报、任务完成逻辑审查中发现的问题。

涉及文件：
- `virt-control/src/main/java/com/virt/control/zmq/AgentClient.java`
- `virt-control/src/main/java/com/virt/control/zmq/Operation.java`
- `virt-control/src/main/java/com/virt/control/zmq/OperationUpdate.java`
- `virt-control/src/main/java/com/virt/control/service/AgentClientManager.java`
- `hyper-agent/hyper_agent/operation.py`
- `hyper-agent/hyper_agent/operation_server.py`

## 关键问题（会直接导致功能错或挂起）

### 1. `query_operation` / `cancel_operation` 的 `operation_id` 取值位置不一致 ✅ 已修
- `operation_server.py:50, 72`：`_handle_query` / `_handle_cancel` 只读 `msg_dict.get('params', {}).get('operation_id')`。
- 但 `message_loop` (`operation_server.py:96-99`) 在最外层又强制要求顶层 `operation_id` 才不丢弃；Java 端 `AgentClient.checkOperationStatus` (`AgentClient.java:106-112`) 只在顶层带 `operation_id`，没放进 `params`。
- 结果：Java 客户端发的查询/取消会通过外层校验，但进入 handler 后 `params.operation_id` 为空，hyper-agent 回 `{"type":"error", ...}`。整个查询/取消通道是坏的。
- 修复：抽出 `_extract_target_operation_id`，先取 `params.operation_id`、回退到顶层 `operation_id`，`_handle_query` / `_handle_cancel` 都用它。错误提示同步改成 "operation_id is required"。

### 2. Java 端查找 `type=="query_response"`，但 hyper-agent 从不发这个 type ✅ 已修
- `AgentClient.java:255` 处理 `query_response`，但 `_handle_query` (`operation_server.py:67`) 实际回的是 `{"type":"update_operation", ...}`。
- 同时 Java `receiveMessages` 完全不处理 `type=="error"` 和 `type=="ack"` 两种合法回包，全部静默丢弃。
- 后果：上面问题 1 真发生时，hyper-agent 回的 `error` 包不会唤醒等待方，`execute()` 一直 poll 直到超时。
- 当前掩盖项：`statusChecker` 线程在 `AgentClient.java:75-82` 被注释掉，主动查询路径没用上，所以问题 1+2 的影响暂时只体现在异常路径。一旦打开 statusChecker 立刻翻车。
- 修复：
  - hyper-agent `_handle_query` 在 op 不存在时不再回 `error`，改回 `update_operation` + `status="not_found"`、`progress=100`、`code=1`、`message/error` 同步给出，不需要客户端引入新 type 分支即可拿到终态。
  - Java `receiveMessages` 删掉无用的 `query_response` 分支；改成 `update_operation` 主分支，并新增 `error`（仅记日志）和 `ack`（按 `operation_id` 路由出一条 `status=canceled`、`progress=100` 的终态 update 唤醒等待方）的处理。
  - 未识别 type 改为打 warn 不再静默丢。
  - 顺便把原先 `System.out.println("收到返回消息：：：：：："...)` 改成 `log.debug`，避免线上每条消息都打到 stdout。

### 3. `progress == 100` 作为完成判据过于脆弱 ✅ 已修
- Java `execute()` 唯一的"任务完成"判定是 `progress == 100`（`AgentClient.java:208-210`）。
- hyper-agent handler 拿到的是裸 `update_callback(progress=...)`，约定上能在中途汇报 100。任何 handler 不慎调 `progress=100` 但 status 还在 RUNNING，就会被 Java 当作完成提前返回，而真正的成功/失败 update 后到达时操作已经从 `operations` map 移除（`AgentClient.java:217`），事件被静默丢。
- 修复：在 `AgentClient` 加私有终态集合 `TERMINAL_STATUSES = {success, failed, canceled, cancelled}`（保留双 l 拼写兼容问题 9 修前的写法）；`execute()` 完成判定改为 `if (TERMINAL_STATUSES.contains(status)) return update;`，progress 仅用于展示。
- 边界处理：若终态 status 但 progress != 100、或 progress == 100 但 status 非终态，分别打 warn 日志（前者按终态返回，后者继续等下一条 update）。
- error / not_found / agent_down 三种 status 仍由前置分支抛异常，不进终态判定。

### 4. status="failed" 不抛 `OperationException` ✅ 已修
- Javadoc 上 `execute()` 声明"操作执行失败时抛 OperationException"。但 Java 只在 `update.getStatus()` 等于 `"error"` / `"not_found"` / `"agent_down"` 时抛（`AgentClient.java:187-199`）。
- hyper-agent 实际终态枚举是 `success / failed / canceled / cancelling`（`operation.py:14-20`），**根本没有 `error` 这个值**。失败会带 `status="failed"`、`progress=100`、`code=1`、`error="..."`。
- 后果：失败任务在 Java 看来是"成功返回 OperationUpdate"，调用方必须自己检查 `code`/`error` 字段——和文档/异常约定不符，所有依赖 try/catch 的上层逻辑都失效。
- 修复：
  - `OperationException` 增加可选构造器 `OperationException(String message, OperationUpdate update)` 和 `getUpdate()`，让上游能从异常上拿到原始 status/code/error/result。原 `(String)` / `(String, Throwable)` 构造器保留兼容。
  - `AgentClient.execute()` 在终态判定后检查：`status == "failed"` 或 `code != null && code != 0` 时抛 `OperationException`，message 优先取 `update.error`，回退到 `update.message`，再回退到 `操作 X 失败 (status=..., code=...)`。
  - canceled/cancelled 终态视为正常返回（取消是用户行为，不应当成异常）。
  - error / not_found / agent_down 仍由前置分支处理，不变。

### 5. `running_operations` 在 unknown action 路径泄漏 ✅ 已修
`operation.py:113-130`：
```python
controller = self.action_map.get(action)
if controller is None:
    self.update_operation(operation_id, status=FAILED, code=1, ...)
    return       # ← 没有 self.running_operations.discard(operation_id)
```
`_scheduler_loop` 在 pop 后已经 `running_operations.add(...)`，这条早返回路径没清理，集合无限增长（也会让未来引入的冲突检测失效）。
- 修复：在该早返回前补 `with self.lock: self.running_operations.discard(operation_id)`，与正常路径末尾 `running_operations.discard` 保持一致语义。

### 6. 取消 vs 调度的竞态 ✅ 已修
`_scheduler_loop` (`operation.py:103-111`) 拿锁后 `pop pending → add running → spawn_n → 释放锁`，但此时 `op.status` 仍是 `PENDING`。如果在 `_execute_operation` 真正跑起来之前，调用方 `cancel_operation` 拿锁，看到 `status == PENDING`，把它置成 `CANCELED`。然后 spawn 出来的 `_execute_operation` 不检查 status，直接调 handler 跑完，最后又把 status 覆盖回 `SUCCESS/FAILED`。
- 修复：
  - `_scheduler_loop` 在 pop 之后、release 锁之前立刻把 `op.status` 翻成 `RUNNING`，让随后到达的 `cancel_operation` 看到的不再是 PENDING（保持原来"running 不可取消"的语义），race 被序列化掉。
  - `_execute_operation` 入口加兜底：若拿到的 op 为 None 或 status == CANCELED，则在锁内 `running_operations.discard`、记日志后直接返回。覆盖未来可能绕过 scheduler 直接 spawn 的代码路径。

### 7. JeroMQ socket 多线程并发发送 ✅ 已修
`AgentClient.sendMessage` (`AgentClient.java:90-101`) 用 `socket.send("", SNDMORE) && socket.send(json)` 两帧。`AgentClientManager.executeOnAgent` 被 Spring web 多线程同时调用，多个线程会对**同一个 socket** 并发跑这两帧。JeroMQ 的 `ZMQ.Socket` 不是线程安全的，会出现两请求的帧交叉、ROUTER 端解不出 JSON、或者 `SNDMORE` 错配。
- 修复：`AgentClient` 新增私有 `ReentrantLock sendLock`，`sendMessage` 把 JSON 序列化留在锁外，锁仅串行化 `socket.send("", SNDMORE)` + `socket.send(json)` 两帧发送（finally 释放）。多个调用方并发时按 FIFO 串行写，避免帧交叉。
- 备注：receiver 线程仍单线程独占 `socket.recv`；JeroMQ 同一 socket 上 send 与 recv 的内部路径分离，不需要互斥。如未来想进一步硬化，可考虑把所有 socket 操作收敛到单一 IO 线程 + inproc PAIR 队列。

## 语义/类型一致性问题

### 8. bool=False 的失败码丢失 ✅ 已修
`operation.py:164-168`：
```python
elif isinstance(res, bool):
    update_info = {
        "code": 0,                                    # ← False 时也是 0
        "status": OperationStatus.SUCCESS if res else OperationStatus.FAILED,
    }
```
handler 返回 `False` 表示失败时，`code` 仍是 0。结合上面问题 4，Java 侧靠 code 判失败会判错。改成 `"code": 0 if res else 1`。
- 修复：`code` 改为 `0 if res else 1`，与 `(flag, result)` 二元组失败路径一致；与问题 4 修法相互呼应——Java 端遇到 status=failed 或 code≠0 都会抛 `OperationException`。

### 9. 枚举拼写不统一 ✅ 已修
`OperationStatus.CANCELED = "canceled"` 单 l，`OperationStatus.CANCELING = "cancelling"` 双 l（`operation.py:19-20`）。Java 这边 `OperationUpdate.status` 是 String，没有强约束，但任何字符串匹配（包括日志、前端）都会踩。
- 修复：
  - `operation.py`：`CANCELING = "canceling"` 单 l，与 `CANCELED` 对齐。
  - `AgentClient.java`：`TERMINAL_STATUSES` 从 `{success, failed, canceled, cancelled}` 收紧为 `{success, failed, canceled}`，原先误把双 l `cancelled` 当成 `canceled` 别名加入是错的——`cancelling/canceling` 是过渡态（CANCELING），不应归为终态。注释同步修正。
  - `entity/Task.java` 注释字段说明里 `cancelling` → `canceling` 同步修正。

### 10. `Operation.result` 类型注解与实际不符 ✅ 已修
dataclass 注解 `result: Optional[dict]`（`operation.py:44`），但 `_execute_operation` 把 `(code, result, error)` 里的任意 result 直接塞进去，常见就是 string/list。运行时无校验，但 IDE/类型检查会误导。
- 修复：改为 `result: Optional[Any] = None`，与运行时实际允许的类型一致。

### 11. Java `OperationUpdate.progress` 是 `int` 不是 `Integer` ✅ 已修
缺省值 0。如果 hyper-agent 发的 update 没有 `progress` 字段（理论上 `update_operation` 只在被显式调时设置 progress，但 dataclass `asdict(op)` 总是带），不会报错，但语义上 0 和 "未上报" 无法区分。
- 修复：`OperationUpdate.progress` 类型从 `int` 改为 `Integer`。`AgentClient.execute()` 内 `int progress = update.getProgress()` 改为 `Integer progressBoxed = update.getProgress(); int progress = progressBoxed != null ? progressBoxed : 0;` 避免 NPE；`LiveCheck` 里所有 `r.getProgress() == 100` 改成 `Integer.valueOf(100).equals(r.getProgress())` 防止 unboxing NPE。

## 鲁棒性问题

### 12. `abortOnAgentFailure=true` 实际不工作 ✅ 已修
`statusChecker` 线程被注释（`AgentClient.java:75-82`），`agent_down` 标记永远不会被设置。`execute()` 里 `operation.isAgentDown()` 永远 false（`AgentClient.java:156`）。Agent 真挂了，调用方只会 timeout，看不出 "agent down"。要么把 statusChecker 修好（先解决问题 1、2），要么把 `abortOnAgentFailure` 参数和这段代码删掉别误导调用者。
- 修复：选择"删死代码"路线。
  - 删除 `statusCheckerThread` 字段、构造器中已注释的启动块、`statusChecker()` 方法、`checkOperationStatus()` 方法。
  - `execute()` 内移除 `abortOnAgentFailure && operation.isAgentDown()` 检查、`status=="agent_down"` 分支。
  - 保留 `execute(action, params, timeout, statusCheck, abortOnAgentFailure)` 五参数签名以兼容现有调用方（`AgentClientManager.executeOnAgent`），但 `statusCheck` 与 `abortOnAgentFailure` 实参已不再有任何效果，Javadoc 标注已废弃。
  - `throws AgentDownException` 也保留以兼容旧 catch 语句，当前实现下不会再抛。
  - `Operation` 中 `agentDown` / `checkFailures` 字段没有再被读写，可在后续清理时与 `Operation` 类一起裁剪。

### 13. `abortOnAgentFailure=false` 时超时被无限延长 ✅ 已修
`AgentClient.java:163-170`：超时到了直接 `endTime = now + timeout*1000`，循环里持续刷新。等于"永远不超时"，且每过一轮 timeout 就打一条 warn 日志。要么删了这段，要么改成 N 次后真的抛。
- 修复：超时分支统一改成 `throw new TimeoutException("操作 " + operationId + " 超时 (" + timeout + "s)")`，不再依赖 `abortOnAgentFailure`，与问题 12 的清理一并落地。`timeout=null` 仍表示永久等待（走 `queue.take()`）。

### 14. Java 接收侧没有 `hasReceiveMore()` 校验 ✅ 已修
`receiveMessages` 直接连两次 `socket.recv()`（`AgentClient.java:227, 233`）。如果某次只收到 1 帧（异常发送方/socket 半开），第二次 recv 会按 `recvTimeOut` 阻塞，下一帧又被当成 identity，导致永久 desync。建议第一次 recv 后检查 `socket.hasReceiveMore()`，没有就丢弃。
- 修复：第一次 recv 拿到 identity 帧后立即 `if (!socket.hasReceiveMore()) { log.warn(...); continue; }`，再读 payload；丢弃单帧异常，避免永久 desync。

### 15. ZMQ socket 被多个 greenlet 并发写 ✅ 已修
hyper-agent 这边，多个 handler 协程都通过 `_update_operation_callback → _send → socket.send_multipart` 推 update。`eventlet.green.zmq` 的 socket 在协程间不会被抢占切换是约定俗成的（`send_multipart` 内部不会主动 yield 中途），多数情况下安全，但官方文档没保证。建议在 `_send` 上加 `eventlet.semaphore.Semaphore`。
- 修复：`OperationServer.__init__` 加 `self._send_lock = eventlet.semaphore.Semaphore()`，`_send` 内 `with self._send_lock: socket.send_multipart(...)` 串行化所有 ROUTER 发送路径。

### 16. `update_operation` 不在锁内 ✅ 已修
`operation.py:182` 直接读写 `self.operations[operation_id]`，没有 lock。eventlet 单线程下访问 dict 是原子的，问题不大；但同对象的多字段更新（progress、status、message）和 `cancel_operation` 翻 status 会交错可见。如果将来引入 conflict 检测/取消逻辑，要把这段 update 也包进 lock。
- 修复：`update_operation` 把字段更新 + asdict + caller 取出整段包进 `with self.lock:`，保证多字段一致快照；`response_callback`（涉及 ZMQ send，会让出协程）放到锁外执行，避免持锁等 IO。`op` 不存在时 warn 后直接返回（防御）。

## 建议的小步改动顺序

1. `operation_server.py` 里 `_handle_query` / `_handle_cancel` 改用 `msg_dict.get('operation_id') or msg_dict.get('params', {}).get('operation_id')`（修问题 1）。
2. `_handle_query` 在 op 为 None 时回 `update_operation` 而非 `error`，状态用 `not_found`，这样 Java 端不用引入新分支（半修问题 2）；同时 Java 端给 `error` / `ack` 加分支，把对应 operation 出队。
3. Java `execute()` 完成判定换成"`status in {success, failed, canceled}`"，并在非 success 时抛 `OperationException`（修问题 3、4）。
4. hyper-agent `_execute_operation` 在 controller 不存在的早返回里加 `with self.lock: self.running_operations.discard(operation_id)`（修问题 5）。
5. `_execute_operation` 入口检查 `op.status == CANCELED` 直接退出（修问题 6）。
6. Java `AgentClient.sendMessage` 加锁（修问题 7）。
7. bool=False 路径设置 `"code": 1`（修问题 8）。
8. 把 `OperationStatus.CANCELING` 的值改成 `"canceling"`，统一拼写（修问题 9）。
