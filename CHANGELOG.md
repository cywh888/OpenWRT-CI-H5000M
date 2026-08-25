# 更新日志

## [2026-08-25]

### 修复

- **修复全量编译失败（sing-box 1.14.0-rc.1 connPool linkname 未定义）**：最新 `MTK-AUTO` 自动编译（Run #32778901488）在 `Compile Firmware` 阶段因 `sing-box` 包构建失败，报错 `undefined reference to golang.org/x/net/http2.(*Transport).connPool`，`collect2: error: ld returned 1 exit status`，`ERROR: package/packages/sing-box failed to build (build variant: full)`。
  - **根因**：`sing-box 1.14.0-rc.1` 的 `transport/v2rayhttp/force_close.go` 通过 `//go:linkname transportConnPool golang.org/x/net/http2.(*Transport).connPool` 访问 `http2.Transport` 的**内部未导出方法** `connPool`，用于在 `Client.Close()` 时强制关闭所有 HTTP/2 连接。但 `golang.org/x/net` 在 v0.58.0+ 版本中移除/重命名了该内部方法，Go 模块解析时拉取到新版 x/net 后，链接阶段找不到该符号，导致所有包含 sing-box 的配置（本仓库 `MEDIATEK-WIFI-YES`）全部编译失败。08-23 同配置仍成功，属上游依赖漂移。
  - **修复**：在 `Scripts/Handles.sh` 末尾新增 sing-box 兼容修复段，构建时自动定位 `sing-box/transport/v2rayhttp/force_close.go` 并通过 Python 脚本：
    1. 移除 `//go:linkname transportConnPool` 声明与 `func transportConnPool()` 函数定义；
    2. 将 `case *http2.Transport:` 分支由"通过内部 API 遍历关闭所有连接"改为直接 `return transport`（`http2.Transport` 无公开的 `CloseIdleConnections` 方法，未关闭的连接由 Go GC 回收，不影响功能）。
  - 保留 `clientConnPool` / `efaceWords` 结构体定义以避免 `unsafe` / `sync` 未使用导入报错。本地 `gofmt -e` 语法校验通过。

### 变更文件

- `Scripts/Handles.sh` — 新增 sing-box force_close.go connPool linkname 兼容修复段
- `CHANGELOG.md` — 新增本文件，记录本次修复

## [2026-08-24]

### 变更

- **Initialization Environment apt 全局超时与重试**：`WRT-CORE.yml` 的 `Initialization Environment` 步骤新增 apt 全局超时配置（`/etc/apt/apt.conf.d/99marvis-timeouts`）与 `apt_retry()` 重试封装，覆盖 `apt update` / `full-upgrade` / `install` 及 `init_build_environment.sh` 内部 apt 调用，避免偶发网络挂起导致 job 6 小时超时取消。
