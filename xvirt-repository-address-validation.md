# xvirt 仓库地址验证报告

验证时间：2026-07-18  
验证方式：`git ls-remote --exit-code <url> HEAD`，只读探测，不修改 remote。  
内网地址：`ssh://git@192.168.60.105:2222/liuwfng/<repo>.git`  
GitHub 地址：`git@github.com:liu-weifeng/<repo>.git`  

## 1. 结论摘要

- 新内网 Git 服务 `192.168.60.105:2222/liuwfng`：本次验证的 24 个仓库全部可访问。
- GitHub SSH `git@github.com:liu-weifeng`：18 个仓库可访问，5 个仓库返回 `Repository not found`，1 个仓库 `cdp-agent` 返回 `rc=2` 且无错误输出，需要单独排查。
- GitHub 是私有仓，必须用 SSH key 验证；HTTPS 会等待/失败，不适合作为本次验证方式。
- GitHub 上多个仓库 HEAD 与内网 HEAD 不一致，说明两端并非简单实时镜像，需要以后按仓库确认同步方向。

## 2. 内网 192.168.60.105 验证结果

全部 OK。

| 仓库 | 内网 HEAD |
|---|---:|
| xvirt | `4b09dac57337670692482eb7de241014be959606` |
| ferrovirt | `b1f3466869117e3fca4d4dd8619a72ae53b9691d` |
| xvirt-client | `47df377af4f1de0e0b1c2f8c5819e555caa716d3` |
| hyper-agent | `a7791047b763487c109e0e210ffed1e7540fcc04` |
| lictor | `2daa170a061dc6c216090a0e80ccb396253cd409` |
| concilium | `7019d5437f576ca44484f75a6eb07e2657240b40` |
| petrel | `0e28b36338396f981e81e74fd8e35bd3f391ec4c` |
| qemu | `0fdc2fe5b0fd7bf7142c728b48d4e632d0a6454d` |
| build-deb | `f5156a502e341c9f0ec5c8b5ca484ee5e236c3cb` |
| xvirt-autotest | `14910a479d2e1d4fef7a76ef1f3606b707e2db82` |
| aegis-sdk | `ac0c5f0ab422cff8240d370292dd941b1b5cd6bd` |
| cdp-agent | `498f026b0211d62c24d94ef7dae5a6218716a614` |
| fledge | `6ed351c8948caa136e4e2176b1f2522d33eac53e` |
| lycan | `2a8696b379a9a4a3355448e0439beaae43e739ef` |
| cdp-worker | `624a01f33656d8eba255bbeac3ca28e31c5e9488` |
| imgtools | `548f8c75c41324b0ad59a8f2d1982dc2f066a4e4` |
| krait | `6d8822cebb5a9c775b8d8348b3c29ffc5254921b` |
| ocfs2 | `86e3246191ec7f8277627663d46ec7e198f7c2a1` |
| snapshot-helper | `6ab1a4feb0b9c5695caade8822d0bc618ba84c59` |
| virt-control | `6bc631a7177bedba2d8b75cb46a8e43200b16718` |
| xvirt-blockdiff | `0b4c8da161569ce0407f55cf4aaf1957ae5fb353` |
| xvirt-libs | `59ee30cbe2f0a4366c1d62838d73a1de27a3d002` |
| xvirt-snapmig | `805411ee598f3be8647e1cad88f46a8474464ecc` |
| xvirt-tools | `787b231567ec0c6efee8c0c1d5fb72a41a1f816b` |

## 3. GitHub SSH 验证结果

### 3.1 可访问仓库

| 仓库 | GitHub HEAD | 与内网是否一致 |
|---|---:|---|
| xvirt | `9a6c451a3f3fd33e20f92ce94fd5d0e7e6062a6c` | 不一致 |
| ferrovirt | `b6e69ab4e89d9ae639f925ebbfd8c2a3dcd358d5` | 不一致 |
| xvirt-client | `358b8880c0a6b02f42da786adb4ef4f56a7a7657` | 不一致 |
| hyper-agent | `34c1574ec4d7bf2701599111fca1a24512d79bb5` | 不一致 |
| concilium | `4eeea1e41030fe001a085cc94c3755e6a884be18` | 不一致 |
| petrel | `d1fc04ad02e4242f2b1639434a9a32762afb3083` | 不一致 |
| qemu | `6e9d724e6fd2eb52f12966f5afb5186adf9412b5` | 不一致 |
| xvirt-autotest | `0d71b0695af946834625f48f00736fc2f297e4eb` | 不一致 |
| fledge | `6f82d03c644fac244f0999ed94e8aa0d340f48e5` | 不一致 |
| cdp-worker | `624a01f33656d8eba255bbeac3ca28e31c5e9488` | 一致 |
| krait | `de3a5e58052ac368f7572df18a4c702f7715fd1d` | 不一致 |
| ocfs2 | `ae425dc02c780e92ea9466732388348067b3ebf6` | 不一致 |
| snapshot-helper | `7fdea7d13a0bd5e96fcb7ea387dc1278743d8510` | 不一致 |
| virt-control | `a7277e0ee98a298810230e0cb0525fdf2754f4ea` | 不一致 |
| xvirt-blockdiff | `eb6c78e769d9b0b3c52b552e8a737064761b5eac` | 不一致 |
| xvirt-libs | `96469d9b6a6e68877022da3b71395d6931db6a3b` | 不一致 |
| xvirt-snapmig | `ed14e8f52fddefd91d0ba385734968caa5ea7491` | 不一致 |
| xvirt-tools | `a17b8f28b5f409da41a5b4f63fe486c5653bce95` | 不一致 |

### 3.2 GitHub 返回 Repository not found

这些地址用 SSH key 验证时明确返回：

```text
ERROR: Repository not found.
fatal: Could not read from remote repository.
```

| 仓库 | GitHub 地址 |
|---|---|
| lictor | `git@github.com:liu-weifeng/lictor.git` |
| build-deb | `git@github.com:liu-weifeng/build-deb.git` |
| aegis-sdk | `git@github.com:liu-weifeng/aegis-sdk.git` |
| lycan | `git@github.com:liu-weifeng/lycan.git` |
| imgtools | `git@github.com:liu-weifeng/imgtools.git` |

可能原因：

- GitHub 上尚未创建这些仓库。
- 仓库名称不同。
- 当前 SSH key 没有这些私有仓权限。
- 仓库在 GitHub organization/账户下路径不同。

### 3.3 GitHub 异常返回

| 仓库 | 结果 |
|---|---|
| cdp-agent | `git ls-remote` 返回 `rc=2`，没有错误输出 |

这不像普通的 `Repository not found`。建议后续用下面命令单独排查：

```bash
GIT_SSH_COMMAND="ssh -vvv" git ls-remote git@github.com:liu-weifeng/cdp-agent.git HEAD
```

## 4. 需要更新代码库索引的点

之前的 `xvirt-repository-index.md` 中 GitHub 地址按模板列出。经过验证后，应补充：

- GitHub 私有仓验证必须使用 SSH 地址。
- `lictor`、`build-deb`、`aegis-sdk`、`lycan`、`imgtools` 当前 GitHub 模板地址不可访问。
- `cdp-agent` 当前 GitHub 模板地址返回异常 rc=2，需要单独排查。
- GitHub 可访问仓库 HEAD 多数不同于内网 HEAD，不能假定两端一致。

## 5. 建议下一步

1. 在 GitHub 上确认缺失仓库是否存在、是否改名、是否未授予当前 key 权限。
2. 对 `cdp-agent` 开 SSH debug 单独排查。
3. 明确内网 `192.168.60.105` 与 GitHub 的同步策略：哪个是主、哪个是 mirror。
4. 如果 GitHub 是主远端，逐仓比对内网 HEAD 与 GitHub HEAD 的差异，避免覆盖未同步提交。
5. 更新 `xvirt-repository-index.md`，把“模板地址”改成“已验证/未验证/不可访问”状态。

