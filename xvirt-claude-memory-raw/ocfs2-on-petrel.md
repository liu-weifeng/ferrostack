---
name: ocfs2-on-petrel
description: "OCFS2 shared FS on a petrel volume across 123+106 — setup, node creds, and the petrel gateway cache single-writer gotcha (--no-cache)"
metadata: 
  node_type: memory
  type: project
  originSessionId: 971c4a11-a171-41bc-95c9-04a8c2c9b62a
---

OCFS2 双机共享文件系统跑在 petrel 卷上(2026-06-11 落地,123+106):

**节点访问**:192.168.2.123(xvirt-standalone)/ 192.168.2.106(xvirt-node3)root 密码 `weifeng`。沙箱里 sshpass 不可用,用 `SSH_ASKPASS=/tmp/ap.sh SSH_ASKPASS_REQUIRE=force DISPLAY=:0 setsid -w ssh ... </dev/null`(参见 [[lycan-autoinstall-iso]])。

**关键发现 — petrel 网关缓存是单写者模型**:`petrel-ublk serve --mon`(远端模式)把后端包了 `CachingObjectAccess`(petrel-client/src/lib.rs,16MiB 脏/128MiB 读缓存,源码注释明说"单写者⇒无需失效")。两台主机同时 serve 同一卷时各自缓存互不失效 ⇒ 块级脑裂:OCFS2 心跳互相看不见,后挂载节点把活节点当死节点**回放其日志**(数据损坏)。诊断指纹:已被某网关读/写过的对象跨网关读到旧值,全新对象一致。

**修复**:petrel-ublk 加 `serve --no-cache`(bins/petrel-ublk/src/main.rs,ServeArgs + serve() 远端分支跳过 CachingObjectAccess 直达 OSD)。共享卷(OCFS2/GFS2 多网关)必须带它;VM 盘单网关照旧默认开缓存。已提交 petrel `c19f952`。

**部署形态**(两台同构):
- ocfs2-tools 1.8.9-1 经 `apt-get download + dpkg -i` 装(依赖 libdlm3/libcmap4/libcorosync-common4/libqb100;**apt install 不可用**——节点上 ovmf 缺 ovmf-generic 的破损状态挡 resolver,不能 --fix-broken 否则拉回归档 qemu)。内核 7.0.0 自带 ocfs2.ko,无需本仓 Rocky 专用 kmod。
- 卷:`ocfs2-shared` 20GiB thin,pool nas_data(id=1),mon 192.168.2.123:6790。
- systemd unit `petrel-ublk-ocfs2.service`(两台):`ExecStartPre=-modprobe ublk_drv` + `petrel-ublk serve --vol ocfs2-shared --mon 192.168.2.123:6790 --pool 1 --no-cache`,KillMode=process。设备号 /dev/ublkbN 不稳定,挂载用 LABEL。
- o2cb:/etc/ocfs2/cluster.conf 集群 `xvirtocfs2`,node0=xvirt-standalone(123)/node1=xvirt-node3(106),端口 7777(ufw 已放行);/etc/default/o2cb O2CB_ENABLED=true BOOTCLUSTER=xvirtocfs2;o2cb+ocfs2 服务已 enable。
- FS:`mkfs.ocfs2 -L ocfs2shared -N 4 --cluster-stack=o2cb --cluster-name=xvirtocfs2`(UUID `2eabde5d-0b73-4081-b8ae-1835a80147ff`),挂 **/datastore/ocfs2-shared**;fstab `UUID=2eabde5d... /datastore/ocfs2-shared ocfs2 _netdev,x-systemd.requires=petrel-ublk-ocfs2.service,x-systemd.device-timeout=30 0 0`。**坑:ocfs2 mount 不认 `nofail` 选项**(`Unknown parameter 'nofail'` → EINVAL),去掉它。
- **datastore 接入 xvirt**:两台都是 standalone(role.conf ROLE=standalone,未纳管 center,无 host_membership 行)。datastore 表 type enum 已含 ocfs2,map_agent_type ocfs2→shared_fs。但 ferrovirt 的 `create_datastore` 对 shared_fs **仍是占位**(storage_net.rs:353 直接拒;hyper-agent SharedFsBackend 全 NotImplementedError)——所以是**手动接入**:直接往各节点本地 DB `INSERT INTO datastore(...type='ocfs2'...) + host_datastore` 关联本机 host(name NOT LIKE `__%` 排除 `__self__` 占位),两台各插一次。经各自 `/api/vcenter/datastore` + detail 可见(capacity 21.5GB,shared=true,type OCFS2)。要产品化得实现 shared_fs backend 的 create/import_pool(mkfs.ocfs2 已在 LocalBackend 里有 `_mkfs_ocfs2`,可复用)+ ferrovirt 编排(多主机 import 挂载,类似 NFS 的 create→import 链)。
- 验证:106 dmesg `o2net: Connected to node xvirt-standalone` + 无活节点日志回放;双向写文件 md5 一致即时可见。

**坑**:① 106 的 OSD(4/5/6)当时全停,手动按 hyper-agent 命令行拉起(`petrel-osd demo --mon ... --osd-config /var/lib/petrel/deploy-out/osd.N.toml --addr-ip 192.168.2.106 --replicate --transport binary --pool-name nas_data --daemon`);OSD 中断窗口内写的对象副本曾分歧。② mkfs/挂载前务必先做双向裸块一致性测试(O_DIRECT 写读对比)。

**UNMAP/discard 支持(2026-06-11,petrel `4b84d65`,已推送)**:fstrim 在 OCFS2 上回收 petrel 精简卷空间。链路:OCFS2 fstrim → 内核 discard bio → petrel-ublk `UBLK_IO_OP_DISCARD` handler → `Volume::discard(off,len)` → 对被区间**完整覆盖**的数据对象 `remove_object`(副本/EC 全删,真回收;头尾残片不动,不广告 discard_zeroes_data)。abi 加 `UBLK_PARAM_TYPE_DISCARD`,set_params 广告粒度 4KiB/单请求上限 512MiB,DeviceParams 加 discard 字段。瘦克隆(有 parent)no-op。E2E:写 2GB→pool 2154 对象;rm+fstrim(19.4GiB trimmed)→1126 对象,回收 1028 对象/~4.17GB raw,跨主机一致。`petrel-mon usage --mon <ip>:6790` 报 pool 字节+对象数,是验证回收的手段。

**删文件+定向回收脚本(2026-06-11,petrel `a94c37b`,已推送)**:`scripts/ocfs2_reclaim_delete.py`(已装两节点 /usr/local/bin)。删 OCFS2 文件同时按文件 extent 位图定向 discard 回收 petrel 空间(OCFS2 没 -o discard 挂载,删文件不自动 trim;比 fstrim 全盘扫快、不动别的文件)。统一处理两情况:①独占文件→filefrag「空 flags」extent 全回收(unwritten 跳);②reflink 文件→shared extent 查 `debugfs.ocfs2 -R "refcount <fs相对路径>"` 树,只回收 refcount==1(本文件独占)交集,refcount≥2 共享块留给别的文件。**关键坑**:(a) discard 必须在 rm 之前(rm 后块会被重分配,再删误伤别人);(b) `blkdiscard` 工具对已挂载设备 O_EXCL→`Device or resource busy`,改用 **BLKDISCARD ioctl(0x1277,不带 O_EXCL 开块设备)**(参考脚本的 sg_unmap 同理绕过,但 ublk 非 SCSI 用不了);(c) refcount cpos→字节按 FS 真实 cluster size 换算(本卷 4K cluster,非设计文档的 1M)。实测:case1 独占 256MB→回收 122 对象/~513MB;case2 删 B 只回收独占 4MiB COW 段、A md5 不变。**局限**:petrel 按 4MiB 对象粒度回收(只删完整覆盖的对象),小/不对齐独占 free 区回收不到对象,大块连续 free 才有效。

**生产场景验证(1M cluster + SCSI + sg_unmap,2026-06-11,petrel `374e235`)**:设计文档(/tmp/空间回收方案设计.md)的真实场景是 OCFS2 在多路径 SCSI LUN、1M cluster、用 `sg_unmap`。脚本加 `--method {auto,sg_unmap,blkdiscard}`(auto:sg_readcap 成功=SCSI→sg_unmap,否则 blkdiscard);sg_unmap 的 lba/num 按设备逻辑块大小(blockdev --getss)换算 + 按 --chunk 切块 + --rate 限速。用 `modprobe scsi_debug dev_size_mb=1024 lbpu=1 lbpws=1`(/dev/sde,LBPU=1 支持 UNMAP)+ `mkfs.ocfs2 -C 1M --fs-features=local,refcount` 复现:1M cluster 的 cpos 换算 ÷256 正确(phys 37376×4096÷1M=cpos146 对上 refcount 树);删 reflink B 用 sg_unmap 只 UNMAP 独占 4MiB,`sg_get_lba_status` 该 LBA mapped→deallocated(SCSI 精简层真回收)、共享 12MiB 留、A md5 不变;B 删后 A 共享块 refcount 2→1,再删 A 全 16MiB 可回收。验证手段:scsi_debug + sg_get_lba_status 看 mapped/deallocated。
