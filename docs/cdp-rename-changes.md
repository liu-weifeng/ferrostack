# CDP 命名整改与行为修复：变更说明

日期：2026-11-05

## 1. 状态

- **所有改动都还没有提交，也没有编译和测试过。** 开发机性能不足，验证留到测试机就绪后做，步骤见第 7 节。
- 改动分布在两个仓库的工作区：

| 仓库 | 位置 | 基线 |
|---|---|---|
| `qemu` | 新分支 `cdp-rename`，改动未提交 | `github/stable-11.1-dev`（8bd0b37d5b） |
| `cdp` | `main` 分支，改动未提交；目录改名已用 `git mv` 放入暂存区 | 2858e19 |

- 这份文档覆盖两类改动：命名整改（第 2、3 节），以及在整改过程中发现并修掉的五处行为问题（第 4 节）。

## 2. 为什么改名

- QEMU 侧的功能原来叫 "IO 重定向"（io-redirection），但写 IO 并没有被改道，只是在写成功后旁路复制一份到共享内存。功能本身、源文件、协议 magic 都叫 CDP，只有 QMP 命令叫 io-redirection。
- 同一个东西在 QEMU 代码里有六种缩写（`io_redirection`、`iord`、`rdio`、`REIO`、`IORD`、`ioredir`）。
- 外部记录进程的名字不统一：仓库叫 `cdp-worker`，二进制和 RPM 包叫 `cdpwork`，文档里三种写法混用；"worker" 也看不出它的职责。

整改后统一为：QEMU 侧叫 **CDP**，每块开启了 CDP 的磁盘对应一个 **CDP 通道**；外部进程叫 **cdp-recorder**。

## 3. 接口变化

### 3.1 QMP 命令

旧命令全部保留，参数和返回值不变，只是标记为 deprecated。上层可以按自己的节奏迁移。

| 旧命令 | 新命令 | 参数变化 |
|---|---|---|
| `io-redirection-add` | `cdp-attach` | `key`→`shm-key`；`size`（MiB）→`shm-size`（**字节**）；`fifo`→`notify-fifo`，改为可选；去掉 `enable` |
| `io-redirection-del` | `cdp-detach` | 无 |
| `io-redirection-update` | `cdp-update` | 参数名同 `cdp-attach`；`enable`→`enabled`；除 `device` 外都可选，不给的保持原值 |
| `io-redirection-query` | `query-cdp` | 返回新结构 `CdpInfo`，见下 |
| `io-redirection-sync` | `cdp-checkpoint` | `basetime`（本地时间字符串）→`timestamp`（纪元起的纳秒，可选，不给时由 cdp-recorder 取读到标记时的时间）；`enable`→`type`（`sync` 或 `fsync`，默认 `sync`） |
| `io-redirection-upgrade` | `cdp-resume` | `snapshot`→`snapshot-path` |

迁移时最容易出错的两点：

- `shm-size` 的单位是字节，旧接口的 `size` 是 MiB。
- `cdp-checkpoint` 的 `type` 只影响 cdp-recorder 给文件组起的名字（`_sync_` / `_fsync_`），QEMU 对两者的处理相同。

### 3.2 查询结果 `CdpInfo`

| 字段 | 含义 | 对应的旧字段 |
|---|---|---|
| `device` | 开启时给的设备名或节点名 | `device` |
| `filename` | 镜像文件名 | `pathname` |
| `enabled` | 是否在采集 | `enable` |
| `mode` | `data` 正常；`meta-only` 已降级 | 无（新增） |
| `shm-key` | 共享内存 key | `shmkey` |
| `shm-size` | 共享内存大小，字节 | `shmmaxsize` |
| `shm-used` | 尚未被读走的**字节数** | `shmused`（千分比） |
| `shm-state` | 共享内存状态字，调试用 | `shmstate` |
| `iov-count` | 已写入的 iovec 段和标记个数 | `ioniov` |
| `fsync-checkpoints` | 已插入的 fsync 标记个数 | `synccnt` |
| `notify-fifo` | 管道路径，未设置时不返回 | `fifo` |

旧结构里永远为空的 `error` 字段在新结构中去掉了。

### 3.3 cdp-recorder（原 cdpwork）

| 项目 | 旧 | 新 | 兼容处理 |
|---|---|---|---|
| 二进制 | `cdpwork` | `cdp-recorder` | RPM 保留 `/usr/local/bin/cdpwork` 软链接 |
| RPM 包名 | `cdpwork` | `cdp-recorder` | spec 中有 `Obsoletes` / `Provides: cdpwork` |
| 源码目录 | `worker/cdpwork/` | `recorder/cdp-recorder/` | 无 |
| socket 指令前缀 | `commond` | `command` | 旧拼写仍接受 |
| 长选项 | `--path` `--name` `--key` `--memorysize` `--FIFO` | `--log-dir` `--log-prefix` `--shm-key` `--shm-size` `--notify-fifo` | 旧长选项保留；短选项不变 |
| 阻塞选项 | `-e`（代码里有，但从未能用） | `-e <秒数>` / `--block-on-full <秒数>` | 见 4.4 |
| 测试环境变量 | `CDP_CDPWORK` | `CDP_RECORDER` | **不兼容**，需要改 |

其他变化：

- 文档 `docs/io-redirection.md` 改名为 `docs/qemu-cdp.md` 并重写，里面有同样的新旧名对照。
- `tools/generate_cdp_chain_config.py` 输出里的布局名 `cdpwork` 改为 `recorder`。
- 降级期间 socket `setlevel` 被拒绝时的提示文字改为指向 `cdp-resume`。

### 3.4 没有变的

- **共享内存线上协议**：布局、magic、带内标记字符串（`cdp_ioredir_sync`、`cdp_ioredir_fsyn`、`cdp_ioredir_upgr`、`cdp_level_lower`）都没动。
- **IO 日志、索引、位图的文件格式和文件名**。
- **cdp-recorder 的 socket 指令名和级别名**（`LOW` / `MIDDLE` / `HIGH`）。
- 文档中的"降级 / 升级"概念不变；"升级"对应新命令 `cdp-resume`。

## 4. 行为修复

### 4.1 管道错误标志的位置

- **之前**：QEMU 把"写管道失败"的标志写在阻塞字节之后 8 字节处，cdp-recorder 读的是之后 1 字节处，永远读不到。
- **现在**：QEMU 改为写在紧邻的 1 字节处。cdp-recorder 没改。
- **影响**：cdp-recorder 读到该标志后只打一条日志，没有其他动作。

### 4.2 `cdp-update` 更换管道

- **之前**：更换管道时打开的是旧路径，更换实际不生效；打开失败也不报错。
- **现在**：打开新路径，并且在改动任何状态之前先打开；打不开就报错，通道保持原样。
- **影响**：以前"静默成功"的调用现在可能返回错误，旧命令 `io-redirection-update` 也一样。

### 4.3 写零没有被采集

- **之前**：虚拟机的写零（write-zeroes）请求不会进 IO 日志，恢复出来的磁盘在这些区域保留旧数据。ext4 延迟初始化 inode 表、`blkdiscard -z`、`fallocate -z` 等都会走这条路径。
- **现在**：写零按 1 MiB 分块，每块作为一条数据全为 0 的普通记录写入共享内存。协议和 cdp-recorder 都不用改。
- **代价**：大范围写零会产生等量的 IO 日志，来不及记录时按现有规则降级。协议和文件格式里没有"零区间"记录，要做紧凑表示需要同时改 cdp-recorder、文件格式和恢复工具。

**discard 保持不采集**，这是有意的决定：

- discard 之后的内容对虚拟机是未定义的，QEMU 模拟的 virtio-blk、SCSI、IDE 都不承诺读回 0，备端保留旧数据同样有效。
- 如果把 discard 当成写零采集，虚拟机每次 `fstrim` 都会把整片空闲空间以全零数据灌进日志，基本必然触发降级。
- 副作用：在开启了 discard 透传的 raw 镜像上，被 discard 的区域主备逐字节比较可能不同。

### 4.4 阻塞模式

阻塞模式的本意是：共享内存写满时不立即降级，而是让虚拟机的写等一等 cdp-recorder。

- **之前**：这个模式从来没有生效过，原因有两个。cdp-recorder 的 `-e` 选项不在 getopt 的选项串里，无法从命令行开启；写入共享内存的阻塞标志拷贝自一个恒为 0 的地址。而 QEMU 侧的实现是持着全局锁无限空转，cdp-recorder 一旦卡死，虚拟机 IO 就挂死。
- **现在**：
  - cdp-recorder 用 `-e <秒数>`（0 到 255，默认 0 即不阻塞）指定等待上限，通过共享内存里原来的阻塞字节传给 QEMU。
  - QEMU 发现写不下时，释放 CDP 的锁并让出协程，每 1 毫秒重试一次；累计超过指定秒数仍写不下就降级。
  - 已降级的通道不再等待，所以 cdp-recorder 卡死时虚拟机最多被拖慢一次。
  - 等待期间其他磁盘的写入和 QMP 命令照常执行。
- **使用限制**：
  - 不要对旧版 QEMU 设置非 0 值，它会把任何非 0 都当作无限等待。
  - 秒数不宜设太大：等待中的写算未完成的 IO，暂停虚拟机、打快照这类需要排空 IO 的操作会跟着等。

非阻塞模式（默认）的降级规则没有变。

### 4.5 写钩子记录的数据范围

- **之前**：钩子把整个 qiov 当作写入内容，忽略了请求的长度和 qiov 内的起始偏移。
- **现在**：只记录这次写实际涉及的那一段。
- **影响**：虚拟机设备的写请求都是整个 qiov、偏移为 0，记录内容不变；受益的是带偏移的内部调用方。

## 5. 版本混用

| 组合 | 结果 |
|---|---|
| 新 QEMU + 旧 cdpwork | 可用。写零和管道错误标志的修复照常生效；旧 cdpwork 不会设置阻塞秒数 |
| 旧 QEMU + 新 cdp-recorder | 可用，但 `-e` 必须保持 0；旧 QEMU 的各项问题仍在 |
| 新 QEMU + 旧上层（用旧 QMP 命令） | 可用 |
| 旧 QEMU + 新上层（用新 QMP 命令） | 不可用，旧 QEMU 没有 `cdp-*` 命令 |
| 新 RPM + 按旧名启动的脚本 | 可用，靠 `cdpwork` 软链接和对 `commond` 的兼容 |

## 6. 没有做的事

- 独立的旧仓库 `cdp-worker`、`cdp-agent` 里对 cdp-worker 的引用、顶层的 `xvirt-*.md` 都没有改。
- cdp-recorder 的 socket 指令动词不统一（`getioinfo` 与 `levelinfo`，`setpriority` 与 `updaterpo`）、级别名含义不直观，这次没有动。
- `query-cdp` 的 `iov-count` 对"只写 qiov 一部分"的请求会略偏大，它只是统计值，没有再处理。
- 共享内存布局仍然在 QEMU 和 cdp-recorder 两边各定义一份，没有抽成公共头文件。

## 7. 验证清单

全部在测试机上执行。前四步是基础，后面按修复项逐条验证。

1. **QEMU 构建**：重点看 QAPI 生成是否通过，`block/cdp.c`、`blockdev.c`、`block/block-backend.c` 是否编译通过。
2. **cdp-recorder 构建**：`make -C recorder/cdp-recorder`，以及 `recorder/build.sh` 打 RPM。
3. **端到端回归**：`cdp/tests/vm` 全套（约 25 分钟），覆盖开启、同步点、降级、恢复记录和恢复盘比对。
4. **旧命令兼容**：手工用 `io-redirection-*` 六个命令各调一遍。
5. **写零**：在虚拟机里对 CDP 盘上一段有数据的区域写零（`blkdiscard -z` 或 `fallocate -z`），恢复后确认该区域为 0。
6. **更换管道**：用 `cdp-update` 把 `notify-fifo` 换到新管道，确认 cdp-recorder 继续收到通知；再换到不存在的路径，确认报错且原通道正常。
7. **阻塞模式**：
   - `-e 3` 启动 cdp-recorder 并对它发 `SIGSTOP`，虚拟机持续写：写入应在约 3 秒后恢复，`query-cdp` 的 `mode` 为 `meta-only`，期间 QMP 正常响应。
   - `-e 3` 下让虚拟机写入速度略高于记录速度：不应降级，写入被限速到记录速度。
   - `-e 0` 和不带 `-e`：共享内存写满时立即降级。
   - `-e abc`、`-e 256`：报错退出。

## 8. 涉及的文件

`qemu`（8 个文件，其中 `roms/seabios` 是原有的子模块差异，与本次无关）：

| 文件 | 内容 |
|---|---|
| `qapi/block-core.json` | 新增 `cdp-*` 命令和 `CdpInfo`、`CdpMode`、`CdpCheckpointType`；旧命令标记 deprecated |
| `block/cdp.h`、`block/cdp.c` | 内部命名统一为 `cdp_*`；第 4 节的全部 QEMU 侧修复 |
| `blockdev.c` | 新命令的处理函数；旧命令改为调用同一套实现 |
| `block/block-backend.c` | 写钩子区分普通写和写零，并传入长度和 qiov 偏移 |
| `block.c` | 调用点改名 |
| `cdp/cdp.md` | QEMU 树内的说明文档 |

`cdp`（57 个文件）：

| 范围 | 内容 |
|---|---|
| `recorder/`（原 `worker/`） | 目录、二进制、RPM 改名；内部类型改名；socket 前缀、长选项、`-e` 选项；阻塞字节的写入 |
| `docs/` | `qemu-cdp.md`（原 `io-redirection.md`，重写）、`design.md`、`format.md`、`cdp.md`、`vmware-recovery.md` |
| `tests/vm/`、`tests/test_generate_chain_config.py` | 改用新 QMP 命令、新二进制名、新环境变量 |
| `tools/`、`README.md`、`CMakeLists.txt` | 名称引用更新 |
