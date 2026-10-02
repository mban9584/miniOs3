# 关于本仓库（miniOs3）

本仓库是 **MinIO 服务端源码的一个固定版本快照镜像**，不是 MinIO 官方仓库，也不是我自己写的东西。
官方仓库：<https://github.com/minio/minio>（注意：MinIO 是**对象存储服务器**，不是操作系统，仓库名里的 "Os" 只是代号）。

| 项目 | 值 |
|---|---|
| 上游版本 | `RELEASE.2025-04-22T22-12-26Z`（社区版 release tag） |
| 上游来源 | <https://github.com/minio/minio/archive/refs/tags/RELEASE.2025-04-22T22-12-26Z.tar.gz> |
| 语言 / 模块 | Go，`module github.com/minio/minio`，`go 1.24.0`（toolchain `go1.24.2`） |
| 许可证 | **AGPL-3.0**（`LICENSE`、`NOTICE`、`CREDITS` 全部原样保留，未做任何修改） |
| 文件数 | 1367 |
| 内容一致性 | 已逐个校验：1367 个文件的 git blob SHA-1 与上游 `RELEASE.2025-04-22T22-12-26Z` 的 tree **全部相同**（2026-09-23 用 GitHub Trees API 比对） |
| 提交历史 | 源码导入为 1 个快照 commit，后续提交记录本镜像的维护；源码归档不含上游的 commit / tag / PR 历史 |

**仓库首页那份 `README.md` 是 MinIO 官方的**（快速开始、容器安装、S3 API 兼容性说明等），我没有改动它。
本文件只记录"这个仓库是怎么来的、和上游有什么不同、怎么自己编译运行"。

---

## 一、和上游的差异（重要）

1. **没有 git 历史**：无法 `git blame`、无法切到其他版本、无法 `git pull` 上游更新。
   要升级就直接下载新的 tag 归档替换，或者另建仓库 `git remote add upstream https://github.com/minio/minio.git` 后重新打快照。
2. **`.gitignore` 用 `git add -A -f` 强制补齐了**。上游那份 `.gitignore` 里有 `minio`、`inspect`、`xattr`、
   `xl-meta*`、`hash-set`、`s3-check-md5*`、`*.gz` 等规则，本意是忽略**编译产物**；但源码归档里没有 git 索引
   （`.gitignore` 只对未跟踪文件生效），直接 `git add .` 会把被这些规则误伤的**上游已跟踪源文件**丢掉。
   实测命中 26 个：
   - `docs/debugging/{hash-set,healing-bin,inspect,pprofgoparser,reorder-disks,s3-check-md5,s3-verify,xattr,xl-meta}/`
     下的 22 个 `main.go` / `go.mod` / `go.sum` / `export.go` / `decrypt-v*.go` / `utils.go`
     —— 这些是自带 go.mod 的独立调试小工具子模块，缺了等于少一批工具
   - `cmd/testdata/xl-meta-{consist,merge,inline-notinline}.zip` 和 `cmd/testdata/xl.meta-corrupt.gz`
     —— 单元测试的固定输入，缺了 `go test ./cmd/...` 会失败
   本仓库已全部强制包含，所以是完整的 1367 个文件。
   两个反直觉的点：`helm/minio/` 整条链**没有**被 `minio` 规则误伤（上游靠 `.gitignore` 里的 `!*/` 把目录重新放开了）；
   而 `minio` 这条规则只匹配不带扩展名的名字，所以编译出的 **`minio.exe` 不会被忽略**——自己改代码提交前记得手动排除产物。
3. **`core.autocrlf` 在本仓库设为 `false`**。Windows 上全局 git 常是 `autocrlf=true`，那样 `git add` 会把
   文件里的 CRLF 清洗成 LF 再存——上游 `CREDITS` 里有 43 处真实 CRLF，第一次 add 就被改掉了一个文件
   （已用 `git add --renormalize CREDITS` 修正回上游字节）。其余 1366 个文件上游本来就是纯 LF，不受影响。

---

## 二、目录结构

```
go.mod / go.sum        Go 依赖声明（依赖很多，第一次 go build 会下载不少包）
main.go                入口，真正的实现在 cmd/
cmd/                   主体：S3 handler、admin API、`minio`/`mc` 命令行、内嵌 Console
  testdata/            单测固定数据（含 8.7MB 的大文件，仓库体积主要来自这里）
internal/              内部包：grid(节点通信)、bucket(生命周期/复制/版本)、config、kms、
                       s3select(SQL)、event(事件通知)、lock、amztime …
docs/                  文档：erasure、site-replication、iam、sts、metrics、debugging 工具
helm/minio/            社区版 Helm Chart（K8s 部署）
helm-releases/         打包好的 chart（.tgz）
buildscripts/          CI 用的脚本与测试语料
dockerscripts/         容器入口脚本
Dockerfile*            镜像构建（release / hotfix / old_cpu 等多份）
.github/workflows/     上游 CI 配置——文件原样保留。多数构建/测试只在目标为 `master` 的 PR 上触发；
                       depsreview.yaml 和 typos.yml 不限定 PR 目标分支。vulncheck.yml 的 push 仅限
                       `master`，所以推送到本仓库默认分支 `main` 不会触发它。issues.yaml 监听 Issue 事件。
                       lock.yml 原本每天定时运行，已在本仓库的 Actions 设置中停用（原因见下文）。
LICENSE / NOTICE / CREDITS   AGPL-3.0 正文、第三方声明（勿改）
```

### Actions 维护记录（2026-10-02）

2026-09-24 至 2026-10-02 的 9 次失败均来自 `Lock Threads`，不是 MinIO 编译或测试失败。
`lock.yml` 使用旧版 `dessant/lock-threads@v3`，其参数校验将 `github-token` 长度限制为 100 字符，
与当前 GitHub 自动提供的 token 不兼容，因此每天定时运行都会在参数检查阶段失败。

本仓库用于保存固定版本源码快照，不需要自动锁定旧讨论，已通过 GitHub Actions 设置停用该工作流。
这是仓库设置，不会改变上游工作流文件；在其他仓库导入此快照时，这个停用状态不会随文件复制。
如需恢复锁帖功能，应先升级插件并核对新版本参数及权限，再启用工作流。
历史失败记录保留供排查，停用后不再产生该任务的每日失败。

其他工作流继续保留原设置；需要上游 secrets 的任务在本镜像中可能无法运行。
如需停用某个工作流，可进入 Actions → 选择工作流 → 右上角菜单 → Disable workflow。
如需关闭全部 Actions，可进入 Settings → Actions → General → Actions permissions → Disable actions。

---

## 三、自己编译

需要 Go **1.24.0 或更高**（`go.mod` 里 `toolchain go1.24.2`，1.21+ 的 Go 会自动拉 toolchain）。

```powershell
# Windows
cd 仓库目录
go build -o minio.exe .          # 产物：当前目录下的 minio.exe
```

```bash
# Linux / macOS
go build -o minio .
```

跑上游的完整构建与静态检查（会先 `go install` golangci-lint 等工具）：

```bash
make getdeps    # 装 lint 等构建依赖
make build      # 构建并写入版本信息
make test       # lint + 全量测试（很慢，且部分用例需要多磁盘/Linux 特性）
make crosscompile
```

国内网络下载依赖慢的话先设代理：

```powershell
$env:GOPROXY = "https://goproxy.cn,direct"
```

> 网页控制台不在本仓库里：它来自依赖 `github.com/minio/console v1.7.6`（见 `go.mod`，全仓库没有一
> 处 `go:embed`）。所以**首次编译必须能访问 GOPROXY**，否则 S3 部分照样能编出来，但 Console 相关代码拉不到。

## 四、跑起来

```powershell
# 最简：单节点单盘，数据落在 .\data
.\minio.exe server .\data
```

启动后：

- S3 API 端点：<http://localhost:9000>
- 控制台（Console）：<http://localhost:9001>
- 默认账号 `minioadmin` / 密码 `minioadmin`

生产前至少要做两件事：设置 `MINIO_ROOT_USER` / `MINIO_ROOT_PASSWORD`（默认凭据谁都能登录），
以及用 4 盘 EC 模式或分布式多节点代替单盘（单盘没有任何冗余）：

```powershell
.\minio.exe server --address :9000 --console-address :9001 data{1...4}
```

Kubernetes 用 `helm/minio` 这个 chart；容器方式见官方 README。

## 五、只想要可执行文件的话

不必编译。本仓库**不含**任何编译产物，现成二进制在**上游的 GitHub Release 页**（该 release 有 48 个 asset，含签名与校验和）：

- 发布页：<https://github.com/minio/minio/releases/tag/RELEASE.2025-04-22T22-12-26Z>
- Windows amd64：<https://github.com/minio/minio/releases/download/RELEASE.2025-04-22T22-12-26Z/minio.windows-amd64.RELEASE.2025-04-22T22-12-26Z.exe>
- Linux amd64：<https://github.com/minio/minio/releases/download/RELEASE.2025-04-22T22-12-26Z/minio.linux-amd64.RELEASE.2025-04-22T22-12-26Z>
- 校验：<https://github.com/minio/minio/releases/download/RELEASE.2025-04-22T22-12-26Z/minio.windows-amd64.RELEASE.2025-04-22T22-12-26Z.exe.sha256sum>
  拿到后 `Get-FileHash .\minio.exe -Algorithm SHA256` 比对。

**别再按老文档去 `dl.min.io` 找**：`https://dl.min.io/server/minio/release/...` 这一整条路径（含
`windows-amd64/minio.exe` 和 `.../archive/minio.RELEASE.<版本>`）2026-09-23 实测全部返回 **410 Gone**，
只有 GitHub 上的 release 归档还能取到旧版本二进制。

## 六、许可与合规提醒

AGPL-3.0 是强 copyleft：如果你把这份源码改出来后**通过网络对外提供服务**，必须向使用者公开修改后的完整源码。
公司内部自行部署使用不受影响，但集成进商业产品/云服务前请先看清 `LICENSE`。
本镜像不改变任何许可条款，MinIO 相关商标归 MinIO, Inc. 所有。

---

*快照生成于 2026-09-23，来源为上面的 tag 源码归档，未修改任何上游文件；新增内容只有本 `MIRROR.md`。*
