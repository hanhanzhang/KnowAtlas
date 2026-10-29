---
title: "Go 语言、工程实践与运行时"
domain: tech/languages
type: deep-dive
tags: [go, goroutine, gmp, garbage-collection, generics, modules, testing, profiling]
source: "自整理；参考 Go 官方文档（Go Spec / Effective Go / Release Notes / runtime 源码）与 Go 1.27 标准库文档"
updated: 2026-10-29
---

# Go 语言、工程实践与运行时

> **读者**：有 C/Java/Python 基础，或已能写 Go 但想弄清"为什么这样写、运行时到底做了什么"。
> **结构**：第一部分讲语言内核（1–11 章）；第二部分讲工程实践（12–19 章）；第三部分深挖 runtime（20–26 章）；附录 A–E 是速查与延伸阅读。
> **版本**：正文以 Go 1.27（2026-08）为基线；1.22–1.27 之间的语义变化会在正文中标注首次生效版本。
> **约定**：⚠️ 易错点｜💡 惯用法｜🧭 工程建议。短片段省略 `package`/`import` 时默认在 `main` 包。

---

## 目录

**第一部分　语言内核**
- [1. 工具链与第一个模块](#1-工具链与第一个模块)
  - [1.1 安装与目录](#11-安装与目录)
  - [1.2 `go mod init` 之后的目录](#12-go-mod-init-之后的目录)
  - [1.3 常用命令](#13-常用命令)
  - [1.4 构建、运行与交叉编译](#14-构建运行与交叉编译)
  - [1.5 编辑器与 gopls](#15-编辑器与-gopls)
  - [1.6 环境变量与 `GODEBUG`](#16-环境变量与-godebug)
- [2. 类型与值语义](#2-类型与值语义)
  - [2.1 基本类型与零值](#21-基本类型与零值)
  - [2.2 整数、溢出与转换](#22-整数溢出与转换)
  - [2.3 浮点与复数](#23-浮点与复数)
  - [2.4 字符串、rune、byte 与 UTF-8](#24-字符串runebyte-与-utf-8)
  - [2.5 数组、切片与底层数组](#25-数组切片与底层数组)
  - [2.6 map 的语义与陷阱](#26-map-的语义与陷阱)
  - [2.7 结构体、嵌入与内存布局](#27-结构体嵌入与内存布局)
  - [2.8 常量与 iota](#28-常量与-iota)
  - [2.9 类型定义、别名与可赋值性](#29-类型定义别名与可赋值性)
  - [2.10 指针、值接收者与指针接收者](#210-指针值接收者与指针接收者)
- [3. 表达式与控制流](#3-表达式与控制流)
  - [3.1 `if`、`for`、`switch` 的形态](#31-ifforswitch-的形态)
  - [3.2 `for range` 的六种形态](#32-for-range-的六种形态)
  - [3.3 `switch` 与 `type switch`](#33-switch-与-type-switch)
  - [3.4 标签、`goto` 与 `defer` 的边界](#34-标签goto-与-defer-的边界)
  - [3.5 多返回值与命名返回值](#35-多返回值与命名返回值)
  - [3.6 空标识符与常见陷阱](#36-空标识符与常见陷阱)
- [4. 函数、闭包与 `defer`](#4-函数闭包与-defer)
  - [4.1 函数是一等值](#41-函数是一等值)
  - [4.2 闭包与循环变量](#42-闭包与循环变量)
  - [4.3 `defer` 的三条规则](#43-defer-的三条规则)
  - [4.4 可变参数](#44-可变参数)
  - [4.5 方法值与方法表达式](#45-方法值与方法表达式)
  - [4.6 递归与"没有尾调用优化"](#46-递归与没有尾调用优化)
- [5. 接口与类型断言](#5-接口与类型断言)
  - [5.1 隐式实现](#51-隐式实现)
  - [5.2 `any` 与类型断言](#52-any-与类型断言)
  - [5.3 nil 接口 vs nil 指针（最经典的坑）](#53-nil-接口-vs-nil-指针最经典的坑)
  - [5.4 接口的内部表示（预告）](#54-接口的内部表示预告)
  - [5.5 接口设计原则](#55-接口设计原则)
  - [5.6 `errors.As` / `errors.Is` 与类型开关](#56-errorsas--errorsis-与类型开关)
  - [5.7 约束接口与 type set](#57-约束接口与-type-set)
- [6. 泛型](#6-泛型)
  - [6.1 类型参数与约束](#61-类型参数与约束)
  - [6.2 常用约束与 `~T`](#62-常用约束与-t)
  - [6.3 类型推断](#63-类型推断)
  - [6.4 泛型方法（1.27）](#64-泛型方法127)
  - [6.5 泛型还是接口](#65-泛型还是接口)
  - [6.6 常见错误与性能](#66-常见错误与性能)
- [7. goroutine 与 channel](#7-goroutine-与-channel)
  - [7.1 goroutine 的创建与生命周期](#71-goroutine-的创建与生命周期)
  - [7.2 channel 的语义](#72-channel-的语义)
  - [7.3 `select`](#73-select)
  - [7.4 四种基础并发模式](#74-四种基础并发模式)
  - [7.5 `context`：取消与传值](#75-context取消与传值)
  - [7.6 泄漏与死锁的四种形态](#76-泄漏与死锁的四种形态)
- [8. 同步原语与内存模型](#8-同步原语与内存模型)
  - [8.1 `Mutex` 与 `RWMutex`](#81-mutex-与-rwmutex)
  - [8.2 `sync.Once`](#82-synconce)
  - [8.3 `WaitGroup`](#83-waitgroup)
  - [8.4 原子操作与内存模型](#84-原子操作与内存模型)
  - [8.5 `sync.Pool`](#85-syncpool)
  - [8.6 `sync.Map`、`sync.Cond`](#86-syncmapsynccond)
  - [8.7 竞态检测](#87-竞态检测)
- [9. 错误处理](#9-错误处理)
  - [9.1 `error` 接口与哨兵错误](#91-error-接口与哨兵错误)
  - [9.2 包装、解包与合并](#92-包装解包与合并)
  - [9.3 自定义错误类型](#93-自定义错误类型)
  - [9.4 错误处理惯用法](#94-错误处理惯用法)
  - [9.5 `panic` / `recover` 的边界](#95-panic--recover-的边界)
  - [9.6 错误边界与对外映射](#96-错误边界与对外映射)
- [10. 反射与 `unsafe`](#10-反射与-unsafe)
  - [10.1 `reflect` 基础](#101-reflect-基础)
  - [10.2 可设置性（`CanSet`）](#102-可设置性canset)
  - [10.3 结构体标签](#103-结构体标签)
  - [10.4 性能与替代方案](#104-性能与替代方案)
  - [10.5 `unsafe` 的四条规则](#105-unsafe-的四条规则)
- [11. 标准库核心](#11-标准库核心)
  - [11.1 `strings` / `bytes` / `slices` / `maps`](#111-strings--bytes--slices--maps)
  - [11.2 `strconv` 与 `fmt`](#112-strconv-与-fmt)
  - [11.3 `time`](#113-time)
  - [11.4 `os` / `io` / `fs`](#114-os--io--fs)
  - [11.5 `net/http`](#115-nethttp)
  - [11.6 `encoding/json`](#116-encodingjson)
  - [11.7 `log/slog`](#117-logslog)
  - [11.8 `crypto` 的安全默认值](#118-crypto-的安全默认值)
  - [11.9 `iter` 与 range-over-func](#119-iter-与-range-over-func)

**第二部分　工程实践**
- [12. 模块与依赖管理](#12-模块与依赖管理)
  - [12.1 module 与 package](#121-module-与-package)
  - [12.2 `go.mod` 与 `go.sum`](#122-gomod-与-gosum)
  - [12.3 版本选择：最小版本选择（MVS）](#123-版本选择最小版本选择mvs)
  - [12.4 `replace`、`exclude`、`retract`](#124-replaceexcluderetract)
  - [12.5 vendor、私有模块与代理](#125-vendor私有模块与代理)
  - [12.6 依赖治理](#126-依赖治理)
  - [12.7 `tool` 指令（1.24+）](#127-tool-指令124)
- [13. 项目结构与分层](#13-项目结构与分层)
  - [13.1 三种常见布局](#131-三种常见布局)
  - [13.2 `internal` 的边界](#132-internal-的边界)
  - [13.3 包命名](#133-包命名)
  - [13.4 依赖方向与接口归属](#134-依赖方向与接口归属)
  - [13.5 `main` 包的职责](#135-main-包的职责)
  - [13.6 常见反模式](#136-常见反模式)
- [14. 测试体系](#14-测试体系)
  - [14.1 表驱动测试](#141-表驱动测试)
  - [14.2 `t.Helper` / `t.Cleanup` / `t.TempDir`](#142-thelper--tcleanup--ttempdir)
  - [14.3 子测试与并行](#143-子测试与并行)
  - [14.4 模糊测试](#144-模糊测试)
  - [14.5 基准测试](#145-基准测试)
  - [14.6 示例测试](#146-示例测试)
  - [14.7 集成测试与 `httptest`](#147-集成测试与-httptest)
  - [14.8 `testing/synctest`（1.25+）](#148-testingsynctest125)
  - [14.9 覆盖率与 flaky](#149-覆盖率与-flaky)
- [15. 质量闸门与 CI](#15-质量闸门与-ci)
  - [15.1 格式化](#151-格式化)
  - [15.2 `go vet`](#152-go-vet)
  - [15.3 `staticcheck` 与 `golangci-lint`](#153-staticcheck-与-golangci-lint)
  - [15.4 竞态与覆盖率](#154-竞态与覆盖率)
  - [15.5 最小 CI](#155-最小-ci)
  - [15.6 版本矩阵与模块校验](#156-版本矩阵与模块校验)
  - [15.7 本地一键闸门](#157-本地一键闸门)
- [16. 可观测性](#16-可观测性)
  - [16.1 结构化日志](#161-结构化日志)
  - [16.2 链路追踪](#162-链路追踪)
  - [16.3 指标](#163-指标)
  - [16.4 `pprof` 端点](#164-pprof-端点)
  - [16.5 panic 与优雅退出](#165-panic-与优雅退出)
- [17. API 设计与发布](#17-api-设计与发布)
  - [17.1 命名与包设计](#171-命名与包设计)
  - [17.2 选项模式 vs 配置结构体](#172-选项模式-vs-配置结构体)
  - [17.3 接受接口，返回具体类型](#173-接受接口返回具体类型)
  - [17.4 零值可用与 nil 语义](#174-零值可用与-nil-语义)
  - [17.5 兼容性：Go 1 承诺下的破坏性变更](#175-兼容性go-1-承诺下的破坏性变更)
  - [17.6 版本与 v2 路径](#176-版本与-v2-路径)
  - [17.7 文档与示例](#177-文档与示例)
- [18. 性能工程](#18-性能工程)
  - [18.1 先测量](#181-先测量)
  - [18.2 分配与逃逸](#182-分配与逃逸)
  - [18.3 字符串与字节](#183-字符串与字节)
  - [18.4 锁与并行](#184-锁与并行)
  - [18.5 PGO（1.21+）](#185-pgo121)
  - [18.6 工具链](#186-工具链)
- [19. 项目骨架模板](#19-项目骨架模板)
  - [19.1 目录](#191-目录)
  - [19.2 配置](#192-配置)
  - [19.3 错误边界](#193-错误边界)
  - [19.4 优雅退出](#194-优雅退出)
  - [19.5 `Makefile`](#195-makefile)
  - [19.6 验收清单](#196-验收清单)

**第三部分　运行时深挖**
- [20. 编译、链接与启动](#20-编译链接与启动)
  - [20.1 `go build` 的流水线](#201-go-build-的流水线)
  - [20.2 逃逸分析](#202-逃逸分析)
  - [20.3 内联](#203-内联)
  - [20.4 边界检查消除](#204-边界检查消除)
  - [20.5 链接与二进制布局](#205-链接与二进制布局)
  - [20.6 启动过程](#206-启动过程)
- [21. 调度器 GMP](#21-调度器-gmp)
  - [21.1 为什么需要三者](#211-为什么需要三者)
  - [21.2 关键结构](#212-关键结构)
  - [21.3 调度主循环](#213-调度主循环)
  - [21.4 work stealing](#214-work-stealing)
  - [21.5 系统调用与 netpoller](#215-系统调用与-netpoller)
  - [21.6 抢占](#216-抢占)
  - [21.7 栈增长与收缩](#217-栈增长与收缩)
  - [21.8 常见调度问题](#218-常见调度问题)
- [22. 内存分配器](#22-内存分配器)
  - [22.1 三级结构](#221-三级结构)
  - [22.2 size class 与 span](#222-size-class-与-span)
  - [22.3 tiny allocator](#223-tiny-allocator)
  - [22.4 分配路径](#224-分配路径)
  - [22.5 Go 1.27 的 size-specialized malloc](#225-go-127-的-size-specialized-malloc)
  - [22.6 碎片与 RSS](#226-碎片与-rss)
- [23. 垃圾回收](#23-垃圾回收)
  - [23.1 三色标记与混合写屏障](#231-三色标记与混合写屏障)
  - [23.2 GC 周期](#232-gc-周期)
  - [23.3 Pacer、`GOGC` 与 `GOMEMLIMIT`](#233-pacergogc-与-gomemlimit)
  - [23.4 辅助标记与 STW](#234-辅助标记与-stw)
  - [23.5 对象生命周期与清理](#235-对象生命周期与清理)
  - [23.6 调优方法论](#236-调优方法论)
- [24. channel、select 与锁的运行时实现](#24-channelselect-与锁的运行时实现)
  - [24.1 `hchan` 结构](#241-hchan-结构)
  - [24.2 发送路径](#242-发送路径)
  - [24.3 接收路径](#243-接收路径)
  - [24.4 `close` 的语义](#244-close-的语义)
  - [24.5 `select` 的实现](#245-select-的实现)
  - [24.6 `Mutex` 的两种模式](#246-mutex-的两种模式)
  - [24.7 信号量与 `netpoll`](#247-信号量与-netpoll)
- [25. 接口、itab 与 map 的运行时](#25-接口itab-与-map-的运行时)
  - [25.1 `iface` 与 `eface`](#251-iface-与-eface)
  - [25.2 `itab` 的构造与缓存](#252-itab-的构造与缓存)
  - [25.3 接口调用的成本](#253-接口调用的成本)
  - [25.4 `map` 的结构](#254-map-的结构)
  - [25.5 扩容与渐进搬迁](#255-扩容与渐进搬迁)
  - [25.6 遍历随机与并发检测](#256-遍历随机与并发检测)
  - [25.7 map 的替代方案](#257-map-的替代方案)
- [26. 诊断与版本演进](#26-诊断与版本演进)
  - [26.1 五类 profile](#261-五类-profile)
  - [26.2 trace](#262-trace)
  - [26.3 内存泄漏排查](#263-内存泄漏排查)
  - [26.4 goroutine 泄漏排查（1.27）](#264-goroutine-泄漏排查127)
  - [26.5 延迟毛刺归因](#265-延迟毛刺归因)
  - [26.6 `GODEBUG` 与升级 checklist](#266-godebug-与升级-checklist)
  - [26.7 不要依赖的实现细节](#267-不要依赖的实现细节)
- [附录 A　常用命令速查](#附录-a常用命令速查)
- [附录 B　Go 与 Rust 对照](#附录-bgo-与-rust-对照)
- [附录 C　Code Review 检查清单](#附录-ccode-review-检查清单)
- [附录 D　延伸阅读](#附录-d延伸阅读)
- [附录 E　术语表](#附录-e术语表)

---

# 第一部分　语言内核

## 1. 工具链与第一个模块

### 1.1 安装与目录

```bash
go version          # go version go1.27.0 darwin/arm64
go env GOROOT       # 标准库与工具链位置
go env GOPATH       # 工作区（默认 ~/go）
go env GOMODCACHE   # 模块缓存（默认 $GOPATH/pkg/mod）
go env GOCACHE      # 构建缓存
```

| 变量 | 作用 |
|---|---|
| `GOROOT` | Go 安装目录，一般不需要手动设置 |
| `GOPATH` | `bin/`（`go install` 产物）、`pkg/mod` 的父目录 |
| `GOBIN` | 覆盖安装目录，需在 `PATH` 里 |
| `GOMODCACHE` | 依赖源码缓存，可指向大磁盘 |
| `GOCACHE` | 编译缓存，删除后首次构建变慢但无副作用 |
| `GOFLAGS` | 默认参数，如 `-mod=readonly`、`-trimpath` |

⚠️ 不要提交 `GOPATH` 到代码或脚本里；CI 与本地路径不同，写死会挂。

### 1.2 `go mod init` 之后的目录

```bash
mkdir app && cd app
go mod init example.com/app
```

```text
app/
├── go.mod
├── go.sum          # 首次拉取依赖后生成
├── main.go
└── internal/       # 可选：只允许本模块导入
```

```go
package main

import "fmt"

func main() {
    fmt.Println("hello")
}
```

- **module** 是版本与依赖的边界；**package** 是编译与可见性的边界。
- 一个目录只能有一个 package（`_test` 与 `main` 例外）；目录名不必等于包名，但**强烈建议一致**。
- `internal/` 下的包只能被 `internal` 的父目录及其子目录导入，这是编译器强制的封装。

🧭 模块路径用真实可解析的域名（`example.com/app`），否则别人 `go get` 会失败。内部项目也应使用公司域名占位。

### 1.3 常用命令

| 命令 | 用途 |
|---|---|
| `go build ./...` | 编译所有包（不安装） |
| `go run .` | 编译并运行 main 包 |
| `go test ./...` | 跑测试 |
| `go test -race ./...` | 开竞态检测 |
| `go test -bench . -benchmem` | 基准测试 + 内存统计 |
| `go vet ./...` | 静态检查（编译器附带的"必查项"） |
| `go fmt ./...` | 格式化（等价 `gofmt -l -w`） |
| `go mod tidy` | 增删依赖到"刚好够用" |
| `go mod why -m X` | 为什么依赖 X |
| `go list -m all` | 列出最终选中的模块版本 |
| `go doc fmt.Println` | 查文档（1.27 支持 `pkg@version`） |
| `go tool pprof` | 分析 profile |
| `go tool trace` | 调度与延迟追踪 |
| `go clean -cache` | 清构建缓存 |
| `go install pkg@version` | 安装可执行文件到 `GOBIN` |
| `go get -tool pkg@version` | 1.24+：把工具加入 `tool` 指令 |

### 1.4 构建、运行与交叉编译

```bash
# 交叉编译：产物名带上目标平台，避免覆盖
GOOS=linux GOARCH=amd64 CGO_ENABLED=0 go build -trimpath -o bin/app-linux-amd64 .

# 静态二进制 + 缩小体积
CGO_ENABLED=0 go build -trimpath -ldflags="-s -w" -o bin/app .

# 查看会编译哪些包
go build -n ./... | head
```

| 变量 | 影响 |
|---|---|
| `GOOS`/`GOARCH` | 目标平台；`go tool dist list` 查全部 |
| `CGO_ENABLED=0` | 纯 Go 构建，产物不依赖 libc，便于放进 scratch 镜像 |
| `-trimpath` | 去掉构建机的绝对路径，保证可复现 |
| `-ldflags="-s -w"` | 去符号与 DWARF，体积减小但无法调试栈 |
| `-race` | 竞态检测（仅 `go test`/`go run`，不用于发布构建） |

⚠️ `CGO_ENABLED=0` 会改变 DNS 解析器（纯 Go 实现）与 `os/user` 的行为，容器里若依赖 NSS 需确认。

### 1.5 编辑器与 gopls

`gopls` 是官方语言服务器，提供补全、跳转、重命名、`go mod tidy` 提示、内联诊断。

最小配置要点：

- 用**模块根**打开项目，不要同时打开多个无关模块；
- 大仓库设置 `gopls.build.directoryFilters` 排除生成代码；
- 打开 `gopls.ui.semanticTokens` 与保存时 `gofmt`；
- `gopls` 的诊断包含 `vet` 与 `staticcheck` 的部分规则，比编辑器插件更准。

### 1.6 环境变量与 `GODEBUG`

| 变量 | 作用 |
|---|---|
| `GOMAXPROCS` | P 的数量；1.25 起默认尊重 cgroup 限额，仍可显式覆盖 |
| `GOGC` | GC 触发阈值（默认 100，即堆翻倍触发） |
| `GOMEMLIMIT` | 软内存上限，如 `GOMEMLIMIT=2GiB` |
| `GODEBUG` | 运行时行为开关，如 `gctrace=1`、`inittrace=1`、`http2debug=1` |
| `GOTRACEBACK` | panic 时打印的栈级别（`none`/`single`/`all`/`system`/`crash`） |
| `GOTOOLCHAIN` | `auto`（默认）/`local`/`go1.27.0` |

`go.mod` 里的 `go` 指令是**语义开关**（例如 1.22 的循环变量语义），不只是"最低版本要求"；`toolchain` 指定建议工具链，CI 里可用 `GOTOOLCHAIN=local` 禁止自动下载。`GODEBUG` 也可以写进 `go.mod` 的 `godebug` 段或源码的 `//go:debug` 注释，避免依赖部署环境。

🧭 生产容器里至少设置 `GOMEMLIMIT`（略低于容器上限），否则 GC 不知道 cgroup 边界，容易 OOMKilled。

---

## 2. 类型与值语义

Go 的类型系统比 Rust 简单，但"值语义 + 隐式接口 + 零值可用"这三件事决定了绝大多数惯用法。

### 2.1 基本类型与零值

| 类别 | 类型 | 零值 |
|---|---|---|
| 布尔 | `bool` | `false` |
| 整数 | `int8/16/32/64`、`uint8/16/32/64`、`int`、`uint`、`uintptr` | `0` |
| 浮点 | `float32`、`float64` | `0` |
| 复数 | `complex64`、`complex128` | `0+0i` |
| 字符串 | `string` | `""` |
| 指针/函数/接口/map/slice/channel | `*T`、`func`、`interface`、`map`、`[]T`、`chan` | `nil` |
| 结构体/数组 | `struct{}`、`[N]T` | 逐字段零值 |

`int`/`uint` 的宽度随平台（32 或 64 位）；`uintptr` 是"能装下指针的整数"，只用于 `unsafe` 与反射。

💡 **零值可用**是 Go 的设计目标：`var buf bytes.Buffer`、`var mu sync.Mutex`、`var wg sync.WaitGroup` 都能直接用。写类型时优先让零值有意义，能省掉构造函数与 nil 判断。

### 2.2 整数、溢出与转换

**溢出在 Go 里是静默回绕**，没有 debug/release 差异，也没有 panic：

```go
var x uint8 = 255
x++                  // 0
var y int8 = 127
y++                  // -128
```

需要检测时显式判断或用 `math/bits`：

```go
if x > math.MaxUint8-1 { /* 溢出 */ }
sum, carry := bits.Add64(a, b, 0)   // 无符号加法 + 进位
```

**转换必须显式**，而且 `int` 与 `int64` 之间也要写转换：

```go
var n int32 = 300
var m int64 = int64(n)     // ✅
// var k int64 = n         // ❌ 编译错误：不允许隐式转换
var b byte = byte(n)       // 44（截断低 8 位）
var f int = int(3.9)       // 3（向零截断）
var u uint = uint(-1)      // 巨大值（补码重解释）
```

⚠️ 三个高频事故：

1. `len(s)` 返回 `int`，在 32 位平台上大文件会溢出；
2. `int(a) * int(b)` 溢出后参与后续计算，结果全错；
3. 时间/金额用 `float64` 或过小的整型。

🧭 金额用最小货币单位（分）的 `int64`；跨平台长度计算用 `int64`，只在索引处转 `int`。

### 2.3 浮点与复数

```go
const eps = 1e-9
if math.Abs(a-b) < eps { /* 近似相等 */ }

// 比较与排序
sort.Slice(xs, func(i, j int) bool { return xs[i] < xs[j] })

// NaN 不等于自身
var nan = math.NaN()
fmt.Println(nan == nan)          // false
```

- `float64` 有 53 位有效位，能精确表示到 2^53 的整数；超过就别用浮点存 ID。
- 不要用 `==` 比较计算结果；需要全序排序时用 `math.Float64bits` 或专门库。
- 复数主要用于信号处理/数学；`complex128` 是两个 `float64`。

### 2.4 字符串、rune、byte 与 UTF-8

`string` 是**只读的字节序列**（通常是 UTF-8），`len(s)` 返回字节数而不是字符数。

```go
s := "你好, Go"
fmt.Println(len(s))              // 10（字节）
fmt.Println(utf8.RuneCountInString(s))  // 7（字符）

for i, r := range s {            // i 是字节偏移，r 是 rune
    fmt.Println(i, r, string(r))
}

for _, b := range []byte(s) {    // 按字节
    _ = b
}
```

| 类型 | 含义 | 典型用途 |
|---|---|---|
| `byte` | `uint8` 别名 | 二进制、IO |
| `rune` | `int32` 别名，一个 Unicode 码点 | 字符处理 |
| `string` | 只读字节序列 | 文本 |
| `[]byte` | 可变字节切片 | 构造/修改文本 |
| `[]rune` | 码点切片 | 按字符随机访问 |

```go
// 高频操作
strings.Builder{}                 // 拼接，避免 O(n²)
strings.Cut(s, ":")               // 1.18+，比 SplitN 更省
strings.CutLast(s, ":")           // 1.27+
strings.EqualFold(a, b)           // 大小写无关比较
strings.Contains/Index/ReplaceAll
```

⚠️ 用 `+=` 在循环里拼字符串会反复分配；用 `strings.Builder` 并 `Grow` 预估容量。

⚠️ `string` 与 `[]byte` 互转**通常复制**。热路径可以用 `unsafe.String`/`unsafe.Slice` 免拷贝，但前提是**生命周期严格受控**（源不被修改、不被回收）。

### 2.5 数组、切片与底层数组

```go
var arr [3]int              // 数组：长度是类型的一部分，值语义（赋值=拷贝）
s := arr[:]                 // 切片：{ptr, len, cap} 三元组，共享底层数组

s2 := s[1:3]                // len=2, cap=2（从索引 1 到底层数组末尾）
s3 := s[1:3:3]              // 三索引切片：强制 cap=2，防止 append 覆盖原数组
```

`append` 的行为取决于容量：

```go
a := make([]int, 0, 2)
b := append(a, 1)           // 未扩容：b 与 a 共享底层数组
c := append(a, 2)           // a 的 len 仍是 0，但底层数组第一个元素被改写
fmt.Println(b, c)           // [1] [2] —— b 的值被"偷改"
```

⚠️ 三个经典坑：

1. **切片共享**：函数内 `append` 后修改元素可能影响调用方；需要隔离时 `slices.Clone`。
2. **大数组切片泄漏内存**：`big[1000:]` 只保留小片段，但底层整个数组无法回收 → `slices.Clone` 一份。
3. **nil 切片与空切片**：`var s []int` 与 `s := []int{}` 都能 `append`、`len` 为 0，但 JSON 序列化分别是 `null` 与 `[]`。

```go
s := make([]int, 0, 100)    // 预分配容量，避免多次扩容
s = append(s, 1)
s = slices.Clip(s)          // 1.21+：把 cap 收紧到 len，释放多余内存
```

### 2.6 map 的语义与陷阱

```go
m := map[string]int{"a": 1}
v := m["missing"]           // 0（零值），不 panic
v, ok := m["missing"]       // ok=false
delete(m, "a")              // 删除不存在的键是安全的
```

| 行为 | 说明 |
|---|---|
| 读不存在的键 | 返回零值，不报错 |
| 写 nil map | **panic**：`assignment to entry in nil map` |
| 取元素地址 | 不允许：`&m["k"]` 编译错误（元素可能被扩容搬移） |
| 遍历顺序 | **随机**，每次不同（规范明确不保证） |
| 并发读写 | **运行时直接 fatal**，不是 panic，`recover` 抓不到 |

```go
// 需要"取地址"时：存指针，或先取出再放回
type Item struct{ N int }
m := map[string]*Item{}
m["a"] = &Item{N: 1}

// 需要顺序遍历：显式排序键
keys := slices.Sorted(maps.Keys(m))
for _, k := range keys { _ = m[k] }
```

⚠️ `map` 的并发读写会被 runtime 检测并直接 `fatal error: concurrent map read and map write`，进程无法恢复。要么加锁，要么用 `sync.Map`（只适合读多写少或键空间稳定的场景）。

### 2.7 结构体、嵌入与内存布局

```go
type User struct {
    ID    int64
    Name  string
    Email string
    age   uint8      // 未导出字段，同包可见
}

u := User{ID: 1, Name: "a"}     // 其余字段零值
p := &u
p.Name = "b"                    // 指针自动解引用
```

**嵌入（embedding）不是继承**，是字段提升：

```go
type Base struct{ ID int64 }
func (b Base) Describe() string { return "base" }

type Post struct {
    Base                        // 嵌入：Post 获得 ID 字段与 Describe 方法
    Title string
}

p := Post{Base: Base{ID: 1}, Title: "t"}
fmt.Println(p.ID, p.Describe()) // 提升访问
```

⚠️ 嵌入会把内部类型的方法集一起暴露，破坏封装；跨包嵌入尤其要谨慎。**优先组合（命名字段），只有确实要"是某种东西"时才嵌入。**

**内存布局**：字段按声明顺序排列，编译器插入填充以满足对齐（`int64` 8 字节对齐，`int32` 4 字节）。

```go
type Bad  struct{ A bool; B int64; C bool }   // 24 字节（两个填充）
type Good struct{ B int64; A, C bool }        // 16 字节
```

🧭 高频分配的结构体按"从大到小"排列字段能省内存；用 `unsafe.Sizeof` 或 `fieldalignment` 分析器验证。

### 2.8 常量与 iota

```go
const (
    StatusOK Status = iota   // 0
    StatusNotFound           // 1
    StatusError              // 2
)

const (
    _  = iota
    KB = 1 << (10 * iota)    // 1024
    MB                       // 1048576
    GB
)
```

- 常量是**无类型**的，只在需要时确定类型，因此 `const x = 1 << 62` 合法；
- `iota` 在每个 `const` 块内从 0 递增，遇到新块重置；
- 常量表达式在编译期求值，可用于数组长度与 `case` 标签。

💡 枚举用自定义类型 + `iota`，再配 `String()`（用 `stringer` 生成），能避免日志里出现裸数字。

### 2.9 类型定义、别名与可赋值性

```go
type UserID int64          // 定义：新类型，与 int64 不兼容（需显式转换）
type Alias = int64         // 别名：与 int64 完全相同

var a UserID = 1
var b int64 = int64(a)     // ✅ 显式转换
// var c int64 = a         // ❌
```

可赋值性规则（简化）：

1. 类型完全相同；
2. 一方是未命名类型且底层类型相同（如 `type MyInt int` 与 `int` 字面量）；
3. 接口：实现方可以赋给接口；
4. channel 方向：双向可赋给单向。

⚠️ 两个命名类型即使底层相同也不能直接赋值——这正是 newtype 的价值：`UserID` 与 `OrderID` 不会混用。

### 2.10 指针、值接收者与指针接收者

```go
type Counter struct{ n int }

func (c Counter) Inc()      { c.n++ }   // 值接收者：改的是副本，无效
func (c *Counter) IncPtr()  { c.n++ }   // 指针接收者：改原值

var c Counter
c.Inc()        // 编译通过，但没有效果
c.IncPtr()     // 编译器自动取地址：(&c).IncPtr()
```

方法集规则：

| 接收者 | `T` 的方法集 | `*T` 的方法集 |
|---|---|---|
| 值 `func (t T)` | 包含 | 包含 |
| 指针 `func (t *T)` | **不包含** | 包含 |

因此 `*T` 能满足的接口，`T` 未必能满足：

```go
type Incrementer interface{ IncPtr() }

var _ Incrementer = &Counter{}   // ✅
// var _ Incrementer = Counter{} // ❌ 值没有 IncPtr 方法
```

🧭 选择规则：

- 结构体含 `sync.Mutex`、切片/map 字段、或需要修改状态 → **指针接收者**；
- 小不可变值（`time.Time`、坐标）→ 值接收者；
- **同一类型的方法接收者保持一致**，混用会让方法集难以推理。

⚠️ 不要把 `nil` 指针接收者当成"安全默认"：方法内解引用 `nil` 会 panic，除非方法显式处理 `if c == nil`。

---

## 3. 表达式与控制流

### 3.1 `if`、`for`、`switch` 的形态

```go
if v, ok := m[key]; ok {        // 初始化语句 + 条件
    use(v)
} else if other() {
    // ...
} else {
    // ...
}

for i := 0; i < n; i++ { }      // 三段式
for cond { }                    // 只有条件（等价 while）
for { break }                   // 无限循环
```

Go 的条件**必须是 `bool`**，不存在"非零为真"：

```go
// if n { }        // ❌ 编译错误
if n != 0 { }      // ✅
```

💡 `if` 的初始化变量作用域仅限整个 `if/else` 链，能有效缩小变量生命周期，也帮助逃逸分析。

### 3.2 `for range` 的六种形态

```go
for i, v := range slice { }        // 下标 + 副本
for i := range slice { }           // 只要下标
for _, v := range slice { }        // 只要值（仍是副本）
for k, v := range m { }            // 键 + 值（顺序随机）
for i, r := range str { }          // 字节偏移 + rune
for v := range ch { }              // 通道：收到关闭为止
for i := range 10 { }              // 1.22+：0..9
for v := range seq { }             // 1.23+：range-over-func（iter.Seq）
```

⚠️ `range` 得到的是**副本**：修改 `v` 不会改原切片。要修改必须用下标：

```go
for i := range items { items[i].Done = true }
```

⚠️ 循环变量语义在 1.22 发生根本变化：

```go
// go 1.21 及更早：所有 goroutine 共享同一个 v，通常全打印最后一个
// go 1.22+（go 指令 ≥1.22）：每次迭代一个新变量，下面是安全的
for _, v := range items {
    go func() { fmt.Println(v) }()
}
```

老代码要兼容 1.21 时，必须显式 `v := v`；升级到 1.22 后这行可以删掉（但不是必须删）。

### 3.3 `switch` 与 `type switch`

```go
switch n {
case 1, 2:
    // 默认不穿透，无需 break
case 3:
    fallthrough          // 显式穿透到下一个 case
default:
}

switch {                 // 无表达式的 switch，等价 if/else 链
case n < 0:
case n == 0:
default:
}
```

`type switch` 是接口解包的标准方式：

```go
switch v := x.(type) {
case nil:
case string:
    fmt.Println(len(v))
case fmt.Stringer:
    fmt.Println(v.String())
case int, int64:          // 多类型时 v 仍是原接口类型
    fmt.Printf("%T\n", v)
default:
}
```

⚠️ `case int, int64` 中的 `v` 类型是接口本身，不能直接当 `int` 用；需要分别处理。

### 3.4 标签、`goto` 与 `defer` 的边界

```go
outer:
for i := 0; i < 3; i++ {
    for j := 0; j < 3; j++ {
        if i*j > 4 { break outer }      // 跳出外层
        if j == 2 { continue outer }    // 继续外层
    }
}
```

- `goto` 只能在同一函数内跳转，且**不能跳过变量声明**；
- `goto` 的合理用途是跳出深层嵌套或统一错误处理，但大多数情况用函数拆分更清晰；
- 标签名建议大写或以 `loop:` 形式表达意图。

### 3.5 多返回值与命名返回值

```go
func divmod(a, b int) (int, int) { return a / b, a % b }

// 命名返回值：可在函数体内直接赋值，也可被 defer 修改
func count() (n int, err error) {
    defer func() {
        if err != nil { n = 0 }
    }()
    return 1, nil
}
```

⚠️ 裸 `return` 只建议用于**很短的函数**；超过 10 行时，读者无法一眼看出返回了什么。

💡 多返回值的第二个值惯用 `ok bool` 或 `err error`，不要用 `int` 表示状态码。

### 3.6 空标识符与常见陷阱

```go
_, err := f()                  // 丢弃第一个返回值
for range ch { }               // 只关心"有事件"
var _ io.Reader = (*File)(nil) // 编译期断言接口实现
```

⚠️ 空标识符会掩盖错误：

```go
_ = json.Unmarshal(data, &v)   // 解析失败被静默丢弃
```

🧭 除非确定"失败也无所谓"，否则显式处理 `err`；lint 工具（`errcheck`）能强制这一点。

---

## 4. 函数、闭包与 `defer`

### 4.1 函数是一等值

```go
var f func(int) int = double
funcs := map[string]func(int) int{"double": double}
sort.Slice(xs, func(i, j int) bool { return xs[i] < xs[j] })
```

函数类型只由**参数与返回值**决定，与函数名无关。两个签名相同的函数可互相赋值。

### 4.2 闭包与循环变量

```go
func counter() func() int {
    n := 0                 // 逃逸到堆
    return func() int { n++; return n }
}

c := counter()
fmt.Println(c(), c(), c())  // 1 2 3
```

闭包捕获的是**变量本身**，不是值：

```go
x := 1
f := func() { fmt.Println(x) }
x = 2
f()                         // 2
```

配合 1.22 的循环变量语义，下面两种写法在 1.22+ 都安全：

```go
for _, v := range items {
    go func() { use(v) }()
}
for _, v := range items {
    v := v                  // 兼容旧版本的习惯写法，1.22+ 可省略
    go func() { use(v) }()
}
```

### 4.3 `defer` 的三条规则

1. **参数在 `defer` 语句处立即求值**；
2. **执行顺序是 LIFO**（后进先出）；
3. **可以修改命名返回值**（因为 `return` 先赋值再执行 defer）。

```go
for i := 0; i < 3; i++ {
    defer fmt.Println(i)     // 2 1 0
}

func f() (err error) {
    defer func() {
        if err != nil { err = fmt.Errorf("wrap: %w", err) }
    }()
    return io.EOF            // err 先被赋值为 io.EOF，再被 defer 改写
}
```

性能：Go 1.14 起，函数内 `defer` 数量少且不在循环里时使用 **open-coded defers**，开销接近直接调用；循环内的 `defer` 仍会堆分配。

⚠️ 常见错误：

```go
// 错误：defer 了函数调用，但参数已求值
defer resp.Body.Close()      // 正确
// defer func() { resp.Body.Close() }()  // 也可，但没必要

// 错误：在循环里 defer 文件关闭，句柄会堆积到函数返回
for _, p := range paths {
    f, _ := os.Open(p)
    defer f.Close()          // 上千个文件时可能耗尽 fd
}
```

🧭 循环内需要及时释放时，把循环体拆成函数，或显式 `defer` 一个闭包并在每轮结束前调用。

### 4.4 可变参数

```go
func sum(xs ...int) int {
    t := 0
    for _, x := range xs { t += x }
    return t
}

sum(1, 2, 3)
nums := []int{1, 2, 3}
sum(nums...)                 // 展开切片
```

- `xs` 在函数内是 `[]int`，可以直接 `append`、`range`；
- 调用 `sum(nums...)` **不会复制**底层数组，函数内修改会影响调用方；
- 可变参数必须是最后一个参数，且一个函数只能有一个。

### 4.5 方法值与方法表达式

```go
type Counter struct{ n int }

func (c *Counter) Inc() { c.n++ }

c := &Counter{}
inc := c.Inc            // 方法值：绑定接收者，等价 func()
inc()

expr := (*Counter).Inc  // 方法表达式：第一个参数是接收者
expr(c)
```

- 方法值在**取用时**就绑定了接收者（若是指针，绑定的是指针副本）；
- 方法值常用于把方法当回调传递，避免写 `func() { c.Inc() }`；
- 方法表达式较少用，但在泛型/反射场景需要"未绑定的方法"。

### 4.6 递归与"没有尾调用优化"

Go **不做尾调用优化**，递归深度受 goroutine 栈限制（初始 2 KB，按需翻倍，最大 1 GB）。

```go
func depth(n int) int {
    if n == 0 { return 0 }
    return 1 + depth(n-1)     // 每层占栈帧
}
```

深层递归（如解析嵌套 JSON、遍历大树）应改为显式栈：

```go
stack := []*Node{root}
for len(stack) > 0 {
    n := stack[len(stack)-1]
    stack = stack[:len(stack)-1]
    stack = append(stack, n.children...)
}
```

---

## 5. 接口与类型断言

### 5.1 隐式实现

Go 的接口是**结构化**的：类型不需要声明"实现了哪个接口"，只要方法集匹配即可。

```go
type Reader interface {
    Read(p []byte) (n int, err error)
}

type File struct{ /* ... */ }

func (f *File) Read(p []byte) (int, error) { /* ... */ }

var r Reader = &File{}      // 编译期检查：*File 有 Read 方法
```

编译期断言实现的惯用写法：

```go
var _ Reader = (*File)(nil)
var _ io.Closer = (*File)(nil)
```

好处是解耦：接口可以定义在**使用方**包里，实现方无需知道接口存在。

### 5.2 `any` 与类型断言

```go
var x any = "hello"

s, ok := x.(string)          // comma-ok：失败时 s 为零值，ok=false，不 panic
if !ok { /* 类型不符 */ }

n := x.(int)                 // 单返回值：失败直接 panic
```

| 写法 | 失败行为 | 何时用 |
|---|---|---|
| `v, ok := x.(T)` | `ok=false` | 不确定类型时（首选） |
| `v := x.(T)` | panic | 已确定类型（如内部约定） |
| `switch v := x.(type)` | 走 default | 多类型分支 |

⚠️ 断言 `nil` 接口会 panic（单返回值形式）；`var x any; x.(string)` 直接崩。

### 5.3 nil 接口 vs nil 指针（最经典的坑）

```go
type MyErr struct{}
func (e *MyErr) Error() string { return "my" }

func bad() error {
    var p *MyErr = nil
    return p                 // ⚠️ 返回的是"类型 *MyErr + 值 nil"的接口
}

func main() {
    err := bad()
    fmt.Println(err == nil)  // false！
}
```

原因：接口值由 **(动态类型, 动态值)** 两部分组成。`p` 是 `nil` 指针，但装进 `error` 后类型部分是 `*MyErr`，因此接口本身不为 nil。

正确写法：

```go
func good() error {
    return nil               // 显式返回 nil 接口
}
```

🧭 函数返回接口时，**永远不要返回一个"值为 nil 的具体类型指针"**；如果内部可能为 nil，先判断再返回 nil。

### 5.4 接口的内部表示（预告）

```go
// 非空接口：itab（类型 + 方法表）+ 数据指针
type iface struct {
    tab  *itab
    data unsafe.Pointer
}

// 空接口：类型元数据 + 数据指针
type eface struct {
    _type *_type
    data  unsafe.Pointer
}
```

- 把具体值装进接口通常需要**装箱（boxing）**：小整数可能被静态表优化，大对象会分配；
- 接口调用是**间接调用**（通过 itab 的函数指针），无法内联；
- `itab` 有全局缓存，重复的 (接口, 类型) 组合不会重复构造。

性能细节见 25 章。

### 5.5 接口设计原则

```go
// 好：小接口，职责单一，定义在使用方
type Fetcher interface { Fetch(ctx context.Context, url string) ([]byte, error) }

// 差：大接口，实现方被迫实现用不到的方法
type Service interface {
    Fetch(...)
    Save(...)
    Delete(...)
    Migrate(...)
}
```

🧭 三条经验：

1. **接口越小越好**：`io.Reader`/`io.Writer` 只有一个方法，才能被无数类型实现；
2. **接口定义在使用方**，不要为了"分层"在实现方预先造接口；
3. **返回具体类型，接受接口**（"accept interfaces, return structs"）——调用方能用到全部方法，也便于后续扩展。

⚠️ 不要为了 mock 而给每个类型都造接口：接口会带来装箱与间接调用成本；只在**确实需要替换实现**（测试、多后端）时引入。

### 5.6 `errors.As` / `errors.Is` 与类型开关

```go
var netErr net.Error
if errors.As(err, &netErr) && netErr.Timeout() { /* ... */ }

if errors.Is(err, os.ErrNotExist) { /* ... */ }

var pe *fs.PathError
if errors.As(err, &pe) { fmt.Println(pe.Path) }
```

- `errors.Is` 沿 `Unwrap()` 链比较**值**（支持自定义 `Is` 方法）；
- `errors.As` 沿链查找**类型**并赋值，目标必须是指针；
- 判断具体类型用 `errors.As`，不要用类型断言 `err.(*T)`——后者不会解包。

### 5.7 约束接口与 type set

泛型约束本质是"允许的类型集合"：

```go
type Number interface {
    ~int | ~int32 | ~int64 | ~float64
}

func Sum[T Number](xs []T) T {
    var t T
    for _, x := range xs { t += x }
    return t
}
```

- `~T` 表示"底层类型为 T 的所有命名类型"，`~int` 能匹配 `type UserID int`；
- `|` 是并集，多个约束行是交集；
- `comparable` 是预声明约束，表示可用 `==`/`!=` 比较；
- 约束接口**不能作为值类型**使用（不能声明 `var n Number`）。

⚠️ 约束里混入方法会导致"类型集合无法表达"的编译错误，例如 `interface{ ~int; String() string }` 的交集可能为空，编译器会拒绝某些操作。

---

## 6. 泛型

Go 1.18 引入类型参数，1.24 支持泛型类型别名，1.27 支持泛型方法。设计目标是"用约束表达可操作的类型集合"，而不是 C++ 式的模板元编程。

### 6.1 类型参数与约束

```go
func Map[T, U any](xs []T, f func(T) U) []U {
    out := make([]U, 0, len(xs))
    for _, x := range xs {
        out = append(out, f(x))
    }
    return out
}

nums := Map([]int{1, 2, 3}, func(i int) string { return strconv.Itoa(i) })
```

约束写在类型参数后面，可以是 `any`、`comparable`、具名接口或字面接口：

```go
type Number interface{ ~int | ~float64 }

func Sum[T Number](xs []T) T {
    var zero T
    for _, x := range xs { zero += x }
    return zero
}
```

### 6.2 常用约束与 `~T`

| 约束 | 含义 | 能做什么 |
|---|---|---|
| `any` | 任意类型 | 只能赋值、传递、比较指针 |
| `comparable` | 可用 `==`/`!=` | 可作 map 键、可比较 |
| `~int` | 底层类型为 `int` 的命名类型 | 算术、比较、位运算 |
| `interface{ ~int \| ~string }` | 并集 | 按分支处理 |
| `cmp.Ordered` | 有序类型（1.21+） | `<`、`>`、排序 |

```go
func Max[T cmp.Ordered](a, b T) T {
    if a > b { return a }
    return b
}

type Celsius float64
Max(Celsius(1), Celsius(2))     // ✅ ~float64 匹配命名类型
```

⚠️ `comparable` 不保证可作为 map 键的**运行时**安全：接口类型的动态值若包含 slice/map，比较会 panic。

### 6.3 类型推断

```go
Map([]int{1}, func(i int) string { return "" })   // T、U 均可推断
Max(1, 2)                                          // T=int
Max[int8](1, 2)                                    // 显式指定
```

推断规则（简化）：从实参类型匹配形参类型，支持函数实参的**上下文推断**；不能从返回值推断。

```go
// ❌ 无法推断 T（返回值不参与推断）
// x := zero()          func zero[T any]() T
x := zero[int]()        // ✅ 显式
```

### 6.4 泛型方法（1.27）

方法可以声明自己的类型参数，而不只是使用接收者的类型参数：

```go
type Pair[A any] struct{ First, Second A }

// 方法级类型参数 B，与接收者的 A 独立
func (p Pair[A]) Map[B any](f func(A) B) Pair[B] {
    return Pair[B]{First: f(p.First), Second: f(p.Second)}
}

p := Pair[int]{1, 2}
s := p.Map(strconv.Itoa)      // Pair[string]
```

限制：

- **接口的方法不能声明类型参数**；
- **接口方法不能被泛型方法实现**（因此泛型方法无法满足需要该方法的接口）；
- 泛型方法不能与接收者类型参数重名。

🧭 泛型方法适合"类型转换/映射"这类与接收者元素类型无关的操作；如果新类型参数要与接收者协同变化，通常用包级泛型函数更清晰。

### 6.5 泛型还是接口

| 维度 | 泛型 | 接口 |
|---|---|---|
| 分发 | 编译期实例化（stenciling + GC shape） | 运行期 itab 间接调用 |
| 性能 | 通常更好，可内联/去装箱 | 有装箱与间接调用成本 |
| 异构集合 | 不支持（`[]T` 类型固定） | 支持（`[]Shape`） |
| 代码体积 | 每个类型组合一份（部分共享） | 一份 |
| 表达能力 | 需要满足约束的静态类型 | 任意实现者 |

🧭 判断标准：

- **元素类型固定、要榨性能** → 泛型（`slices.Sort`、`maps.Clone`）；
- **需要在运行时替换实现 / 存异构对象** → 接口；
- 两者可以组合：`type Cache[K comparable, V any] struct { store Store[V] }`。

### 6.6 常见错误与性能

```go
// ❌ 约束缺失：any 不能做算术
// func Add[T any](a, b T) T { return a + b }

// ✅
func Add[T ~int | ~float64](a, b T) T { return a + b }
```

⚠️ 三个易错点：

1. **方法集不满足约束**：`T` 是值类型时只能调用值接收者方法；要调用指针方法得用 `*T`；
2. **`nil` 不能赋给类型参数**：`var zero T` 得到零值，但 `T` 可能是 `int`，不能 `return nil`；
3. **泛型不能用于方法上的"重载"**：Go 没有重载，同一类型不能有两个同名泛型方法。

性能上，Go 采用 **GC shape stenciling**：指针形状相同的类型共享同一份实例化代码，因此 `[]*T` 的不同 `T` 可能共享代码，而 `int` 与 `string` 各一份。这通常比完全单态化省体积，但牺牲了一点特化优化。

---

## 7. goroutine 与 channel

### 7.1 goroutine 的创建与生命周期

```go
go func() { doWork() }()      // 启动后立即返回，不等待
```

- 初始栈 2 KB，按需增长（最大 1 GB），因此"开十万个 goroutine"是可行的；
- goroutine 没有 ID、没有 join 返回值、不能从外部 kill；
- **只有函数返回或 panic 才会结束**；`runtime.Goexit` 也会结束（会执行 defer）。

⚠️ `main` 返回时所有 goroutine 立即消失，不会等待。需要等待就用 `sync.WaitGroup`、`errgroup` 或 channel。

### 7.2 channel 的语义

```go
ch := make(chan int)        // 无缓冲：发送阻塞直到有接收方
ch := make(chan int, 10)    // 有缓冲：缓冲满时发送阻塞
ch <- 1                     // 发送
v := <-ch                   // 接收
v, ok := <-ch               // ok=false 表示已关闭且缓冲为空
close(ch)                   // 关闭：只能由发送方关闭
```

| 操作 | nil channel | 已关闭 channel |
|---|---|---|
| 发送 | 永久阻塞 | **panic** |
| 接收 | 永久阻塞 | 立即返回零值，`ok=false` |
| 关闭 | **panic** | **panic**（重复关闭） |

规则：

1. **谁发送谁关闭**；接收方永远不要关闭；
2. 关闭是"广播"：所有阻塞的接收方同时被唤醒；
3. 向已关闭的 channel 发送会 panic，因此关闭前要确保没有其他发送方。

```go
// 生产者关闭
go func() {
    defer close(out)
    for _, x := range xs { out <- x }
}()
for v := range out { use(v) }
```

🧭 **无缓冲 channel 是同步点**（发送方等到接收方取走）；有缓冲 channel 是队列。需要背压时用有界缓冲，不要用无界队列（Go 没有无界 channel，`len` 很大的缓冲等于隐藏内存风险）。

### 7.3 `select`

```go
select {
case v := <-a:
    use(v)
case b <- x:
    // 发送成功
case <-time.After(time.Second):
    // 超时
case <-ctx.Done():
    return ctx.Err()
default:
    // 没有任何 case 就绪时立即执行
}
```

- 多个 case 同时就绪时**随机选一个**（避免饥饿）；
- `default` 让 select 变成非阻塞轮询，**不要放在 for 循环里空转**（会吃满 CPU）；
- `time.After` 每次都会创建定时器，热路径用 `time.NewTimer` + `Stop`。

```go
// 正确的超时模式
timer := time.NewTimer(d)
defer timer.Stop()
select {
case v := <-ch:
    use(v)
case <-timer.C:
}
```

### 7.4 四种基础并发模式

```go
// 1) worker pool：固定并发度消费任务
jobs := make(chan Job)
var wg sync.WaitGroup
for i := 0; i < 8; i++ {
    wg.Add(1)
    go func() {
        defer wg.Done()
        for j := range jobs { process(j) }
    }()
}
for _, j := range all { jobs <- j }
close(jobs)
wg.Wait()

// 2) pipeline：阶段间用 channel 连接
func stage(in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for v := range in { out <- v * 2 }
    }()
    return out
}

// 3) fan-out/fan-in：多个消费者 + 合并
func merge(cs ...<-chan int) <-chan int {
    var wg sync.WaitGroup
    out := make(chan int)
    for _, c := range cs {
        wg.Add(1)
        go func(c <-chan int) {
            defer wg.Done()
            for v := range c { out <- v }
        }(c)
    }
    go func() { wg.Wait(); close(out) }()
    return out
}

// 4) or-done：任一完成即返回
func first(ctx context.Context, a, b <-chan int) int {
    select {
    case v := <-a: return v
    case v := <-b: return v
    case <-ctx.Done(): return 0
    }
}
```

### 7.5 `context`：取消与传值

```go
ctx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
defer cancel()                       // 必须调用，否则泄漏定时器

req = req.WithContext(ctx)
select {
case <-ctx.Done():
    return ctx.Err()                 // context.DeadlineExceeded / Canceled
case res := <-resultCh:
    return res
}
```

| 函数 | 用途 |
|---|---|
| `WithCancel` | 手动取消 |
| `WithTimeout` / `WithDeadline` | 超时/截止时间 |
| `WithValue` | 传请求域数据（trace id、用户身份） |
| `WithoutCancel`（1.21+） | 脱离父取消，但保留值 |
| `WithCancelCause`（1.20+） | 带原因取消，`context.Cause(ctx)` 取回 |

🧭 规则：`context.Context` 是**第一个参数**；不要存进结构体；不要用它传可选参数；`WithValue` 的 key 用自定义类型避免冲突。

⚠️ 忘记 `cancel()` 会让父 context 持有子节点直到父被取消——长生命周期服务里这是典型泄漏。

### 7.6 泄漏与死锁的四种形态

| 形态 | 表现 | 修法 |
|---|---|---|
| 发送方无人接收 | goroutine 卡在 `ch <-` | 用 `select` + `ctx.Done()`，或保证接收方存在 |
| 接收方无人发送 | goroutine 卡在 `<-ch` | 关闭 channel 或使用超时 |
| 忘记 cancel | context 树增长、定时器不释放 | `defer cancel()` |
| 锁顺序相反 | 两个 goroutine 互等 | 全局锁顺序、`TryLock`、减少锁嵌套 |

Go 1.27 提供 **goroutine leak profile**，能自动识别"永远不可能被唤醒"的 goroutine（见 26 章）：

```go
pprof.Lookup("goroutineleak").WriteTo(os.Stdout, 1)
```

---

## 8. 同步原语与内存模型

### 8.1 `Mutex` 与 `RWMutex`

```go
type Counter struct {
    mu sync.Mutex
    n  int
}

func (c *Counter) Inc() {
    c.mu.Lock()
    defer c.mu.Unlock()     // 临界区短时，defer 开销可接受
    c.n++
}
```

- `sync.Mutex` **不可重入**：同一 goroutine 重复 `Lock` 会死锁；
- 零值可用，不需要初始化；**不要复制**已使用的 Mutex（包含它的结构体也不要值传递）；
- Go 的 Mutex 有**正常模式与饥饿模式**：等待超过 1ms 会切换到饥饿模式，把锁直接交给队首，避免长尾。

```go
var rw sync.RWMutex
rw.RLock()   // 读锁：可多个并发
rw.Lock()    // 写锁：独占
```

⚠️ `RWMutex` 只适合**读远多于写**且临界区不短的情况；读多但临界区极短时，普通 `Mutex` 往往更快（RWMutex 的原子操作更多）。

### 8.2 `sync.Once`

```go
var (
    once sync.Once
    cfg  *Config
)

func Get() *Config {
    once.Do(func() { cfg = load() })
    return cfg
}
```

1.21+ 提供更简洁的包装：

```go
var Get = sync.OnceValue(func() *Config { return load() })
var Must = sync.OnceValue(func() error { return init() })
var Pair = sync.OnceValues(func() (int, error) { return 1, nil })
```

`OnceValue` 会缓存**第一次的返回值**（包括 error），后续调用直接返回同一结果——适合"初始化一次，失败也缓存"的场景。

### 8.3 `WaitGroup`

```go
var wg sync.WaitGroup
for _, x := range xs {
    wg.Add(1)
    go func() {
        defer wg.Done()
        work(x)
    }()
}
wg.Wait()
```

1.25+ 可以更紧凑：

```go
var wg sync.WaitGroup
for _, x := range xs {
    wg.Go(func() { work(x) })     // 自动 Add(1) + Done
}
wg.Wait()
```

⚠️ `Add` 必须在 `Wait` 之前调用；在 goroutine 内 `Add` 可能与 `Wait` 竞争，导致提前返回。

### 8.4 原子操作与内存模型

```go
type Stats struct {
    hits atomic.Int64
    errs atomic.Int64
}

s.hits.Add(1)
n := s.hits.Load()
```

Go 的 `sync/atomic` 提供 `Add/Load/Store/Swap/CompareAndSwap`，以及 1.19+ 的类型化版本（`atomic.Int64`、`atomic.Pointer[T]` 等）。

**Go 内存模型的关键保证**：

| 操作对 | happens-before 关系 |
|---|---|
| channel 发送 → 对应接收完成 | 发送前的写入对接收方可见 |
| 无缓冲 channel 接收 → 发送完成 | 接收前的写入对发送方可见 |
| `close(ch)` → 收到零值 | 关闭前的写入可见 |
| `Unlock` → 下一次 `Lock` | 临界区写入可见 |
| `Once.Do` 返回 → 其他 `Do` 返回 | 初始化写入可见 |
| `WaitGroup.Done` → `Wait` 返回 | 完成前写入可见 |
| 原子操作之间 | 顺序一致性（Go 不提供 relaxed 语义） |

⚠️ **数据竞争 = 未定义行为**，不只是"可能读到旧值"。race detector 能发现大部分，但没发现不等于没有。

### 8.5 `sync.Pool`

```go
var bufPool = sync.Pool{
    New: func() any { return new(bytes.Buffer) },
}

b := bufPool.Get().(*bytes.Buffer)
b.Reset()
defer bufPool.Put(b)
```

- 池中对象**可能在任意 GC 时被清空**，不能当缓存用（不能假设 Put 后一定还在）；
- 适合"临时缓冲、高频分配、生命周期短"的对象；
- 取出后必须 `Reset`，否则残留上一次的数据；
- Go 1.13 起每个 P 有私有/共享队列，减少锁竞争。

⚠️ 把 `sync.Pool` 当对象池复用"有状态连接/大对象"会踩坑：GC 后池空了，代码必须能在 `Get` 返回新对象时正确工作。

### 8.6 `sync.Map`、`sync.Cond`

```go
var m sync.Map
m.Store("k", 1)
v, ok := m.Load("k")
m.Range(func(k, v any) bool { return true })
```

`sync.Map` 适合两种场景：**键集合基本固定、读多写少**；或**多个 goroutine 读写不相交的键**。其他场景普通 `map` + `Mutex` 更快、类型更安全。

```go
var mu sync.Mutex
cond := sync.NewCond(&mu)

// 等待方
mu.Lock()
for !ready { cond.Wait() }
mu.Unlock()

// 通知方
mu.Lock()
ready = true
cond.Broadcast()
mu.Unlock()
```

`Cond.Wait` 会原子地释放锁并挂起，被唤醒后重新加锁；**必须在循环里检查条件**（虚假唤醒）。

### 8.7 竞态检测

```bash
go test -race ./...
go run -race .
go build -race .
```

- race detector 基于 happens-before 的动态分析，运行时开销约 5–20 倍、内存 5–10 倍；
- 只能发现**实际执行到**的竞争，测试覆盖率决定效果；
- CI 里至少跑一次 `-race`，尤其是并发相关的包；
- 检测到竞争会打印两个访问点的完整栈，按栈定位共享变量。

---

## 9. 错误处理

### 9.1 `error` 接口与哨兵错误

```go
type error interface{ Error() string }
```

错误是**普通值**，不是控制流异常。标准库用"哨兵错误"表达可判定的条件：

```go
if err == io.EOF { }              // 精确相等
if errors.Is(err, io.EOF) { }     // 推荐：能穿透包装
```

```go
var ErrNotFound = errors.New("not found")

func Find(id int) (*User, error) {
    if id <= 0 { return nil, ErrNotFound }
    return &User{ID: id}, nil
}
```

- 哨兵错误是包级变量，命名 `ErrXxx`，文档里要写清"调用方该如何处理"；
- 不要用字符串比较 `err.Error() == "not found"`——文案会变，包装后也不相等。

### 9.2 包装、解包与合并

```go
// 包装：保留原始错误，附加上下文
if err != nil {
    return fmt.Errorf("load config %q: %w", path, err)
}

// 判定：沿链查找
if errors.Is(err, fs.ErrNotExist) { }

// 取类型
var pe *fs.PathError
if errors.As(err, &pe) { fmt.Println(pe.Path) }

// 合并多个错误（1.20+）
err := errors.Join(validateName(), validateAge())
```

`%w` 与 `%v` 的区别：

| 动词 | 是否可 `Is/As` | 用途 |
|---|---|---|
| `%w` | ✅ 保留错误链 | 需要调用方判定时 |
| `%v` | ❌ 只留文本 | 确定不再解包时（日志、对外文案） |

⚠️ 一个格式化串里可以有多个 `%w`（1.20+ 支持），但会让错误链分叉，`Is/As` 仍能遍历；不要滥用。

⚠️ **不要包装 `nil`**：`fmt.Errorf("x: %w", nil)` 返回非 nil 错误（Go 1.20 起会返回一个包含 "nil" 的错误），容易制造"看似有错"的 bug。

### 9.3 自定义错误类型

```go
type ValidationError struct {
    Field string
    Value any
}

func (e *ValidationError) Error() string {
    return fmt.Sprintf("invalid %s: %v", e.Field, e.Value)
}

// 可选：实现 Is 支持语义比较
func (e *ValidationError) Is(target error) bool {
    t, ok := target.(*ValidationError)
    return ok && t.Field == e.Field
}
```

选择标准：

| 需求 | 做法 |
|---|---|
| 调用方只需判断"是不是这类错" | 哨兵错误 + `errors.Is` |
| 需要携带字段（路径、ID、状态码） | 结构体错误类型 + `errors.As` |
| 只需要上下文文本 | `fmt.Errorf("%w")` 包装 |

🧭 错误类型**放在产生它的包里**，命名 `XxxError`；不要导出内部字段，用方法暴露。

### 9.4 错误处理惯用法

```go
// 1) 尽早返回，减少嵌套
if err != nil { return err }

// 2) 只处理一次：要么返回，要么记日志并吞掉，不要两者都做
if err != nil {
    return fmt.Errorf("step: %w", err)   // 交给上层记日志
}

// 3) 给错误加"动作上下文"，但不重复底层信息
return fmt.Errorf("connect to %s: %w", addr, err)

// 4) 不要用 error 表示"正常分支"
// if err == io.EOF { return nil } 是合理的；但别把"没找到"设计成 error 再到处判
```

| 反模式 | 问题 |
|---|---|
| `if err != nil { log.Println(err) }` 后继续执行 | 状态可能已损坏 |
| 每层都 `log` 再返回 | 日志重复，噪音淹没根因 |
| `panic(err)` | 库不应让调用方面对崩溃 |
| `errors.New(err.Error())` | 丢失错误链 |
| 用 `err != nil` 判断业务状态 | 违背"错误是异常情况"的语义 |

### 9.5 `panic` / `recover` 的边界

```go
func safeCall(f func()) (err error) {
    defer func() {
        if r := recover(); r != nil {
            err = fmt.Errorf("panic: %v", r)
        }
    }()
    f()
    return nil
}
```

规则：

1. **库代码不要 panic**（除非是"程序员错误"，如索引越界、非法参数）；
2. `recover` 只在 `defer` 中有效，且只能捕获**当前 goroutine** 的 panic；
3. **子 goroutine panic 会终止整个进程**，不会传到父 goroutine 的 `recover`：

```go
go func() {
    defer func() {
        if r := recover(); r != nil { log.Println("recovered:", r) }
    }()
    risky()
}()
```

4. `panic` 的 `recover` 值可能是 `error`，也可能是任意类型（`panic("x")`），断言要小心。

🧭 进程边界（HTTP handler、消息消费循环、任务调度器）用 `recover` 做"单个请求失败不影响整体"，并记录完整栈（`debug.Stack()`）。

### 9.6 错误边界与对外映射

```go
type APIError struct {
    Code    string `json:"code"`
    Message string `json:"message"`
}

func (s *Server) handle(w http.ResponseWriter, r *http.Request) {
    if err := s.do(r); err != nil {
        s.log.Error("request failed", "err", err, "path", r.URL.Path)
        code, msg := mapError(err)          // 内部错误 → 稳定对外码
        writeJSON(w, code, APIError{Code: msg, Message: msg})
    }
}
```

原则：**内部错误可以很细（带栈、SQL、路径），对外错误必须稳定且不泄漏实现细节**。HTTP 层统一把 `error` 映射成状态码 + 业务码，避免 handler 里到处 `http.Error`。

---

## 10. 反射与 `unsafe`

### 10.1 `reflect` 基础

```go
v := reflect.ValueOf(user)        // 传入接口，得到动态值
t := v.Type()                     // 具体类型

fmt.Println(t.Name(), t.Kind())   // User struct
for i := 0; i < t.NumField(); i++ {
    f := t.Field(i)
    fmt.Println(f.Name, f.Type, f.Tag.Get("json"))
}
```

| 操作 | 方法 |
|---|---|
| 取类型 | `reflect.TypeOf(x)` |
| 取种类 | `t.Kind()`（`Struct`/`Ptr`/`Slice`…） |
| 解指针 | `v.Elem()` |
| 解接口 | `v.Elem()`（对 `interface{}` 值） |
| 取字段 | `v.Field(i)` / `v.FieldByName("X")` |
| 调方法 | `v.MethodByName("M").Call(args)` |
| 新建 | `reflect.New(t)`（返回 `*T` 的 Value） |
| 转换 | `v.Convert(t)`、`v.Interface()` |

⚠️ `Kind` 是底层种类，`Type` 是静态类型；`type MyInt int` 的 `Kind` 是 `Int`，但 `Type` 是 `MyInt`。

### 10.2 可设置性（`CanSet`）

```go
x := 1
v := reflect.ValueOf(x)
// v.SetInt(2)                 // ❌ panic：不可设置

p := reflect.ValueOf(&x)
p.Elem().SetInt(2)             // ✅ 通过指针的 Elem 可设置
fmt.Println(x)                 // 2
```

规则：**只有通过指针得到的、可寻址的 Value 才能修改**。`reflect.ValueOf(x)` 是副本，永远不可设置。

### 10.3 结构体标签

```go
type User struct {
    ID    int    `json:"id" db:"user_id"`
    Name  string `json:"name,omitempty" validate:"required,min=2"`
    Email string `json:"-"`        // 忽略
}

tag := reflect.TypeOf(User{}).Field(1).Tag
fmt.Println(tag.Get("json"))       // name,omitempty
```

解析带逗号的标签：

```go
name, opts, _ := strings.Cut(tag.Get("json"), ",")
for _, o := range strings.Split(opts, ",") { /* ... */ }
```

⚠️ 标签是**运行期字符串**，拼错不会编译报错；用 `go vet` 的 `structtag` 检查。

### 10.4 性能与替代方案

| 方案 | 性能 | 灵活性 | 适用 |
|---|---|---|---|
| 反射 | 慢（10–100 倍） | 最高 | 序列化框架、ORM、通用工具 |
| 代码生成 | 接近手写 | 中（需构建步骤） | protobuf、stringer、mock |
| 泛型 | 接近手写 | 中（需静态类型） | 容器/算法 |
| 类型断言 | 快 | 低 | 已知少量类型 |

🧭 热路径不要用反射。需要"按类型生成代码"时优先 `go:generate`：

```go
//go:generate stringer -type=Status
type Status int
```

`go generate ./...` 会执行注释里的命令，生成 `status_string.go` 并提交到仓库（CI 里校验"重新生成后无 diff"）。

### 10.5 `unsafe` 的四条规则

```go
import "unsafe"

unsafe.Sizeof(x)          // 类型大小（编译期常量）
unsafe.Alignof(x)         // 对齐
unsafe.Offsetof(s.Field)  // 字段偏移
unsafe.Pointer(&x)        // 任意指针 ↔ unsafe.Pointer
```

合法的转换模式（Go 规范明确列出）：

1. `*T1` ↔ `unsafe.Pointer` ↔ `*T2`（重新解释内存）；
2. `uintptr` ↔ `unsafe.Pointer`（仅用于系统调用/反射）；
3. `unsafe.Pointer` ↔ `uintptr` 后**立即**转回指针（不能存下来跨语句）；
4. `unsafe.String` / `unsafe.Slice`（1.20+）从指针构造字符串/切片。

```go
// 零拷贝：[]byte -> string（源必须保持不变）
b := []byte("hello")
s := unsafe.String(&b[0], len(b))
```

⚠️ 三类 UB：

- 把 `uintptr` 保存到变量后再转回指针（GC 可能已移动/回收对象）；
- 用 `unsafe.Slice` 越过原分配边界；
- 修改 `string` 底层字节（字符串可能被驻留/共享）。

🧭 只有在 profile 证明拷贝是瓶颈、且能用 benchmark 证明收益时才用 `unsafe`；每个用法旁写清"为什么安全"。

---

## 11. 标准库核心

### 11.1 `strings` / `bytes` / `slices` / `maps`

| 包 | 代表 API | 说明 |
|---|---|---|
| `strings` | `Cut`、`CutLast`（1.27）、`Builder`、`Fields`、`TrimPrefix` | 文本处理 |
| `bytes` | `Buffer`、`Cut`、`CutLast`（1.27）、`EqualFold` | 二进制/字节缓冲 |
| `slices` | `Sort`、`SortFunc`、`Contains`、`Index`、`Clone`、`Compact`、`Delete`、`Concat` | 1.21+ |
| `maps` | `Keys`、`Values`、`Clone`、`Equal`、`Copy`、`DeleteFunc` | 1.21+ |
| `cmp` | `Compare`、`Less`、`Or` | 1.21+ |

```go
slices.SortFunc(users, func(a, b User) int {
    return cmp.Or(
        strings.Compare(a.Name, b.Name),
        cmp.Compare(a.ID, b.ID),
    )
})

keys := slices.Sorted(maps.Keys(m))   // 1.23+：直接得到有序切片
```

💡 不要再手写 `sort.Slice` 的 `less` 函数去处理多字段排序，`cmp.Or` 更不易错。

### 11.2 `strconv` 与 `fmt`

```go
n, err := strconv.Atoi("42")            // string -> int
s := strconv.Itoa(42)                   // int -> string
f, err := strconv.ParseFloat("1.5", 64)
b, err := strconv.ParseBool("true")

fmt.Sprintf("%d %s %.2f %v %q", 1, "a", 1.5, m, "quoted")
fmt.Errorf("wrap: %w", err)
fmt.Fprintf(w, "%s\n", s)
```

⚠️ `fmt.Sprintf` 在热路径上很贵（反射 + 分配）；日志/序列化用专用 API（`slog`、`json.Encoder`）。

⚠️ `%v` 打印结构体不含字段名，`%+v` 才有；`%#v` 是 Go 语法表示。生产日志优先结构化字段而不是格式化字符串。

### 11.3 `time`

```go
now := time.Now()                       // 带单调时钟读数
d := 1500 * time.Millisecond
later := now.Add(d)
fmt.Println(later.Format(time.RFC3339))

loc, _ := time.LoadLocation("Asia/Shanghai")
t := time.Date(2026, 10, 29, 12, 0, 0, 0, loc)
```

| 概念 | 说明 |
|---|---|
| `time.Time` | 时刻；内部含单调时钟，用于计算间隔 |
| `time.Duration` | 纳秒整数；`d.Seconds()` 返回 float64 |
| `time.Timer` | 一次性定时器；1.23 起通道为**同步**（无缓冲） |
| `time.Ticker` | 周期触发；必须 `Stop` |
| `LoadLocation` | 依赖系统时区库；容器里可能缺失，用 `time/tzdata` 内嵌 |

```go
ticker := time.NewTicker(time.Second)
defer ticker.Stop()
for {
    select {
    case <-ticker.C:
    case <-ctx.Done():
        return
    }
}
```

⚠️ 不要用 `time.Sleep` 做精确调度；不要用 `t.Sub(now)` 之外的算术跨时区；**存储时间用 UTC，展示时再转本地时区**。

### 11.4 `os` / `io` / `fs`

```go
f, err := os.Open(path)          // 只读
defer f.Close()

data, err := os.ReadFile(path)   // 一次性读小文件
err = os.WriteFile(path, data, 0o644)

// 流式复制，自动处理大文件
_, err = io.Copy(dst, src)

// 判断错误
if errors.Is(err, fs.ErrNotExist) { }
```

关键接口只有两个：

```go
type Reader interface{ Read(p []byte) (n int, err error) }
type Writer interface{ Write(p []byte) (n int, err error) }
```

- `io.Reader` 返回 `n>0` 时**必须先处理这 n 字节**，再看 `err`（可能是 `io.EOF`）；
- 包装用 `bufio.NewReader/Writer`，减少系统调用；
- `io.ReadAll` 会无限增长内存，处理不可信输入时用 `io.LimitReader`；
- 文件路径拼接用 `filepath.Join`，不要手写 `/`。

⚠️ `defer f.Close()` 的错误通常被忽略；写文件时要显式检查 `Close`（可能刷新缓冲失败）。

### 11.5 `net/http`

**客户端**（超时是必须的）：

```go
client := &http.Client{
    Timeout: 10 * time.Second,
    Transport: &http.Transport{
        MaxIdleConns:        100,
        MaxIdleConnsPerHost: 10,
        IdleConnTimeout:     90 * time.Second,
    },
}

req, _ := http.NewRequestWithContext(ctx, http.MethodGet, url, nil)
resp, err := client.Do(req)
if err != nil { return err }
defer resp.Body.Close()
body, err := io.ReadAll(io.LimitReader(resp.Body, 1<<20))
```

- **零值 `http.Client` 没有超时**，会无限等待；
- 必须关闭 `resp.Body`，否则连接无法复用；
- 复用 `Client` 与 `Transport`，不要每次请求新建；
- 4xx/5xx **不会**返回 error，要自己判断 `StatusCode`。

**服务端**（1.22+ 路由）：

```go
mux := http.NewServeMux()
mux.HandleFunc("GET /users/{id}", getUser)      // 方法 + 路径参数
mux.HandleFunc("POST /users", createUser)

func getUser(w http.ResponseWriter, r *http.Request) {
    id := r.PathValue("id")
    // ...
}

srv := &http.Server{
    Addr:              ":8080",
    Handler:           mux,
    ReadHeaderTimeout: 5 * time.Second,
    ReadTimeout:       30 * time.Second,
    WriteTimeout:      30 * time.Second,
    IdleTimeout:       120 * time.Second,
}
srv.ListenAndServe()
```

1.27 的变化（与本机写法相关）：

- HTTP/1 的 `Response.Body` 在 `Close` 时会自动 drain 未读内容（有限额度），提升连接复用；
- HTTP/2 服务端支持 RFC 9218 优先级，可用 `Server.DisableClientPriority` 关闭；
- 新增 `Server.MaxHeaderValueCount` 限制头部数量。

⚠️ 默认 `http.Server` 没有超时，慢连接会耗尽 goroutine；至少设置 `ReadHeaderTimeout`。

### 11.6 `encoding/json`

```go
type User struct {
    ID    int64  `json:"id"`
    Name  string `json:"name"`
    Email string `json:"email,omitempty"`
    Age   int    `json:"-"`
}

data, err := json.Marshal(u)
err = json.Unmarshal(data, &u)
```

v1 的常见陷阱：

| 陷阱 | 说明 |
|---|---|
| 大小写不敏感匹配 | `"Name"` 也能匹配 `name` 标签 |
| 数字精度 | 默认解到 `float64`，大整数丢精度；用 `json.Decoder.UseNumber()` |
| 未知字段 | 默认忽略；需要严格时 `DisallowUnknownFields` |
| `omitempty` | 对 `0`、`false`、空串、nil 都生效；需要区分"零值"用 `omitzero`（1.24+）或指针 |
| HTML 转义 | `<`、`>`、`&` 被转成 `\u003c`；需要关闭用 `Encoder.SetEscapeHTML(false)` |

Go 1.27 引入 `encoding/json/v2`：

```go
import "encoding/json/v2"

data, err := json.Marshal(u, json.Deterministic(true))
err = json.Unmarshal(data, &u, json.RejectUnknownMembers(true))
```

v2 的严格默认值：拒绝非法 UTF-8、拒绝重复键、大小写敏感匹配；v1 包已由 v2 实现，行为保持兼容（错误文案可能变化），可用 `GOEXPERIMENT=nojsonv2` 回退。

🧭 新项目可以直接用 v2；迁移期先读 v1 文档的 "Migrating to v2" 一节，逐项确认行为差异。

### 11.7 `log/slog`

```go
logger := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
    Level: slog.LevelInfo,
}))
slog.SetDefault(logger)

logger.Info("request done",
    "method", r.Method,
    "path", r.URL.Path,
    "status", code,
    "dur_ms", time.Since(start).Milliseconds(),
)

logger.Error("db failed", "err", err)
```

- 键值对优于格式化字符串，便于索引与过滤；
- `slog.With("request_id", id)` 派生带上下文的 logger；
- 敏感字段（token、密码）不要进日志，必要时实现 `slog.LogValuer` 做脱敏；
- `slog` 是结构化日志的**门面**，底层 handler 可替换（JSON、文本、OTel）。

### 11.8 `crypto` 的安全默认值

| 用途 | 推荐 | 不要用 |
|---|---|---|
| 对称加密 | `crypto/aes` + GCM | ECB、手写填充 |
| 哈希 | `crypto/sha256`、`sha3` | MD5、SHA1 |
| 密码存储 | `golang.org/x/crypto/argon2` / `bcrypt` | 明文、单次 SHA256 |
| 随机数 | `crypto/rand` | `math/rand` |
| 密钥交换 | `crypto/ecdh`、`crypto/mlkem`（1.24+） | 自实现 DH |
| 签名 | `crypto/ecdsa`、`crypto/ed25519`、`crypto/mldsa`（1.27+） | RSA-PKCS1v1.5（新系统） |

```go
key := make([]byte, 32)
if _, err := rand.Read(key); err != nil { return err }
```

⚠️ `math/rand` 的默认源在 1.20 起自动播种，但它**不是密码学安全的**；令牌、盐、会话 ID 一律用 `crypto/rand`。

### 11.9 `iter` 与 range-over-func

1.23 起，函数可以返回 `iter.Seq[T]`，直接用于 `range`：

```go
func Backward[T any](xs []T) iter.Seq[T] {
    return func(yield func(T) bool) {
        for i := len(xs) - 1; i >= 0; i-- {
            if !yield(xs[i]) { return }   // 调用方 break 时提前结束
        }
    }
}

for v := range Backward([]int{1, 2, 3}) {
    fmt.Println(v)                        // 3 2 1
}
```

要点：

- `yield` 返回 `false` 表示调用方 `break`/提前退出，生成方**必须**停止并释放资源；
- 需要错误时用 `iter.Seq2[T, error]`；
- 标准库已有 `slices.Values`、`slices.All`、`maps.Keys`、`maps.Values` 等适配器。

⚠️ range-over-func 的循环体是回调，**不能在里面 `return` 外层函数**（只能 `break`）；`defer` 的执行时机也变成"整个 range 结束"。

---

# 第二部分　工程实践

## 12. 模块与依赖管理

### 12.1 module 与 package

| 概念 | 边界 | 标识 |
|---|---|---|
| module | 版本、依赖、发布单位 | `module example.com/app` |
| package | 编译、可见性单位 | 目录内 `package foo` |
| import path | 包的唯一标识 | `example.com/app/internal/store` |

一个 module 可以包含多个 package；一个 package 不能被多个 module 共享（除 `vendor` 复制）。

### 12.2 `go.mod` 与 `go.sum`

```go
module example.com/app

go 1.27

require (
    github.com/go-chi/chi/v5 v5.2.1
    golang.org/x/sync v0.10.0
)

require golang.org/x/text v0.21.0 // indirect
```

| 指令 | 含义 |
|---|---|
| `module` | 模块路径，也是 import 前缀 |
| `go` | 语言语义版本（影响 loop var、`stdversion` 等） |
| `toolchain` | 建议工具链版本 |
| `require` | 直接/间接依赖及最低版本 |
| `replace` | 替换来源（本地路径、fork） |
| `exclude` | 排除某个版本（慎用） |
| `retract` | 声明自己的某个版本有问题，提示下游升级 |
| `tool`（1.24+） | 声明开发工具依赖 |

`go.sum` 记录每个模块内容的哈希，**必须提交**；它防篡改，不参与版本选择。

### 12.3 版本选择：最小版本选择（MVS）

Go 不装"最新版"，而是对每个模块取**满足所有约束的最小版本**：

```text
A requires C v1.2.0
B requires C v1.4.0
→ 最终选 C v1.4.0（最大需求中的最小满足）
```

好处是构建可复现、升级显式；代价是"依赖的依赖"可能引入旧版本的安全问题，需要定期 `go list -m -u all` 检查。

```bash
go list -m -u all          # 列出可升级的模块
go get example.com/lib@latest
go get example.com/lib@v1.4.2
go mod tidy                # 增删依赖，使 go.mod 与代码一致
```

⚠️ `go get -u ./...` 会升级整个依赖树，容易引入不兼容；生产项目应逐个升级并跑测试。

### 12.4 `replace`、`exclude`、`retract`

```go
// 本地联调：指向工作区外的目录（不要提交到主干）
replace example.com/lib => ../lib

// 临时用 fork 修 bug
replace example.com/lib => github.com/me/lib v1.4.3-fix

// 声明自己的坏版本
retract v1.2.0 // 有严重 bug，请升级到 v1.2.1
```

- `replace` 只影响**当前 module 的构建**，不会传递给下游；
- 提交 `replace` 指向本地路径会让 CI 失败，合并前必须删除；
- `replace` 到 fork 时版本号要用伪版本或 tag，否则 `go mod tidy` 会报错。

### 12.5 vendor、私有模块与代理

```bash
go mod vendor          # 复制依赖到 vendor/，构建时优先使用
go mod vendor -o dir
```

| 环境变量 | 作用 |
|---|---|
| `GOPROXY` | 模块代理，默认 `https://proxy.golang.org,direct` |
| `GOPRIVATE` | 匹配的模块跳过代理与校验（公司内网仓库） |
| `GONOSUMDB` / `GONOSUMCHECK` | 跳过校验和数据库 |
| `GOFLAGS=-mod=vendor` | 强制使用 vendor |
| `GONOSUMDB` | 与 `GOPRIVATE` 类似，按前缀匹配 |

```bash
# 公司内网：私有仓库 + 公共代理
go env -w GOPRIVATE=git.company.com/*
go env -w GOPROXY=https://proxy.golang.org,https://goproxy.company.com,direct
```

🧭 是否 vendor：需要"离线构建/审计依赖内容/构建环境无网络"时 vendor；否则用 `go.sum` + 代理即可，vendor 会让 diff 变大、升级变笨重。

### 12.6 依赖治理

```bash
go mod graph | wc -l              # 依赖边数量
go mod why -m github.com/x/y      # 谁引入了它
go list -m -json all | jq '.Path' # 机器可读清单
govulncheck ./...                 # 官方漏洞扫描（基于调用图，误报少）
go mod tidy -diff                 # 1.23+：只显示差异，不写文件
```

CI 建议：

1. `go mod tidy` 后检查 `git diff --exit-code`（防止漏提交）；
2. `govulncheck ./...` 作为独立 job；
3. 定期（每周）跑 `go list -m -u all` 生成升级 PR；
4. 限制新依赖：每个新依赖要回答"标准库能否替代、维护是否活跃、license 是否合规"。

### 12.7 `tool` 指令（1.24+）

以前工具依赖会污染 `go.mod` 的 `require`，或者靠 `tools.go` 空导入。1.24 起：

```bash
go get -tool golang.org/x/tools/cmd/stringer
go tool stringer -type=Status
```

```go
tool golang.org/x/tools/cmd/stringer
```

好处：工具版本被锁定、`go tool` 自动使用对应版本、不参与主程序的依赖图。

---

## 13. 项目结构与分层

### 13.1 三种常见布局

```text
# 1) 单 main 包（小工具）
app/
├── go.mod
├── main.go
└── main_test.go

# 2) cmd + internal（服务/多命令，最常见）
app/
├── go.mod
├── cmd/
│   ├── api/main.go
│   └── worker/main.go
├── internal/
│   ├── config/
│   ├── store/
│   ├── service/
│   └── transport/http/
├── migrations/
└── Makefile

# 3) 库（可被外部导入）
lib/
├── go.mod
├── lib.go            # 包文档 + 对外 API
├── internal/         # 实现细节
└── example_test.go
```

⚠️ 不要照搬 `pkg/` 目录：那是历史习惯，`internal/` 才是编译器强制的边界。没有外部消费者的项目用 `internal/` 就够。

### 13.2 `internal` 的边界

```text
example.com/app/internal/store     # 只能被 example.com/app/... 导入
example.com/app/pkg/store          # 任何人都能导入
```

- `internal` 的父目录决定可见范围：`a/internal/x` 只能被 `a` 及其子目录导入；
- 把"不想承诺兼容"的实现放 `internal`，是 Go 里最便宜的 API 治理手段；
- 测试可以放在 `internal` 内部（同包）或 `internal_test`（外部包），后者验证公开面。

### 13.3 包命名

| 规则 | 示例 |
|---|---|
| 小写、短、无下划线 | `http`, `user`, `postgres` |
| 不用复数 | `user` 而不是 `users` |
| 不重复路径信息 | `store.User` 而不是 `store.StoreUser` |
| 避免 `util`/`common`/`helpers` | 按职责命名：`retry`, `clock`, `ids` |
| 包名与目录名一致 | `internal/config` → `package config` |

💡 包的文档注释写在 `package foo` 上方，第一句以包名开头：`// Package config loads...`。

### 13.4 依赖方向与接口归属

```text
cmd/api → transport/http → service → store(interface)
                                      ↑
                              postgres(实现)
```

规则：

1. **依赖指向抽象**：`service` 定义它需要的 `store.UserStore` 接口，`postgres` 实现它；
2. **接口定义在使用方**，实现方不感知；
3. **禁止循环依赖**：Go 编译器直接拒绝，出现循环说明分层错了；
4. 跨层调用不要"跳级"（handler 直接调 DB），否则业务规则会散落。

```go
// service/service.go
type UserStore interface {
    Get(ctx context.Context, id int64) (*User, error)
}

type Service struct{ store UserStore }
```

### 13.5 `main` 包的职责

`main` 只做四件事：解析配置、构造依赖、启动、等待退出。业务逻辑不应出现在 `main.go`。

```go
func main() {
    if err := run(); err != nil {
        fmt.Fprintln(os.Stderr, "fatal:", err)
        os.Exit(1)
    }
}

func run() error {
    cfg, err := config.Load()
    if err != nil { return err }
    app, err := NewApp(cfg)
    if err != nil { return err }
    return app.Run(context.Background())
}
```

把 `run() error` 抽出来，测试与覆盖率都更容易（`main` 本身不可测）。

### 13.6 常见反模式

| 反模式 | 后果 | 改法 |
|---|---|---|
| `util` 包塞一切 | 无边界、循环依赖 | 按职责拆小包 |
| 全局可变状态（`var db *sql.DB`） | 测试互相污染、并发隐患 | 依赖注入 |
| `init()` 里做 IO/连数据库 | 启动顺序不可控、测试难 | 显式 `New`/`Run` |
| 包之间互相 import | 编译失败 | 提取接口到第三方包 |
| 一个包 5000 行 | 评审困难、冲突多 | 按领域拆文件/子包 |

---

## 14. 测试体系

### 14.1 表驱动测试

```go
func TestParse(t *testing.T) {
    tests := []struct{
        name string
        in   string
        want int
        wantErr bool
    }{
        {"positive", "42", 42, false},
        {"negative", "-1", -1, false},
        {"empty", "", 0, true},
        {"garbage", "x", 0, true},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            got, err := Parse(tt.in)
            if (err != nil) != tt.wantErr {
                t.Fatalf("err = %v, wantErr %v", err, tt.wantErr)
            }
            if got != tt.want {
                t.Errorf("got %d, want %d", got, tt.want)
            }
        })
    }
}
```

表驱动是 Go 测试的默认形态：**新增用例只加一行**，失败信息自带用例名。

### 14.2 `t.Helper` / `t.Cleanup` / `t.TempDir`

```go
func newTestServer(t *testing.T) *Server {
    t.Helper()                       // 失败时报告调用方行号，而不是这里
    dir := t.TempDir()               // 自动清理
    s, err := New(Config{Dir: dir})
    if err != nil { t.Fatalf("new: %v", err) }
    t.Cleanup(func() { s.Close() })  // 测试结束（含失败）时执行
    return s
}
```

- `Helper` 让断言行指向真正的测试代码；
- `Cleanup` 比 `defer` 更适合"测试辅助函数里注册清理"；
- `TempDir` 自动 `RemoveAll`，并行测试也不会冲突。

### 14.3 子测试与并行

```go
func TestAPI(t *testing.T) {
    for _, tc := range cases {
        t.Run(tc.name, func(t *testing.T) {
            t.Parallel()             // 与同层级其他子测试并行
            // ...
        })
    }
}
```

⚠️ `t.Parallel` 会暂停当前测试直到父测试返回；循环变量在 1.22+ 已按次迭代，不需要 `tc := tc`。

⚠️ 并行测试共享全局状态（环境变量、单例、端口）会互相干扰；每个测试用独立实例或 `t.Setenv`（会自动恢复，但不能与 `t.Parallel` 同用）。

### 14.4 模糊测试

```go
func FuzzParse(f *testing.F) {
    f.Add("42")                       // 种子
    f.Fuzz(func(t *testing.T, s string) {
        v, err := Parse(s)
        if err == nil && v < 0 && !strings.HasPrefix(s, "-") {
            t.Fatalf("unexpected negative: %q -> %d", s, v)
        }
    })
}
```

```bash
go test -fuzz=FuzzParse -fuzztime=30s
```

- 模糊测试适合"解析器、协议、编解码"等输入空间大的函数；
- 失败输入会写入 `testdata/fuzz/`，**提交它**作为回归用例；
- 注意：`go test` 默认只跑种子，只有 `-fuzz` 才真正变异。

### 14.5 基准测试

```go
func BenchmarkParse(b *testing.B) {
    b.ReportAllocs()
    for i := 0; i < b.N; i++ {
        _, _ = Parse("42")
    }
}

func BenchmarkParallel(b *testing.B) {
    b.RunParallel(func(pb *testing.PB) {
        for pb.Next() { work() }
    })
}
```

```bash
go test -bench=. -benchmem -count=5
benchstat old.txt new.txt          # 统计显著性，别只看单次数字
```

⚠️ 基准里不要包含被测函数之外的初始化；`b.N` 由框架调整，循环体必须只做被测工作。

### 14.6 示例测试

```go
func ExampleParse() {
    v, _ := Parse("42")
    fmt.Println(v)
    // Output: 42
}
```

带 `// Output:` 的示例会**被执行并比对输出**，既当文档又当测试；`go doc` 1.27 起支持 `-ex` 列出示例。

### 14.7 集成测试与 `httptest`

```go
func TestHandler(t *testing.T) {
    srv := httptest.NewServer(NewRouter())
    defer srv.Close()

    resp, err := http.Get(srv.URL + "/users/1")
    if err != nil { t.Fatal(err) }
    defer resp.Body.Close()
    if resp.StatusCode != 200 { t.Fatalf("status = %d", resp.StatusCode) }
}
```

- `httptest.NewServer` 起真实 HTTP 服务，适合测客户端/中间件；
- `httptest.NewRecorder` 直接调 handler，更快，适合单元级；
- 1.27 新增 `httptest.NewTestServer`，配合 `testing/synctest` 使用内存网络。

数据库等外部依赖：用 `testcontainers-go` 起真实容器，或接口 + 内存实现；**不要用 sqlite 模拟 postgres 的方言差异**。

### 14.8 `testing/synctest`（1.25+）

```go
func TestRetry(t *testing.T) {
    synctest.Test(t, func(t *testing.T) {
        var n int
        go func() {
            for i := 0; i < 3; i++ {
                time.Sleep(time.Second)     // 虚拟时间，瞬间推进
                n++
            }
        }()
        synctest.Wait()                      // 等待所有 goroutine 阻塞
        if n != 3 { t.Fatalf("n = %d", n) }
    })
}
```

它把 `time` 与 channel 放进"气泡"，`Wait` 会推进虚拟时间直到所有 goroutine 静止，因此**不需要真的 sleep**，也不会 flaky。

### 14.9 覆盖率与 flaky

```bash
go test -coverprofile=cover.out ./...
go tool cover -func=cover.out
go tool cover -html=cover.out
```

- 覆盖率是"哪些行被执行过"，不是"逻辑正确"；
- 重点看**错误分支、并发分支、边界条件**是否覆盖；
- flaky 的三大来源：真实时间、端口冲突、共享全局状态 → 分别用 `synctest`、`:0` 端口、`t.Setenv`/独立实例解决；
- CI 里对 flaky 测试设"重试上限 + 报警"，不要无限重试掩盖问题。

---

## 15. 质量闸门与 CI

### 15.1 格式化

```bash
gofmt -l -w .            # 标准格式（制表符缩进、括号风格）
goimports -l -w .        # 额外整理 import 分组（需 golang.org/x/tools）
```

🧭 格式不参与评审：CI 跑 `gofmt -l .`，有输出就失败。1.27 修正了列对齐逻辑，升级后可能出现少量空白 diff，属正常。

### 15.2 `go vet`

```bash
go vet ./...
```

`vet` 是编译器附带的高置信度检查，覆盖：`printf` 格式串与参数不匹配、结构体标签拼写、锁被复制、不可达代码、`lostcancel`（忘记调用 cancel）、`loopclosure`（1.22 前的循环变量）等。

1.27 起 `go test` **默认执行 `stdversion`**：如果代码用了高于 `go.mod` 中 `go` 指令版本的标准库符号，会直接报错。这能防止"本地能跑、CI 老版本编译失败"。

1.27 还新增了 `printf` 对 `%w` 指针包装的检查：

```go
// 报告：*E 实现了 error，用 %w 包装指针会导致 errors.As 判断异常
fmt.Errorf("x: %w", p)
```

### 15.3 `staticcheck` 与 `golangci-lint`

```bash
go install honnef.co/go/tools/cmd/staticcheck@latest
staticcheck ./...

# 或统一入口
golangci-lint run
```

```yaml
# .golangci.yml（节选）
linters:
  enable:
    - errcheck
    - govet
    - staticcheck
    - ineffassign
    - unused
    - bodyclose
    - errorlint
    - exhaustive
    - gocritic
issues:
  exclude-use-default: false
```

| linter | 抓什么 |
|---|---|
| `errcheck` | 未处理的 error 返回值 |
| `bodyclose` | `http.Response.Body` 未关闭 |
| `errorlint` | `%v` 包装 error、用 `==` 比较错误 |
| `exhaustive` | switch 遗漏枚举值 |
| `gocritic` | 惯用法与性能问题 |
| `nilerr` | 拿到 err 却返回 nil |

⚠️ 不要一次性全开会淹没在历史问题里；按"新代码零告警 + 老代码逐步修"推进。

### 15.4 竞态与覆盖率

```bash
go test -race -coverprofile=cover.out -covermode=atomic ./...
```

- `-race` 必须配 `-covermode=atomic`（普通计数模式在并发下不准）；
- 大仓库可以只对并发相关包开 `-race`，其余用普通模式，节省 CI 时间。

### 15.5 最小 CI

```yaml
# .github/workflows/ci.yml
jobs:
  test:
    strategy:
      matrix:
        os: [ubuntu-latest, macos-latest]
        go: ["1.25.x", "1.27.x"]
    runs-on: ${{ matrix.os }}
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version: ${{ matrix.go }}
          cache: true
      - run: gofmt -l . && test -z "$(gofmt -l .)"
      - run: go vet ./...
      - run: go test -race ./...
      - run: go build ./...
```

要点：

1. **格式、vet、test、build 四条独立失败信号**；
2. `setup-go` 的 `cache: true` 复用模块与构建缓存；
3. 最低版本与最新版本各跑一次（最低版本验证 `go` 指令承诺）；
4. `-race` 只在 Linux 上跑，省时间。

### 15.6 版本矩阵与模块校验

```bash
# 确认 go.mod 与代码一致（CI 里防止漏 tidy）
go mod tidy
git diff --exit-code go.mod go.sum

# 确认没有用到高于 go 指令的 API
go vet ./...          # 1.27 起 test 也会检查
```

🧭 升级 Go 版本的顺序：先升 CI 的 `go` 指令 → 修 `stdversion` 报错 → 跑 `-race` 与基准 → 最后更新 `go.mod` 的 `go` 指令（这一步才会启用新语义）。

### 15.7 本地一键闸门

```makefile
.PHONY: check
check:
	gofmt -l .
	go vet ./...
	staticcheck ./...
	go test -race -cover ./...
```

CI 与本地跑同一条命令，才不会出现"本地过、CI 挂"。

---

## 16. 可观测性

### 16.1 结构化日志

```go
logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))
logger = logger.With("service", "api", "version", build.Version)

logger.Info("request",
    "method", r.Method,
    "path", r.URL.Path,
    "status", code,
    "duration_ms", time.Since(start).Milliseconds(),
)
```

| 级别 | 用途 |
|---|---|
| `Debug` | 排查细节，生产默认关闭 |
| `Info` | 关键业务节点（启动、请求完成） |
| `Warn` | 可自愈但值得关注（重试、降级） |
| `Error` | 需要人介入 |

🧭 每条日志问三个问题：**谁**（service/instance）、**发生了什么**（event）、**影响什么**（request_id/user_id）。字段名保持稳定，否则查询与告警都会失效。

### 16.2 链路追踪

OpenTelemetry 是事实标准：

```go
ctx, span := tracer.Start(ctx, "handleRequest")
defer span.End()

span.SetAttributes(attribute.String("user.id", id))
if err != nil {
    span.RecordError(err)
    span.SetStatus(codes.Error, err.Error())
}
```

要点：

- `ctx` 必须贯穿所有 IO 调用，否则 span 断链；
- 在进程边界（HTTP header、消息属性）注入/提取 `traceparent`；
- 采样率按环境配置：开发 100%，生产 1%–10% + 错误全采。

### 16.3 指标

```go
var (
    requests = promauto.NewCounterVec(prometheus.CounterOpts{
        Name: "http_requests_total",
        Help: "Total HTTP requests.",
    }, []string{"method", "path", "status"})

    latency = promauto.NewHistogramVec(prometheus.HistogramOpts{
        Name:    "http_request_duration_seconds",
        Buckets: prometheus.DefBuckets,
    }, []string{"method", "path"})
)
```

四条黄金信号：**延迟、流量、错误、饱和度**。指标标签基数（cardinality）要控制——把 `user_id` 当标签会让时间序列爆炸。

### 16.4 `pprof` 端点

```go
import _ "net/http/pprof"

go func() {
    // 只监听内网/本地，绝不暴露到公网
    log.Println(http.ListenAndServe("127.0.0.1:6060", nil))
}()
```

| 端点 | 用途 |
|---|---|
| `/debug/pprof/profile?seconds=30` | CPU profile |
| `/debug/pprof/heap` | 堆内存 |
| `/debug/pprof/goroutine?debug=1` | goroutine 栈 |
| `/debug/pprof/goroutineleak` | 泄漏 goroutine（1.27） |
| `/debug/pprof/trace?seconds=5` | 调度 trace |
| `/debug/pprof/mutex`、`/block` | 锁竞争、阻塞 |

⚠️ pprof 端点会泄漏运行时内部信息，生产环境必须加鉴权或只绑定 loopback。

### 16.5 panic 与优雅退出

```go
func recoverMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        defer func() {
            if rec := recover(); rec != nil {
                slog.Error("panic", "rec", rec, "stack", string(debug.Stack()))
                http.Error(w, "internal error", http.StatusInternalServerError)
            }
        }()
        next.ServeHTTP(w, r)
    })
}
```

优雅退出三步：

```go
ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
defer stop()

go func() { srv.ListenAndServe() }()
<-ctx.Done()

shutdownCtx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
defer cancel()
if err := srv.Shutdown(shutdownCtx); err != nil { /* 强制关闭 */ }
```

顺序：**停止接收新请求 → 等待在飞请求 → 关闭连接池/队列 → 刷日志**。漏掉任何一步都会丢数据。

---

## 17. API 设计与发布

### 17.1 命名与包设计

| 场景 | 约定 | 示例 |
|---|---|---|
| 构造函数 | `New` / `NewXxx` | `http.NewRequest` |
| 带配置 | `NewXxx(opts ...Option)` | `grpc.NewServer` |
| 解析 | `ParseXxx` | `time.Parse` |
| 转换 | `XxxOf` / `ToXxx` | `slices.Clone` |
| 判定 | `IsXxx` / `HasXxx` | `errors.Is` |
| 获取 | `GetXxx` | `sql.DB` 的 `Get` |

包名不要重复调用方会写的前缀：`user.User` 冗余，`user.Account` 更好。

### 17.2 选项模式 vs 配置结构体

```go
// 选项模式：可选参数多、需要默认值、未来可扩展
type Option func(*Server)

func WithTimeout(d time.Duration) Option { return func(s *Server) { s.timeout = d } }
func WithLogger(l *slog.Logger) Option  { return func(s *Server) { s.log = l } }

func New(opts ...Option) *Server {
    s := &Server{timeout: 5 * time.Second, log: slog.Default()}
    for _, o := range opts { o(s) }
    return s
}

// 配置结构体：字段少、需要序列化、需要校验
type Config struct {
    Addr    string        `yaml:"addr"`
    Timeout time.Duration `yaml:"timeout"`
}

func NewFromConfig(c Config) (*Server, error) { /* 校验 + 构造 */ }
```

🧭 判断：**参数 ≤3 个且都必填** → 直接传参；**可选参数多、需要默认值** → 选项模式；**配置来自文件/环境变量** → 配置结构体 + 校验。

### 17.3 接受接口，返回具体类型

```go
// 好：调用方能用到全部方法，也能按需当接口用
func NewStore(path string) (*Store, error)

// 差：调用方被迫只能看到接口
func NewStore(path string) (StoreInterface, error)
```

例外：返回接口能**隐藏实现类型**（如 `database/sql` 返回 `driver.Conn`），但要有明确理由。

### 17.4 零值可用与 nil 语义

```go
var buf bytes.Buffer      // 直接用
var mu sync.Mutex         // 直接用
var m map[string]int      // 读可以，写 panic → 文档要说明
```

- 让零值有意义，调用方就不需要 `New`；
- 如果零值不可用，文档必须写"必须用 New 构造"；
- nil 参数要么有意义（可选），要么直接 panic（程序员错误），不要"半支持"。

### 17.5 兼容性：Go 1 承诺下的破坏性变更

**安全**：新增函数、类型、包级变量；新增结构体字段（前提是不用无键字面量）；放宽参数类型（接口）。

**破坏性**：

| 变更 | 为什么破坏 |
|---|---|
| 给公开接口加方法 | 所有实现者编译失败 |
| 改函数签名/返回值 | 调用方编译失败 |
| 删除/重命名导出标识符 | 同上 |
| 结构体加字段且用户用无键字面量 | `T{1, 2}` 编译失败 |
| 改语义（同样的输入不同输出） | 运行期行为变化 |
| 方法集变化（值/指针接收者互换） | 接口满足关系变化 |

🧭 公开接口**永远不要加方法**；需要扩展时定义新接口（`io.ReadCloser` 就是 `Reader` + `Closer`）。

### 17.6 版本与 v2 路径

```text
example.com/lib          → v0/v1（import 路径不含版本）
example.com/lib/v2       → v2+（路径必须含 /v2）
```

- v1 之前（`v0.x`）可以自由破坏；
- 升 v2 时新建目录 `/v2`，老用户不受影响；
- `go.mod` 里 `module example.com/lib/v2`，tag 为 `v2.0.0`。

### 17.7 文档与示例

```go
// Package retry provides bounded retry with backoff.
//
// Basic usage:
//
//	p := retry.New(retry.WithMax(3))
//	err := p.Do(ctx, func() error { return call() })
package retry
```

- 包注释是 `go doc` 的首屏，必须写"这个包解决什么问题"；
- 导出标识符的注释以名字开头：`// Do runs f until...`；
- 带 `// Output:` 的示例会被测试，不会过时。

---

## 18. 性能工程

### 18.1 先测量

```bash
go test -bench=. -benchmem -cpuprofile=cpu.out -memprofile=mem.out
go tool pprof -http=:0 cpu.out
```

🧭 顺序永远是：**基准 → profile → 定位热点 → 改 → 再基准**。没有数据的优化是把复杂度换成了猜测。

### 18.2 分配与逃逸

```bash
go build -gcflags='-m' ./...        # 逃逸分析
go build -gcflags='-m -m' ./...     # 更详细
```

常见逃逸原因：

| 原因 | 例子 | 改法 |
|---|---|---|
| 返回局部变量指针 | `return &T{}` | 不可避免；考虑值返回 |
| 装进接口 | `fmt.Println(x)` | 热路径避免 `any` |
| 闭包捕获 | `go func(){ use(x) }()` | 按值传参 |
| 切片扩容 | `append` 未知容量 | `make([]T, 0, n)` |
| 大对象 | 超过栈上限 | 复用/池化 |

```go
// 差：每次循环分配
for _, s := range xs { out = append(out, []byte(s)) }

// 好：预分配
out := make([][]byte, 0, len(xs))
for _, s := range xs { out = append(out, []byte(s)) }
```

### 18.3 字符串与字节

- `strings.Builder` + `Grow` 代替 `+=`；
- `[]byte` → `string` 需要拷贝时，用 `unsafe.String` 前确认源不再修改；
- `io.Copy` 内部有 32 KB 缓冲，不要自己写小缓冲循环；
- `strconv.AppendInt` 直接追加到 `[]byte`，比 `Sprintf` 少一次分配。

### 18.4 锁与并行

| 问题 | 表现 | 改法 |
|---|---|---|
| 锁粒度过粗 | CPU 上不去、`mutex` profile 高 | 分片锁、读写锁、无锁原子 |
| 伪共享 | 多核扩展差 | 填充缓存行（`cpu.CacheLinePad`） |
| goroutine 过多 | 调度开销、内存涨 | 限流（`semaphore`、worker pool） |
| channel 当锁用 | 延迟高 | 简单场景用 `Mutex` |

```go
// 分片计数，降低单锁竞争
type ShardedCounter struct {
    shards [16]struct {
        mu sync.Mutex
        n  int64
    }
}
```

### 18.5 PGO（1.21+）

```bash
# 1) 采集代表性负载的 CPU profile
go test -bench=. -cpuprofile=default.pgo
# 2) 把 default.pgo 放在 main 包目录，正常构建即可
go build ./cmd/api
```

典型收益 2%–14%（内联更激进、分支预测更好）。profile 要覆盖真实热点，否则可能变慢。

### 18.6 工具链

| 工具 | 用途 |
|---|---|
| `pprof` | CPU/内存/锁/阻塞 |
| `go tool trace` | 调度延迟、GC、goroutine 阻塞 |
| `benchstat` | 基准差异的统计显著性 |
| `fieldalignment` | 结构体字段重排省内存 |
| `go build -ldflags=-w -s` | 减小二进制 |
| `-gcflags=-B` | 关闭边界检查（仅调试，不安全） |

---

## 19. 项目骨架模板

### 19.1 目录

```text
app/
├── go.mod
├── Makefile
├── .golangci.yml
├── cmd/
│   ├── api/main.go
│   └── worker/main.go
├── internal/
│   ├── config/config.go
│   ├── app/app.go           # 组装依赖、生命周期
│   ├── transport/http/      # 路由、中间件、DTO
│   ├── service/             # 业务逻辑
│   └── store/               # 存储接口 + 实现
├── migrations/
└── docs/
```

### 19.2 配置

```go
type Config struct {
    Addr     string        `env:"ADDR" default:":8080"`
    Database string        `env:"DATABASE_URL,required"`
    Timeout  time.Duration `env:"TIMEOUT" default:"5s"`
}

func Load() (Config, error) {
    var c Config
    if err := env.Parse(&c); err != nil {   // caarlos0/env 或手写
        return c, fmt.Errorf("config: %w", err)
    }
    return c, c.Validate()
}
```

优先级：**命令行 > 环境变量 > 配置文件 > 默认值**；启动时打印生效配置（脱敏），省掉一半排查。

### 19.3 错误边界

```go
// internal/app/app.go
func (a *App) Run(ctx context.Context) error {
    eg, ctx := errgroup.WithContext(ctx)
    eg.Go(func() error { return a.http.Run(ctx) })
    eg.Go(func() error { return a.worker.Run(ctx) })
    return eg.Wait()
}
```

- 每个子系统返回 `error`，由 `app` 统一决定"是否致命"；
- 对外错误码集中映射，不散落在 handler；
- 内部错误带上下文，日志只记一次。

### 19.4 优雅退出

```go
func main() {
    ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
    defer stop()
    if err := run(ctx); err != nil {
        slog.Error("exit", "err", err)
        os.Exit(1)
    }
}

func run(ctx context.Context) error {
    cfg, err := config.Load()
    if err != nil { return err }
    a, err := app.New(cfg)
    if err != nil { return err }
    return a.Run(ctx)
}
```

### 19.5 `Makefile`

```makefile
.PHONY: build test lint check
build:
	go build -trimpath -o bin/app ./cmd/api

test:
	go test -race -cover ./...

lint:
	gofmt -l .
	go vet ./...
	staticcheck ./...

check: lint test build
```

### 19.6 验收清单

- [ ] `gofmt -l .` 无输出
- [ ] `go vet ./...`、`staticcheck ./...` 通过
- [ ] `go test -race ./...` 通过，覆盖率覆盖错误分支
- [ ] `go mod tidy` 后无 diff
- [ ] 启动失败有清晰错误与非零退出码
- [ ] 有 `ReadHeaderTimeout` 等超时设置
- [ ] SIGTERM 后 10 秒内优雅退出
- [ ] 日志结构化，含 service/version
- [ ] pprof 只绑 loopback 或加鉴权
- [ ] `govulncheck ./...` 无高危

---

# 第三部分　运行时深挖

> 以下内容基于 Go 1.27 的 `src/runtime`。实现细节会随版本变化，**结论比结构体字段名更值得记住**。

## 20. 编译、链接与启动

### 20.1 `go build` 的流水线

```text
源码 → parser → 类型检查 → 中间表示（SSA）→ 机器码（.o）
     → 链接器（内联、重定位、符号解析）→ 可执行文件
```

| 阶段 | 做什么 | 可观察点 |
|---|---|---|
| 类型检查 | 泛型实例化、接口方法集校验 | `go build` 报错 |
| 逃逸分析 | 决定变量在栈还是堆 | `-gcflags='-m'` |
| 内联 | 把小的函数体展开 | `-gcflags='-m'` 的 `can inline` |
| SSA 优化 | 常量折叠、边界检查消除 | `-gcflags='-d=ssa/...'` |
| 链接 | 合并包、生成符号表 | `-ldflags`、`go tool nm` |

构建缓存以"编译输入哈希"为键，改了 `-gcflags` 或 `GOEXPERIMENT` 都会导致全量重建。

### 20.2 逃逸分析

逃逸分析回答一个问题：**这个变量能不能放在栈上？** 放栈上的对象随函数返回自动回收，零 GC 成本。

```bash
go build -gcflags='-m' ./...
# ./main.go:10:6: moved to heap: x
# ./main.go:12:14: parameter y leaks to {heap} with derefs=0
```

常见逃逸模式：

```go
func bad() *int { x := 1; return &x }          // 逃逸：指针返回

func good() int { x := 1; return x }           // 不逃逸

func sink(v any) { }                            // 装箱导致逃逸
sink(42)

var global *T
func set(t *T) { global = t }                   // 逃逸到全局

func closure() func() int { x := 0; return func() int { return x } }  // x 逃逸
```

⚠️ 逃逸分析是**保守**的：编译器无法证明安全时一律放堆上。因此"少用 `any`、少返回指针、闭包按值捕获"能实质减少分配。

### 20.3 内联

内联把小函数体直接展开，消除调用开销并开放后续优化（常量传播、逃逸分析更精确）。

- 预算约 **80 个 AST 节点**，超过不内联；
- 含 `for`/`select`/`defer`（部分）/`recover` 的函数通常不内联；
- 1.12 起支持**中栈内联**（调用链深处的函数也可内联）；
- 递归函数不会被内联。

```go
//go:noinline
func hot() { }        // 调试/基准时禁止内联，确保测的是调用开销
```

🧭 不要为了内联手写重复代码；先看 `-m` 输出，确实需要时再拆小函数。

### 20.4 边界检查消除

```go
func sum(xs []int) int {
    t := 0
    for i := 0; i < len(xs); i++ {
        t += xs[i]        // 每次访问都检查 i < len(xs)
    }
    return t
}

func sum2(xs []int) int {
    t := 0
    for _, x := range xs { t += x }   // range 只检查一次，通常更快
    return t
}
```

编译器的 BCE（bounds check elimination）能消除"已被同一条件约束"的重复检查：

```go
if len(xs) > 3 {
    _ = xs[0] + xs[1] + xs[2] + xs[3]   // 四次检查可能被合并
}
```

用 `-gcflags='-d=ssa/check_bce/debug=1'` 查看哪些检查被保留。

### 20.5 链接与二进制布局

```bash
go tool nm bin/app | head          # 符号表
go tool objdump -s main.main bin/app
go version -m bin/app              # 读取嵌入的构建信息
```

| 段 | 内容 |
|---|---|
| `.text` | 机器码（最大） |
| `.rodata` | 字符串常量、类型元数据、itab |
| `.data`/`.bss` | 全局变量 |
| `.gopclntab` | 行号、函数边界（panic 栈依赖它） |
| `.noptrdata` | 不含指针的全局数据 |

- `-ldflags="-s -w"` 去符号与 DWARF，能减 20%–30%，但 panic 栈仍可用（依赖 `gopclntab`）；
- `-trimpath` 去掉绝对路径，保证可复现构建；
- `go version -m` 能读出 module 版本与 `-ldflags`，是排查"线上跑的是哪个版本"的最快方式。

### 20.6 启动过程

```text
内核加载 ELF → _rt0_amd64（汇编）
  → runtime·rt0_go
    → schedinit        // 初始化调度器、P、栈、内存分配器
    → newproc(main_main)  // 把 main.main 包装成 goroutine
    → mstart           // 主线程进入调度循环
      → schedule() → main_main → main.main
```

`init` 的顺序（规范明确）：

1. 被导入的包先初始化（依赖顺序，后序）；
2. 包级变量按**声明顺序**初始化（有依赖时按依赖顺序）；
3. `init()` 函数按**文件名排序**后的声明顺序执行；
4. 最后执行 `main.main`。

⚠️ `init()` 里做 IO、连数据库、起 goroutine 会让"启动顺序"变成隐式契约；用显式的 `New`/`Run` 代替。

---

## 21. 调度器 GMP

### 21.1 为什么需要三者

| 角色 | 含义 | 数量 |
|---|---|---|
| **G**（goroutine） | 用户态协程：栈 + 寄存器现场 + 状态 | 可达百万 |
| **M**（machine） | OS 线程，真正执行代码 | 按需创建，默认上限 10000 |
| **P**（processor） | 执行上下文：本地队列、mcache、调度状态 | 默认 = GOMAXPROCS |

P 的存在是为了**减少锁竞争**：每个 P 有自己的运行队列与内存缓存，M 必须绑定 P 才能执行 G。

```text
M0 ──绑定── P0 ──本地队列── [G1, G2, G3]
M1 ──绑定── P1 ──本地队列── [G4]
              └── 全局队列（global runq）
```

### 21.2 关键结构

```go
type g struct {
    stack       stack      // [lo, hi)
    sched       gobuf      // 保存的 SP/PC/BP
    atomicstatus atomic.Uint32 // _Grunnable/_Grunning/_Gwaiting/...
    goid        uint64
    m           *m
    // ...
}

type p struct {
    runqhead uint32
    runqtail uint32
    runq     [256]guintptr   // 本地队列，无锁环形缓冲
    runnext  guintptr        // 下一个要跑的 G（优先于 runq）
    mcache   *mcache
    status   uint32
}

type m struct {
    g0      *g        // 调度用的栈（不是用户 goroutine 的栈）
    curg    *g        // 当前运行的 G
    p       *p
    spinning bool     // 正在找活干
    // ...
}
```

### 21.3 调度主循环

`schedule()` 的查找顺序（简化）：

1. `runnext`（刚被唤醒的 G，优先执行，提升局部性）；
2. 本地 `runq`（最多 256 个）；
3. 全局队列（每 61 次调度检查一次，防止本地队列饿死全局）；
4. `netpoll`（有网络事件就绪的 G）；
5. 从其他 P **偷一半**（work stealing）；
6. 都没有 → 休眠 M，等待唤醒。

```go
// 伪代码
func schedule() {
    for {
        gp := findRunnable()   // 上述 1–6
        execute(gp)            // 切换栈，运行 gp
    }
}
```

### 21.4 work stealing

- 偷取方随机选一个受害者 P，偷其队列的**一半**（粒度大，减少偷取次数）；
- 本地队列空且全局队列空时才偷，避免无谓的原子操作；
- 这一机制让"任务分布不均"自动均衡，不需要用户手动分片。

### 21.5 系统调用与 netpoller

**阻塞式系统调用**（文件 IO、`syscall.Read`）：

```text
G 进入 syscall → M 被阻塞 → P 与 M 解绑（handoff）
→ 空闲 M 接管 P，继续跑其他 G
→ syscall 返回 → G 变为 runnable，重新排队
```

**网络 IO**：

```text
G 读 socket 未就绪 → 注册到 netpoller（epoll/kqueue）
→ G 状态 _Gwaiting，M 去跑别的
→ 事件就绪 → netpoller 唤醒 G，放入 P 的队列
```

因此：**网络 IO 用 goroutine 很便宜；阻塞式文件 IO 会占用线程**。高并发文件处理要用 `io_uring` 库或限制并发度（`semaphore`）。

### 21.6 抢占

- 1.13 之前：只有函数调用处的**协作式**抢占，长循环会卡住调度；
- 1.14 起：**基于信号的异步抢占**（`SIGURG`），运行超过 10ms 的 G 会被打断；
- `sysmon` 后台线程每 10ms–10s 检查：长时间运行的 G、处于 syscall 的 P、netpoll 等。

```go
// 不会被抢占的极端情况：纯计算且无函数调用（现在也会被信号打断）
for { }
```

🧭 不要把"抢占"当成并发正确性的依赖；它只保证调度公平，不保证内存顺序。

### 21.7 栈增长与收缩

```text
初始栈 2 KB → 函数序言检查 SP 是否越界
→ 越界触发 morestack → 分配 2 倍大小的新栈 → 复制旧栈 → 调整指针 → 继续
```

- 最大栈：64 位 1 GB，32 位 250 MB；
- GC 时若发现栈使用不足 1/4，会**收缩**，避免长期占用；
- 栈复制需要调整所有指向栈的指针，因此**指向栈的指针不能保存在堆上长期存在**（逃逸分析保证）。

### 21.8 常见调度问题

| 现象 | 原因 | 处理 |
|---|---|---|
| CPU 用不满 | GOMAXPROCS 太小 / 大量阻塞 | 检查 cgroup 限额、用 `spawn` 型并发 |
| 延迟毛刺 | GC、长任务未抢占、锁竞争 | 看 trace，拆分长任务 |
| goroutine 爆炸 | 每个请求无限起 goroutine | 用 `semaphore`/worker pool 限流 |
| M 数量暴涨 | 大量阻塞 syscall | 限制并发、用异步 IO |
| `GOMAXPROCS` 与容器不符 | 1.25 前默认取宿主核数 | 升级或显式设置 |

```bash
GODEBUG=schedtrace=1000 ./app     # 每秒打印调度器状态
go tool trace trace.out           # 可视化
```

---

## 22. 内存分配器

### 22.1 三级结构

```text
mcache（每 P 私有，无锁）
  ↓ 缺货
mcentral（每 size class 一个，全局，有锁）
  ↓ 缺页
mheap（页堆，向 OS 申请 arena）
```

| 层 | 粒度 | 并发 |
|---|---|---|
| mcache | 按 size class 缓存 span | 无锁（P 独占） |
| mcentral | 按 size class 管理 span 列表 | 自旋锁 |
| mheap | 8 KB 页、64 MB arena | 全局锁（分片） |

### 22.2 size class 与 span

- 小对象（≤32 KB）按 **size class** 分配，约 67 档（8、16、24、32、48、64、80…32768 字节）；
- 每个 span 由连续页组成（1 页 = 8 KB），按 size class 切成等大对象；
- 大对象（>32 KB）直接从 mheap 分配**大 span**，不经过 mcache；
- span 的元数据（`mspan`）记录 `startAddr`、`npages`、`freeindex`、`allocBits`。

```text
span(8KB, class=16B) → [obj0][obj1]...[obj511]
```

### 22.3 tiny allocator

小于 16 字节、**不含指针**的对象会走 tiny allocator：多个小对象合并进同一个 16 字节块。

```go
a := new(int8)     // 可能与 b 共享同一个 16B 块
b := new(int8)
```

好处是减少碎片与分配次数；代价是这些对象的生命周期被"绑定"到块上（一个存活，整块不回收）。

⚠️ 因此**不要把长生命周期的小对象与短生命周期的混在一起**（虽然无法直接控制，但可以通过结构体聚合缓解）。

### 22.4 分配路径

```go
func mallocgc(size uintptr, typ *_type, needzero bool) unsafe.Pointer {
    if size <= maxTinySize && typ == nil { /* tiny */ }
    if size <= maxSmallSize && typ != nil { /* 按 size class 走 mcache */ }
    /* 大对象走 mheap */
}
```

关键点：

- **含指针的对象**与**不含指针的对象**分开管理（GC 扫描需要）；
- 分配时可能触发 GC（达到阈值）或**辅助标记**；
- `needzero` 决定是否清零——`make([]T, n)` 一定清零，`new(T)` 也清零。

### 22.5 Go 1.27 的 size-specialized malloc

1.27 让编译器为**小于 80 字节**的分配生成专门的调用，避免通用路径的分支与参数准备：

- 小分配最多快 **30%**；
- 整体分配密集型程序约 **1%** 提升；
- 二进制增大约 60 KB；
- 可用 `GOEXPERIMENT=nosizespecializedmalloc` 关闭（预计 1.28 移除）。

这是一次典型的"用少量代码体积换常量级开销"的优化。

### 22.6 碎片与 RSS

| 现象 | 原因 | 手段 |
|---|---|---|
| RSS 高但堆不大 | span 未归还、`MADV_FREE` 延迟 | `GODEBUG=madvdontneed=1`（旧版本）、`debug.FreeOSMemory()` |
| 堆持续增长 | 泄漏（goroutine、map、缓存） | heap profile + `pprof -alloc_space` |
| 分配慢 | size class 不匹配、锁竞争 | 对齐对象大小、减少大对象 |

```bash
go tool pprof -http=:0 http://localhost:6060/debug/pprof/heap
# 关注 inuse_space（当前占用）与 alloc_space（累计分配）
```

🧭 容器里要同时设置 **`GOMEMLIMIT`** 与合理的 `GOGC`：只设 `GOGC` 时，堆大、对象存活率高的程序会 OOM；只设 `GOMEMLIMIT` 时，GC 可能过于频繁。

---

## 23. 垃圾回收

### 23.1 三色标记与混合写屏障

Go 使用**并发标记-清除**，非分代、非移动（对象地址不变）。

```text
白色：未访问，可能被回收
灰色：已发现，待扫描
黑色：已扫描，其引用都已标记
```

并发标记的难点是"用户程序同时修改指针"。Go 1.8 起使用**混合写屏障**（Dijkstra 插入屏障 + Yuasa 删除屏障）：

```text
写指针时：把旧值标灰，把新值标灰
```

这样无论指针怎么改，都不会漏标；代价是写操作有额外开销。

### 23.2 GC 周期

```text
Sweep Termination（STW，极短）
  → Mark（并发，占大头）
    → Mark Termination（STW，极短）
      → Sweep（并发，惰性：分配时才清扫对应 span）
```

- **STW 只有两小段**，通常几十微秒到几毫秒；
- 标记阶段与用户代码并发，默认最多占用 **25% 的 CPU**；
- 清扫是**惰性**的：分配新对象时才清理对应 span，避免一次性扫描整个堆。

### 23.3 Pacer、`GOGC` 与 `GOMEMLIMIT`

```text
GOGC=100（默认）：下次 GC 在堆大小达到 "存活对象 × 2" 时触发
GOGC=off：关闭 GC（仅调试）
GOMEMLIMIT=2GiB：软上限，接近时更积极地 GC
```

| 参数 | 调大 | 调小 |
|---|---|---|
| `GOGC` | GC 更少、内存更高、CPU 更低 | GC 更频繁、内存更低、CPU 更高 |
| `GOMEMLIMIT` | 允许更高内存 | 更早触发 GC，可能影响吞吐 |

实际策略：

```bash
# 容器 4GiB，留 20% 给非堆内存
GOMEMLIMIT=3200MiB GOGC=200 ./app
```

⚠️ `GOMEMLIMIT` 是**软限制**：如果分配速度超过回收速度，运行时仍会 OOM（此时会打印 "memory limit exceeded"）。

### 23.4 辅助标记与 STW

- 如果用户 goroutine 分配太快，GC 标记跟不上，分配方会被**罚做标记工作**（mutator assist），表现为"业务代码变慢"；
- 这正是"高分配率 → 延迟毛刺"的根源；
- 降低分配率（对象复用、减少装箱）比调 GC 参数更有效。

```bash
GODEBUG=gctrace=1 ./app
# gc 12 @3.421s 0%: 0.012+1.2+0.003 ms clock, 0.15+0.8/1.4/0.007+0.04 ms cpu, 8->9->5 MB, 10 MB goal, 8 P
```

读法：`gc N @时间 占CPU%: STW(扫尾)+并发标记+STW(收尾)`，最后是 `堆大小变化`。

### 23.5 对象生命周期与清理

```go
// 旧方式：finalizer，对象会被"复活"一轮，至少延迟一个 GC 周期
runtime.SetFinalizer(obj, func(o *T) { /* 释放外部资源 */ })

// 1.24+：AddCleanup，不复活对象，可在对象不可达后执行
runtime.AddCleanup(obj, func(h Handle) { h.Close() }, handle)

// 1.24+：弱引用，不阻止回收
var wp weak.Pointer[Node]
if n := wp.Value(); n != nil { /* 还活着 */ }
```

🧭 不要用 finalizer 管理**必须及时释放**的资源（文件、锁、连接）；用 `defer Close()` 或显式生命周期。finalizer 只适合"兜底告警"。

### 23.6 调优方法论

1. **先看 heap profile**：`inuse_space` 找常驻内存，`alloc_space` 找分配热点；
2. **区分"堆大"与"分配多"**：前者调 `GOGC`/`GOMEMLIMIT`，后者改代码；
3. **看 gctrace 的 CPU 占比**：超过 25% 说明分配压力大或存活率高；
4. **不要盲目 `debug.FreeOSMemory()`**：它会强制一次 STW + 归还内存，只适合批处理结束等明确时点；
5. **容器里始终设置 `GOMEMLIMIT`**，并留出 20%–30% 给栈、元数据、cgo。

| 症状 | 可能原因 | 验证 |
|---|---|---|
| GC 频繁但堆不大 | 分配率过高 | `alloc_space` |
| GC 后内存不降 | 存活对象多 / 泄漏 | `inuse_space` 趋势 |
| 延迟毛刺 | STW、assist、系统调用 | `go tool trace` |
| OOMKilled | 未设 `GOMEMLIMIT`、非堆内存超限 | 容器指标 + `GODEBUG=gctrace` |

---

## 24. channel、select 与锁的运行时实现

### 24.1 `hchan` 结构

```go
type hchan struct {
    qcount   uint      // 缓冲区中元素个数
    dataqsiz uint      // 缓冲区容量
    buf      unsafe.Pointer
    elemsize uint16
    closed   uint32
    elemtype *_type
    sendx    uint      // 下一个写入位置
    recvx    uint      // 下一个读取位置
    recvq    waitq     // 等待接收的 sudog 队列
    sendq    waitq     // 等待发送的 sudog 队列
    lock     mutex
}
```

- **无缓冲 channel** 的 `dataqsiz=0`，发送必须直接交给接收方；
- 缓冲区是**环形队列**，`sendx`/`recvx` 回绕；
- 等待的 goroutine 用 `sudog` 表示，挂在 `sendq`/`recvq` 上。

### 24.2 发送路径

```text
ch <- v
  ├─ 有等待的接收方（recvq 非空）→ 直接把 v 拷到接收方的栈，唤醒它
  ├─ 缓冲区未满 → 写入 buf[sendx]，sendx++
  ├─ 缓冲区满 / 无缓冲且无人等 → 当前 G 包装成 sudog，挂到 sendq，park
  └─ channel 已关闭 → panic: send on closed channel
```

关键点：**发送是"值拷贝"**。发送一个大结构体会复制整个结构体，应传指针。

### 24.3 接收路径

```text
v := <-ch
  ├─ 缓冲区有数据 → 取 buf[recvx]，若有等待的发送方则把它排入缓冲
  ├─ sendq 有等待者 → 直接取它的值（无缓冲场景）
  ├─ 已关闭且缓冲空 → 返回零值，ok=false
  └─ 都没有 → 当前 G 挂到 recvq，park
```

⚠️ `for v := range ch` 在 channel 关闭后自动退出；但如果**没人关闭**，循环永远阻塞——这是 goroutine 泄漏最常见的形态。

### 24.4 `close` 的语义

- `close` 是**广播**：所有阻塞的接收方被唤醒并收到零值；
- **只能关闭一次**，重复关闭 panic；
- **向已关闭的 channel 发送 panic**；
- 关闭后仍可接收完缓冲区剩余值，之后返回零值；
- `close(nil)` panic。

运行时用 `closed` 标志 + 唤醒 `recvq` 全部 sudog 实现；没有"关闭回调"。

### 24.5 `select` 的实现

```text
selectgo(cases):
  1. 生成 pollorder：case 的随机排列（保证公平）
  2. 生成 lockorder：按 channel 地址排序（避免死锁）
  3. 按 pollorder 扫描一遍，有就绪的立即执行
  4. 有 default → 执行 default
  5. 否则把所有 case 的 sudog 挂到对应 channel，park
  6. 被唤醒后，从就绪的 case 中挑一个，清理其他 sudog
```

因此：

- **多 case 就绪时随机**，不能依赖优先级；需要优先级时用嵌套 `select` 或 `biased` 式的两段判断；
- `select` 的 case 数量影响开销（每次都要构造 sudog）；
- 空 `select{}` 永久阻塞，可用于"让 goroutine 永不退出"。

### 24.6 `Mutex` 的两种模式

```text
state: locked(1) | woken(1) | starving(1) | waiter count
```

- **正常模式**：新来的 goroutine 可以"插队"自旋抢锁，等待者可能饿死；
- **饥饿模式**：等待超过 **1ms** 后切换，锁直接交给队首，新来者不自旋；
- 自旋最多 4 次，且要求多核、临界区短。

```go
// 不要复制已使用的 Mutex
type S struct{ mu sync.Mutex }
// s2 := s1        // ❌ go vet 会报 copylocks
```

### 24.7 信号量与 `netpoll`

- `Mutex`/`Cond`/`channel` 的阻塞最终都落到 `runtime.semacquire`（信号量）；
- 网络 IO 不走信号量，而是注册到 **netpoller**（Linux epoll、macOS kqueue）；
- `netpoll` 返回就绪的 goroutine 列表，调度器把它们放回运行队列；
- 这解释了"为什么 Go 能轻松处理十万并发连接"：等待连接不占线程。

---

## 25. 接口、itab 与 map 的运行时

### 25.1 `iface` 与 `eface`

```go
type iface struct {          // 非空接口
    tab  *itab
    data unsafe.Pointer
}

type eface struct {          // 空接口 any
    _type *_type
    data  unsafe.Pointer
}
```

- 非空接口需要 **itab**：接口类型 + 具体类型 + 方法表；
- 空接口只需要类型元数据（方法表为空）；
- `data` 指向堆上的值；**指针形状且大小等于一个字的值**可能直接存入 `data`（不额外分配）。

### 25.2 `itab` 的构造与缓存

```go
type itab struct {
    Inter *interfacetype
    Type  *_type
    Hash  uint32
    Fun   [1]uintptr     // 方法地址数组（变长）
}
```

- 首次把某类型赋给某接口时，运行时构造 itab，填入方法地址；
- 构造结果存入全局 **itabTable**（开放寻址哈希表），后续命中缓存；
- 因此"接口转换"第一次有开销，之后接近常数。

### 25.3 接口调用的成本

```go
var r io.Reader = f
r.Read(buf)      // 间接调用：load itab.Fun[i] → call
```

| 成本 | 说明 |
|---|---|
| 装箱 | 值类型转接口可能分配（逃逸） |
| 间接调用 | 无法内联，分支预测可能失败 |
| 缓存未命中 | itab 查找（首次） |

🧭 热路径（每请求数百万次）用泛型或具体类型；每请求几次的调用用接口完全没问题。

### 25.4 `map` 的结构

```go
type hmap struct {
    count     int
    flags     uint8
    B         uint8        // bucket 数量 = 2^B
    noverflow uint16
    hash0     uint32
    buckets   unsafe.Pointer  // 2^B 个 bmap
    oldbuckets unsafe.Pointer // 扩容中的旧桶
    nevacuate uintptr
}

type bmap struct {
    tophash [8]uint8       // 每个槽的高 8 位哈希
    // 之后是 8 个 key、8 个 value、overflow 指针
}
```

- 每个桶存 **8 个键值对**，溢出时链到新桶；
- 负载因子约 **6.5**，超过就扩容；
- `tophash` 让查找可以先比较 1 字节，减少完整 key 比较。

### 25.5 扩容与渐进搬迁

```text
count > 6.5 × 2^B  → 翻倍扩容（B+1）
溢出桶过多（尤其大量删除后）→ 等量扩容（B 不变，重新紧凑）
```

搬迁是**渐进式**的：

- 写操作（insert/delete）时搬迁 1–2 个旧桶；
- 读操作可能同时查新桶与旧桶；
- 搬迁完成后释放旧桶。

这避免了"大 map 扩容导致的长时间 STW"。

### 25.6 遍历随机与并发检测

```go
for k := range m { }    // 起始桶与桶内偏移都随机
```

- 随机化是**故意的**：防止程序依赖遍历顺序；
- 因此"两次遍历顺序不同"是正常现象，不是 bug；
- 并发读写由 `flags` 的 `hashWriting` 位检测，触发 `fatal error: concurrent map read and map write`（不可 recover）。

### 25.7 map 的替代方案

| 场景 | 方案 |
|---|---|
| 读多写少、键基本固定 | `sync.Map` |
| 高并发计数 | 分片 map（每片一把锁） |
| 小集合（<16 元素） | 切片 + 线性查找（更快、无哈希开销） |
| 需要顺序 | `map` + 排序键，或 `BTree` 实现 |
| 需要 LRU | `map` + 双向链表（或用成熟库） |

```go
// 分片 map：降低单锁竞争
type Sharded[K comparable, V any] struct {
    shards [32]struct {
        mu sync.RWMutex
        m  map[K]V
    }
}
```

---

## 26. 诊断与版本演进

### 26.1 五类 profile

| profile | 采集 | 看什么 |
|---|---|---|
| `cpu` | 30s 采样 | 热点函数、内联后的调用栈 |
| `heap` | 采样分配 | `inuse_space`（常驻）、`alloc_space`（累计） |
| `goroutine` | 快照 | 阻塞点、数量、栈 |
| `mutex` | 竞争采样 | 锁等待时间 |
| `block` | 阻塞采样 | channel/select 等待 |

```bash
go tool pprof -http=:0 http://127.0.0.1:6060/debug/pprof/heap
go tool pprof -diff_base=old.pprof new.pprof    # 对比两次快照
```

### 26.2 trace

```bash
go tool trace trace.out
```

能回答的问题：

- 某个 P 为什么空闲/繁忙？
- goroutine 在等什么（channel、锁、syscall、GC）？
- GC 的 STW 与辅助标记占了多少时间？
- 网络轮询是否成为瓶颈？

🧭 pprof 看"CPU 花在哪"，trace 看"时间花在哪"；延迟问题优先 trace。

### 26.3 内存泄漏排查

```text
1. 确认是堆增长还是 RSS 增长
2. heap profile 取两个时间点，diff inuse_space
3. 看对象类型与分配栈
4. 常见来源：全局 map 只增不减、goroutine 持有引用、time.Ticker 未 Stop、切片截取大数组
```

```go
// 典型泄漏：全局缓存无上限
var cache = map[string][]byte{}
func Put(k string, v []byte) { cache[k] = v }   // 永不删除
```

### 26.4 goroutine 泄漏排查（1.27）

```bash
curl http://127.0.0.1:6060/debug/pprof/goroutineleak?debug=1
```

原理：如果 goroutine 阻塞在某个并发原语 P 上，而 P 从所有可运行的 goroutine 都**不可达**，那么这个 goroutine 永远无法被唤醒，判定为泄漏。

```go
// 这会被识别为泄漏：无人关闭 ch
go func() { for range ch { } }()
```

局限：通过全局变量可达的原语不会被判定为泄漏；因此它只能覆盖"一类"泄漏，不能替代人工审查。

### 26.5 延迟毛刺归因

| 来源 | 特征 | 手段 |
|---|---|---|
| GC | 周期性、与分配率相关 | `gctrace`、`GOGC`/`GOMEMLIMIT` |
| 调度 | 大量 runnable、P 不足 | `schedtrace`、trace |
| 锁竞争 | 特定临界区、并发越高越差 | `mutex` profile、分片 |
| 系统调用 | 阻塞式 IO、cgo | `block` profile、异步化 |
| 网络 | 对端慢、DNS | 客户端超时、连接池 |
| 页错误/内存 | RSS 抖动 | `GODEBUG=madvdontneed`、预分配 |

### 26.6 `GODEBUG` 与升级 checklist

Go 通过 `GODEBUG` 让"旧行为"在新版本上继续生效，避免升级即破坏。升级时：

1. 读 release notes 的 "Changes to the language" 与 "Runtime"；
2. 搜代码里是否依赖被改的行为（map 顺序、timer 通道容量、`GOMAXPROCS` 默认值）；
3. 跑 `go vet`（1.27 起 `go test` 自带 `stdversion`）；
4. 跑 `-race` 与基准，对比 `benchstat`；
5. 最后再改 `go.mod` 的 `go` 指令——**这一步才真正启用新语义**。

```bash
# 查当前二进制支持哪些 GODEBUG
go tool dist list >/dev/null
go doc internal/godebug   # 开发者视角
```

### 26.7 不要依赖的实现细节

| 细节 | 为什么不能依赖 |
|---|---|
| map 遍历顺序 | 规范明确随机 |
| goroutine 调度顺序 | 无保证 |
| 接口 itab 的地址/数量 | 实现细节 |
| `runtime.Gosched` 的具体效果 | 只是"让出"，不保证顺序 |
| 结构体内存布局（无 `unsafe` 时） | 编译器可调整填充 |
| GC 触发时机 | 受 pacer、环境变量影响 |
| 函数内联决策 | 随版本变化 |
| `time.Timer` 通道容量 | 1.23 改为同步，1.27 删除了回退开关 |

🧭 依赖实现细节的代码，升级时一定会出问题；把它写进 ADR（架构决策记录）并加集成测试。

---

## 附录 A　常用命令速查

| 命令 | 用途 |
|---|---|
| `go mod init <path>` | 初始化模块 |
| `go mod tidy` | 同步依赖 |
| `go mod tidy -diff` | 只看差异（1.23+） |
| `go get pkg@version` | 添加/升级依赖 |
| `go get -tool pkg` | 添加工具依赖（1.24+） |
| `go list -m -u all` | 可升级模块 |
| `go list -deps ./...` | 依赖图 |
| `go build -trimpath -o bin/app ./cmd/app` | 可复现构建 |
| `GOOS=linux GOARCH=arm64 CGO_ENABLED=0 go build` | 交叉编译 |
| `go test -race -cover ./...` | 竞态 + 覆盖率 |
| `go test -run TestX -v` | 只跑某个测试 |
| `go test -bench=. -benchmem -count=5` | 基准 |
| `go test -fuzz=FuzzX -fuzztime=30s` | 模糊测试 |
| `go vet ./...` | 静态检查 |
| `staticcheck ./...` | 深度静态检查 |
| `govulncheck ./...` | 漏洞扫描 |
| `go tool pprof -http=:0 cpu.out` | 可视化 profile |
| `go tool trace trace.out` | 调度追踪 |
| `go tool nm bin/app` | 符号表 |
| `go version -m bin/app` | 读取构建信息 |
| `go doc pkg.Symbol` | 查文档 |
| `go doc example.com/pkg@v1.2.3` | 查指定版本（1.27+） |
| `gofmt -l -w .` | 格式化 |

---

## 附录 B　Go 与 Rust 对照

| 概念 | Go | Rust |
|---|---|---|
| 内存管理 | 并发 GC，非移动 | 所有权 + 借用检查，无 GC |
| 多态 | 隐式接口 + 泛型 | trait（静态）+ `dyn`（动态） |
| 空值 | `nil`（指针/接口/map/slice/channel） | `Option<T>` |
| 错误 | 返回值 `error` + `errors.Is/As` | `Result<T, E>` + `?` |
| 资源释放 | `defer` | `Drop`（RAII） |
| 并发 | goroutine + channel，runtime 调度 | 线程 + `Send/Sync`；async 需 runtime |
| 可变性 | 默认可变、共享需同步 | 默认不可变，`&mut` 独占 |
| 泛型实现 | GC shape stenciling | 单态化 |
| 编译速度 | 快（无 LLVM 级优化） | 慢（借用检查 + 单态化 + LLVM） |
| 二进制 | 静态、较大、含 runtime | 静态、可选 `no_std`、可更小 |
| 学习曲线 | 语法少，工程约束多 | 语法与类型系统复杂 |

🧭 从 Go 转 Rust 最容易带过去的三个习惯：到处传 `any`（应换泛型/枚举）、用 `nil` 表示"没有"（应换 `Option`）、用 `defer` 管理一切（Rust 用 `Drop` 但受借用规则约束）。

---

## 附录 C　Code Review 检查清单

- [ ] 错误是否都处理了？有没有 `_ = err` 或只打日志后继续？
- [ ] 错误是否用 `%w` 包装，调用方能否 `errors.Is/As`？
- [ ] `goroutine` 是否都有退出路径（context、close、超时）？
- [ ] `channel` 是否由发送方关闭？有没有向已关闭 channel 发送的风险？
- [ ] `context` 是否作为第一个参数贯穿 IO 调用？`cancel` 是否 `defer`？
- [ ] `http.Client` / `http.Server` 是否设置了超时？
- [ ] `resp.Body` / `rows` / `file` 是否关闭？
- [ ] `map` 是否可能并发读写？是否用了 `sync.Map` 却不满足其适用场景？
- [ ] 切片是否可能因截取而持有大数组？是否需要 `slices.Clone`？
- [ ] 是否在热路径用了 `fmt.Sprintf`、反射、`any` 装箱？
- [ ] 公开接口是否新增了方法（破坏实现者）？
- [ ] 是否依赖 map 遍历顺序或 goroutine 调度顺序？
- [ ] 日志是否结构化、是否泄漏敏感信息？
- [ ] 新增依赖是否必要？license、维护状态、漏洞是否确认？
- [ ] 是否有对应测试（含错误分支与并发分支）？

---

## 附录 D　延伸阅读

| 资源 | 用途 |
|---|---|
| The Go Programming Language Specification | 语义的最终依据 |
| Effective Go | 惯用法与风格 |
| Go Code Review Comments | 评审时的默认口径 |
| Go Release Notes（1.22–1.27） | 版本差异与迁移 |
| `src/runtime`（`proc.go`、`malloc.go`、`mgc.go`、`chan.go`、`map.go`） | 运行时实现 |
| "The Go scheduler"（Dmitry Vyukov） | GMP 设计动机 |
| "Go's Memory Model"（官方文档） | happens-before 权威定义 |
| `go tool trace` 文档与 "Profiling Go Programs" | 诊断方法 |
| `uber-go/guide`、`google/go-cmp` 文档 | 工程规范与测试实践 |
| Go 博客（`go.dev/blog`） | 版本特性背后的设计讨论 |

---

## 附录 E　术语表

| 术语 | 含义 |
|---|---|
| **G / M / P** | goroutine / OS 线程 / 调度上下文 |
| **GMP 调度** | Go 的协作 + 抢占式用户态调度模型 |
| **netpoller** | 基于 epoll/kqueue 的网络事件轮询器 |
| **work stealing** | 空闲 P 从其他 P 偷取一半任务 |
| **mcache/mcentral/mheap** | 三级内存分配结构 |
| **size class** | 小对象分配的固定大小档位 |
| **span** | 一组连续页，按 size class 切分 |
| **三色标记** | 白/灰/黑的并发标记算法 |
| **混合写屏障** | 插入 + 删除屏障，保证并发标记不漏对象 |
| **mutator assist** | 分配过快时，分配方协助标记 |
| **STW** | stop-the-world，暂停所有 goroutine |
| **Pacer** | 控制 GC 触发时机与标记速度的算法 |
| **itab** | 接口类型与具体类型的方法表 |
| **eface/iface** | 空接口/非空接口的内部表示 |
| **MVS** | 最小版本选择（Go 模块版本算法） |
| **PGO** | 基于运行时 profile 的编译优化 |
