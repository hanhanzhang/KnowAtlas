---
title: "Rust 语言与工程实践"
domain: tech/languages
type: deep-dive
tags: [rust, ownership, lifetimes, traits, async, cargo, testing, engineering]
source: "自整理；参考 The Rust Book / Rust Reference / Rust API Guidelines / Clippy 文档与 1.99 稳定版行为"
updated: 2026-10-29
---

# Rust 语言与工程实践

> **读者**：希望系统掌握 Rust 的语言规则与工程做法。
> **结构**：第零部分给出版本基线；第一部分讲语言内核（1–13 章）；第二部分讲工程实践（14–20 章）；附录 A–E 是速查与延伸阅读。
> **约定**：⚠️ 易错点｜💡 惯用法｜🧭 工程建议。

---

## 目录

**第零部分　版本基线与阅读指南**
- [0. 版本基线](#0-版本基线)
  - [0.1 三个容易混淆的版本概念](#01-三个容易混淆的版本概念)
  - [0.2 特性 → 最低版本](#02-特性--最低版本)
  - [0.3 MSRV 策略](#03-msrv-策略)

**第一部分　语言内核**
- [1. 工具链与第一个 crate](#1-工具链与第一个-crate)
  - [1.1 rustup 与工具链](#11-rustup-与工具链)
  - [1.2 `cargo new` 之后的目录](#12-cargo-new-之后的目录)
  - [1.3 edition 是什么](#13-edition-是什么)
  - [1.4 常用 cargo 命令](#14-常用-cargo-命令)
  - [1.5 编辑器与 rust-analyzer](#15-编辑器与-rust-analyzer)
  - [1.6 编译产物与环境变量](#16-编译产物与环境变量)
- [2. 类型与值语义](#2-类型与值语义)
  - [2.1 整数：溢出与转换](#21-整数溢出与转换)
  - [2.2 浮点](#22-浮点)
  - [2.3 bool、char、元组、数组、切片](#23-boolchar元组数组切片)
  - [2.4 字符串家族](#24-字符串家族)
  - [2.5 常量、静态与一次性初始化](#25-常量静态与一次性初始化)
  - [2.6 类型转换的四个层次](#26-类型转换的四个层次)
- [3. 表达式、控制流与模式匹配](#3-表达式控制流与模式匹配)
  - [3.1 一切皆表达式](#31-一切皆表达式)
  - [3.2 控制流](#32-控制流)
  - [3.3 模式匹配](#33-模式匹配)
  - [3.4 let-else、if let 链与 while let](#34-let-elseif-let-链与-while-let)
  - [3.5 属性与条件编译](#35-属性与条件编译)
- [4. 所有权、借用与生命周期](#4-所有权借用与生命周期)
  - [4.1 三条规则](#41-三条规则)
  - [4.2 move、Copy、Clone 与 partial move](#42-movecopyclone-与-partial-move)
  - [4.3 借用与 NLL](#43-借用与-nll)
  - [4.4 生命周期](#44-生命周期)
  - [4.5 变型与 `PhantomData`](#45-变型与-phantomdata)
  - [4.6 Drop 与 RAII](#46-drop-与-raii)
  - [4.7 `mem::replace` / `take` / `swap`](#47-memreplace--take--swap)
  - [4.8 `Deref` 与强制转换](#48-deref-与强制转换)
  - [4.9 内部可变性](#49-内部可变性)
  - [4.10 临时值的生命周期](#410-临时值的生命周期)
- [5. 结构体、枚举、trait 与泛型](#5-结构体枚举trait-与泛型)
  - [5.1 struct 三种形态与接收者](#51-struct-三种形态与接收者)
  - [5.2 构造、更新语法与 `Default`](#52-构造更新语法与-default)
  - [5.3 enum 与和类型](#53-enum-与和类型)
  - [5.4 trait：定义、默认方法、关联类型](#54-trait定义默认方法关联类型)
  - [5.5 泛型、const 泛型、GAT 与单态化](#55-泛型const-泛型gat-与单态化)
  - [5.6 `impl Trait`、`dyn Trait` 与对象安全](#56-impl-traitdyn-trait-与对象安全)
  - [5.7 RPITIT 与手写擦除层](#57-rpitit-与手写擦除层)
  - [5.8 孤儿规则、newtype 与转换 trait](#58-孤儿规则newtype-与转换-trait)
  - [5.9 derive 与常见 trait 语义](#59-derive-与常见-trait-语义)
  - [5.10 运算符重载与 `Deref` 滥用](#510-运算符重载与-deref-滥用)
- [6. 闭包与迭代器](#6-闭包与迭代器)
  - [6.1 闭包的三种 trait](#61-闭包的三种-trait)
  - [6.2 `move` 与捕获粒度](#62-move-与捕获粒度)
  - [6.3 闭包 vs 函数指针](#63-闭包-vs-函数指针)
  - [6.4 迭代器协议与惰性](#64-迭代器协议与惰性)
  - [6.5 适配器与消费者速查](#65-适配器与消费者速查)
  - [6.6 `collect` 与自定义迭代器](#66-collect-与自定义迭代器)
  - [6.7 性能：零成本的边界](#67-性能零成本的边界)
- [7. 错误处理](#7-错误处理)
  - [7.1 panic 与 `Result` 的边界](#71-panic-与-result-的边界)
  - [7.2 `?` 与 `From`](#72--与-from)
  - [7.3 `thiserror`：库层错误类型](#73-thiserror库层错误类型)
  - [7.4 `anyhow`：应用层错误](#74-anyhow应用层错误)
  - [7.5 错误边界与对外映射](#75-错误边界与对外映射)
  - [7.6 `main`、panic 与 `catch_unwind`](#76-mainpanic-与-catch_unwind)
  - [7.7 测试中的错误](#77-测试中的错误)
- [8. 智能指针与内部可变性](#8-智能指针与内部可变性)
  - [8.1 选型决策树](#81-选型决策树)
  - [8.2 `Box<T>`](#82-boxt)
  - [8.3 `Rc` / `Arc` / `Weak`](#83-rc--arc--weak)
  - [8.4 `Cell` 与 `RefCell`](#84-cell-与-refcell)
  - [8.5 `Cow`：借用优先，必要时克隆](#85-cow借用优先必要时克隆)
  - [8.6 `Pin` 与 `Unpin`](#86-pin-与-unpin)
  - [8.7 常见反模式](#87-常见反模式)
- [9. 同步并发](#9-同步并发)
  - [9.1 线程与作用域线程](#91-线程与作用域线程)
  - [9.2 `Send` / `Sync` 判定](#92-send--sync-判定)
  - [9.3 锁选型](#93-锁选型)
  - [9.4 原子与 `Ordering`](#94-原子与-ordering)
  - [9.5 channel 选型](#95-channel-选型)
  - [9.6 CPU 并行用 rayon](#96-cpu-并行用-rayon)
  - [9.7 死锁与排查](#97-死锁与排查)
- [10. 异步](#10-异步)
  - [10.1 `Future` 是什么](#101-future-是什么)
  - [10.2 执行器与 runtime](#102-执行器与-runtime)
  - [10.3 `spawn` 的约束](#103-spawn-的约束)
  - [10.4 `select!` 的三个陷阱](#104-select-的三个陷阱)
  - [10.5 取消安全（cancel safety）](#105-取消安全cancel-safety)
  - [10.6 阻塞操作的正确写法](#106-阻塞操作的正确写法)
  - [10.7 时间相关](#107-时间相关)
  - [10.8 `Stream` 与背压](#108-stream-与背压)
  - [10.9 结构化并发与 `abort`](#109-结构化并发与-abort)
  - [10.10 常见 `!Send` / 递归问题](#1010-常见-send--递归问题)
- [11. 宏与元编程](#11-宏与元编程)
  - [11.1 `macro_rules!` 基础](#111-macro_rules-基础)
  - [11.2 hygiene 与 `$crate`](#112-hygiene-与-crate)
  - [11.3 `tt` 吞入（tt munching）](#113-tt-吞入tt-munching)
  - [11.4 proc-macro 三类](#114-proc-macro-三类)
  - [11.5 调试与取舍](#115-调试与取舍)
- [12. `unsafe` 与 FFI](#12-unsafe-与-ffi)
  - [12.1 什么算 `unsafe`，边界在哪](#121-什么算-unsafe边界在哪)
  - [12.2 裸指针、`MaybeUninit`、`NonNull`](#122-裸指针maybeuninitnonnull)
  - [12.3 `repr`](#123-repr)
  - [12.4 FFI 最小示例](#124-ffi-最小示例)
  - [12.5 用工具兜底](#125-用工具兜底)
- [13. 标准库与生态速查](#13-标准库与生态速查)
  - [13.1 集合选型](#131-集合选型)
  - [13.2 `Option` / `Result` 组合子](#132-option--result-组合子)
  - [13.3 排序与比较](#133-排序与比较)
  - [13.4 格式化](#134-格式化)
  - [13.5 serde 惯用法](#135-serde-惯用法)
  - [13.6 常用 crate 地图](#136-常用-crate-地图)

**第二部分　工程实践**
- [14. Cargo 工程化](#14-cargo-工程化)
  - [14.1 单 crate 与 workspace](#141-单-crate-与-workspace)
  - [14.2 workspace 继承](#142-workspace-继承)
  - [14.3 features 设计](#143-features-设计)
  - [14.4 profiles](#144-profiles)
  - [14.5 `build.rs`](#145-buildrs)
  - [14.6 依赖治理](#146-依赖治理)
  - [14.7 版本、MSRV 与发布前检查](#147-版本msrv-与发布前检查)
- [15. 质量闸门与 CI](#15-质量闸门与-ci)
  - [15.1 格式化](#151-格式化)
  - [15.2 clippy 分层策略](#152-clippy-分层策略)
  - [15.3 `allow` vs `expect`](#153-allow-vs-expect)
  - [15.4 供应链与兼容性检查](#154-供应链与兼容性检查)
  - [15.5 最小 CI 矩阵](#155-最小-ci-矩阵)
  - [15.6 本地一键闸门](#156-本地一键闸门)
- [16. 测试体系](#16-测试体系)
  - [16.1 三类测试的分工](#161-三类测试的分工)
  - [16.2 断言与失败信息](#162-断言与失败信息)
  - [16.3 异步测试](#163-异步测试)
  - [16.4 参数化、属性测试、快照、打桩](#164-参数化属性测试快照打桩)
  - [16.5 覆盖率与 flaky 治理](#165-覆盖率与-flaky-治理)
- [17. 可观测性与运维](#17-可观测性与运维)
  - [17.1 `tracing` 的 span 与 field](#171-tracing-的-span-与-field)
  - [17.2 日志层级与采样](#172-日志层级与采样)
  - [17.3 指标、崩溃与优雅退出](#173-指标崩溃与优雅退出)
  - [17.4 配置分层](#174-配置分层)
- [18. API 设计与发布](#18-api-设计与发布)
  - [18.1 命名约定（RFC 430 摘要）](#181-命名约定rfc-430-摘要)
  - [18.2 四种常用模式](#182-四种常用模式)
  - [18.3 关键属性](#183-关键属性)
  - [18.4 公开 API 的取舍](#184-公开-api-的取舍)
  - [18.5 semver 与破坏性变更](#185-semver-与破坏性变更)
  - [18.6 发布流程](#186-发布流程)
- [19. 性能工程](#19-性能工程)
  - [19.1 先测量，再优化](#191-先测量再优化)
  - [19.2 分配的四个高频点](#192-分配的四个高频点)
  - [19.3 内联、LTO 与 PGO](#193-内联lto-与-pgo)
  - [19.4 剖析工具](#194-剖析工具)
  - [19.5 常见反模式](#195-常见反模式)
- [20. 项目骨架模板](#20-项目骨架模板)
  - [20.1 目录结构](#201-目录结构)
  - [20.2 模块切分经验](#202-模块切分经验)
  - [20.3 错误边界](#203-错误边界)
  - [20.4 CLI 与配置](#204-cli-与配置)
  - [20.5 启动与退出骨架](#205-启动与退出骨架)
  - [20.6 这份骨架的验收清单](#206-这份骨架的验收清单)
- [附录 A　编译错误速查](#附录-a编译错误速查)
- [附录 B　Rust 与 C++/Go/Java 对照](#附录-brust-与-cgojava-对照)
- [附录 C　Code Review 检查清单](#附录-ccode-review-检查清单)
- [附录 D　延伸阅读](#附录-d延伸阅读)
- [附录 E　术语表](#附录-e术语表)

---

# 第零部分　版本基线与阅读指南

## 0. 版本基线

### 0.1 三个容易混淆的版本概念

| 概念 | 含义 | 在哪里指定 |
|---|---|---|
| **工具链版本**（1.99.0 等） | 编译器与标准库的版本，决定"能用哪些特性" | `rust-toolchain.toml`、`rustup override` |
| **edition**（2015/2018/2021/2024） | 语法与语义的"方言版本"，同一编译器可编译多个 edition | `Cargo.toml` 的 `edition` |
| **MSRV**（最低支持版本） | 你的 crate 承诺支持的最低工具链 | `Cargo.toml` 的 `rust-version` |

三者独立：`edition = "2024"` 要求编译器 ≥ 1.85；而 `rust-version = "1.75"` 是给下游用户的承诺。

本文基线：**Rust 1.99**（2026-10-01 发布，补丁版 1.99.1）。下表给的是各特性的**首次稳定版本**，不代表只能用在该版本；新代码默认可以全部使用。

### 0.2 特性 → 最低版本

记不清"这个写法能不能用"时查这张表：

| 特性 | 最低版本 | 备注 |
|---|---|---|
| `thread::scope`（借用式作用域线程） | 1.63 | 无需 `Arc` 即可跨线程借用 |
| `let ... else` | 1.65 | 提前退出，替代嵌套 `match` |
| GAT（泛型关联类型） | 1.65 | `type Item<'a>` |
| `OnceLock` / `OnceCell` | 1.70 | 替代 `lazy_static!` |
| Cargo `[lints]` 表、`[workspace.lints]` | 1.74 | 集中配置 lint |
| RPITIT、trait 内 `async fn` | 1.75 | 免 `#[async_trait]`，但不能 `dyn` |
| `c"..."` C 字符串字面量 | 1.77 | FFI 传常量字符串 |
| `LazyLock` | 1.80 | 首次访问初始化 |
| `#[expect(lint)]` | 1.81 | 比 `#[allow]` 更可审计 |
| `resolver = "3"` | 1.84 | edition 2024 默认 |
| edition 2024（含 let-chains、`unsafe extern`、RPIT 捕获规则变更） | 1.85 / 1.88（let-chains） | 迁移见官方 Edition Guide |
| trait upcasting（`Arc<dyn Sub>` → `Arc<dyn Super>`） | 1.86 | 之前需要手写转发 |
| C 可变参数函数**定义**（`extern "C" fn f(args: ...)`） | 1.99 | 之前只能调用 C 的可变参数函数 |
| `size_of_val_raw` / `align_of_val_raw` / `Layout::for_value_raw` | 1.99 | 裸指针 + unsized 类型求布局 |
| `Vec::into_parts` / `Vec::from_parts` | 1.99 | 拆装 Vec 的原始部件 |
| `String::from_utf8_lossy_owned` | 1.99 | 宽松解码且复用原缓冲区 |

⚠️ 1.87–1.98 的逐版本变更请查官方 release notes；本表只列"影响正文写法"的里程碑。同样仍未稳定、看到博客要先确认 nightly 的：`gen` 块、`generic_const_exprs`、specialization、自定义分配器。

### 0.3 MSRV 策略

🧭 三种常见策略，选一种并写进 CI：

1. **滚动**：只支持最近 2–3 个版本。适合应用与内部库，维护成本最低。
2. **固定低水位**：`rust-version` 定在某个 LTS 式版本（如 1.75），CI 跑一个 MSRV job 保证不回退。适合开源库。
3. **不承诺**：只在 README 写"随 stable 滚动"。代价是下游可能装不上。

关键动作：`Cargo.toml` 写 `rust-version`，CI 用该版本单独跑一次 `cargo check`，否则 MSRV 一定会被无意破坏。

---

# 第一部分　语言内核

## 1. 工具链与第一个 crate

### 1.1 rustup 与工具链

```bash
rustup show                        # 当前目录生效的工具链与组件
rustup toolchain install 1.85.0    # 安装指定版本
rustup component add clippy rustfmt rust-analyzer
rustup override set 1.85.0          # 仅当前目录锁定版本（写进 .git 不生效）
```

仓库里固定版本用文件（可提交）：

```toml
# rust-toolchain.toml
[toolchain]
channel = "1.85.0"
components = ["clippy", "rustfmt"]
```

### 1.2 `cargo new` 之后的目录

```bash
cargo new demo            # bin crate
cargo new --lib mylib     # lib crate
```

```text
demo/
├── Cargo.toml
├── Cargo.lock        # 首次构建生成；bin 要提交，lib 通常不提交
└── src/
    └── main.rs       # lib 则是 lib.rs
```

- **package**：一个 `Cargo.toml` 描述的发布单元；**crate**：一次编译单元。
- package 名习惯用连字符（`my-app`），编译出的 crate 名把连字符换成下划线（`my_app`），`use my_app::...` 才是对的。

🧭 约定：`src/lib.rs` 只做三件事——声明模块、定义 crate 级属性、re-export 公共 API；实现细节留在子模块。

### 1.3 edition 是什么

edition 不是"新版本才有的库"，而是**语法规则的批次更新**。同一份代码在不同 edition 下可能编译结果不同（典型：数组 `for x in arr` 在 2015/2018 里迭代引用，2021 起迭代值）。

迁移用：

```bash
cargo fix --edition          # 自动改写大部分不兼容语法
# 手动把 Cargo.toml 的 edition 改成 "2024" 后再构建
```

edition 2024 的主要变化：let-chains 转为稳定语法、`unsafe extern` 与 `#[unsafe(no_mangle)]`、RPIT 默认捕获所有在作用域内的生命周期、尾表达式临时值的 drop 顺序调整、`gen` 成为保留字。

### 1.4 常用 cargo 命令

| 命令 | 用途 |
|---|---|
| `cargo check` | 只做类型检查，最快反馈 |
| `cargo build --release` | 优化构建，产物在 `target/release/` |
| `cargo run -- args` | 构建并运行 bin |
| `cargo test` | 单测 + 集成测试 + doctest |
| `cargo test -- --nocapture` | 显示测试中的 stdout |
| `cargo bench` | 运行基准（需 criterion 等） |
| `cargo fmt` / `cargo fmt --check` | 格式化 / 只检查 |
| `cargo clippy -- -D warnings` | lint，警告即错误 |
| `cargo add serde --features derive` | 添加依赖（自动选版本） |
| `cargo tree -d` | 找重复依赖（同一 crate 多版本） |
| `cargo update -p tokio --precise 1.40.0` | 精确升级单个依赖 |
| `cargo doc --open` | 生成本地文档 |
| `cargo expand`（外部） | 展开宏，调试 derive |
| `cargo nextest run`（外部） | 更快的测试运行器 |
| `cargo miri test`（组件） | 检测 unsafe 里的未定义行为 |

### 1.5 编辑器与 rust-analyzer

`rust-analyzer` 是语言服务器，提供补全、跳转、内联类型提示、`cargo check` 诊断。最小配置要点：

- 项目根有 `Cargo.toml`，多 crate 用 workspace 根打开；
- 打开 `rust-analyzer.check.command = "clippy"`，把 lint 提前到编辑器；
- 大仓库若索引慢，设 `rust-analyzer.cargo.features = "all"` 或指定要检查的 package。

### 1.6 编译产物与环境变量

| 变量 | 作用 |
|---|---|
| `RUST_BACKTRACE=1` / `full` | panic 时打印回溯 |
| `RUST_LOG=debug` | `tracing`/`env_logger` 的过滤（需程序支持） |
| `RUSTFLAGS="-C target-cpu=native"` | 追加编译参数（会破坏缓存复用，慎用） |
| `CARGO_TARGET_DIR` | 重定向构建目录，便于共享缓存 |

⚠️ `target/` 永远不要提交；CI 里用 `actions/cache` 缓存它能把构建时间压到 1/3 以下。

---

## 2. 类型与值语义

### 2.1 整数：溢出与转换

| 类型 | 位宽 | 默认 |
|---|---|---|
| `i8/i16/i32/i64/i128`、`isize` | 有符号 | `i32` |
| `u8/u16/u32/u64/u128`、`usize` | 无符号 | — |
| `usize` / `isize` | 指针宽度（32 或 64 位） | 下标、长度、容量专用 |

**溢出行为随构建模式不同，这是最容易上生产的事故源**：

```rust
let x: u8 = 255;
let y = x + 1;              // debug: panic "attempt to add with overflow"
                            // release: 回绕成 0（未定义行为？不是，是明确定义的回绕）
```

四种显式策略，生产代码应主动选择：

```rust
let a: u8 = 255;
a.checked_add(1);           // None
a.wrapping_add(1);          // 0
a.saturating_add(1);        // 255
a.overflowing_add(1);       // (0, true)
```

`as` 是**位截断**，不做检查也不 panic：

```rust
let big: u32 = 300;
let small = big as u8;      // 44（300 % 256）
let neg: i32 = -1;
let u = neg as u32;         // 4294967295（补码重解释）
let f = 3.99_f64 as i32;    // 3（向零截断）
```

💡 规则：**只在"取值范围已被证明安全"时用 `as`**（如 `u8 as u32` 扩展），其余一律 `TryFrom`：

```rust
use std::convert::TryFrom;
let n: u32 = 70000;
let s = u16::try_from(n)?;  // Err(TryFromIntError)
```

其他易错点：

- 字面量可加后缀与下划线：`1_000_000u64`、`0xff_u8`、`1e-3_f64`；
- `usize` 在 32 位平台会溢出：跨平台代码里长度计算优先 `u64`，只在索引处转 `usize`；
- 减法下溢同样 panic（debug）/回绕（release），计数递减用 `saturating_sub` 或先判空。

### 2.2 浮点

```rust
let a = 0.1_f64 + 0.2;      // 0.30000000000000004
assert!((a - 0.3).abs() < 1e-9);   // 永远不要用 == 比较浮点
```

- `f32`/`f64` 只实现 `PartialEq`/`PartialOrd`，**不实现 `Eq`/`Ord`/`Hash`**，因此不能直接做 `HashMap` key 或 `BTreeMap` key；需要时用 `ordered-float` 或转成定点/字符串。
- `NaN != NaN`，排序时 `partial_cmp` 返回 `None`，`sort_by(|a, b| a.partial_cmp(b).unwrap())` 会 panic。
- 需要全序排序用 `f64::total_cmp`：`v.sort_by(|a, b| a.total_cmp(b))`。

### 2.3 bool、char、元组、数组、切片

```rust
let t: (i32, &str, bool) = (1, "a", true);
let (x, _, z) = t;               // 解构；_ 丢弃

let arr: [u8; 3] = [1, 2, 3];    // 长度是类型的一部分，栈上
let slice: &[u8] = &arr[..];     // 视图：指针 + 长度
let mut v = arr.to_vec();        // 堆上的可增长版本
```

| 类型 | 语义 | 何时用 |
|---|---|---|
| `[T; N]` | 定长、值语义、`Copy`（若 T: Copy） | 编译期已知长度 |
| `&[T]` | 借用的一段连续内存 | 函数参数（比 `&Vec<T>` 通用） |
| `Vec<T>` | 拥有、可增长 | 需要动态长度 |
| `&mut [T]` | 可变借用段 | 原地修改、`split_at_mut` |

```rust
// 切片模式与工具
let s = &[1, 2, 3, 4, 5][..];
for chunk in s.chunks(2) { /* [1,2] [3,4] [5] */ }
for w in s.windows(2) { /* [1,2] [2,3] ... */ }

// 把一个 &mut [T] 拆成两个不重叠的可变借用
let (left, right) = s.split_at_mut(2);
```

💡 函数签名用 `&[T]` 而不是 `&Vec<T>`、用 `&str` 而不是 `&String`——调用方可以传切片，不必先造容器。

### 2.4 字符串家族

Rust 有六种"字符串"，各司其职：

| 类型 | 内容 | 典型场景 |
|---|---|---|
| `String` | UTF-8、拥有、可增长 | 构造/拼接/持有 |
| `&str` | UTF-8、借用 | 函数参数、字段借引用 |
| `&[u8]` | 原始字节 | 网络/文件二进制 |
| `OsString` / `&OsStr` | 平台原生字符串（可能非 UTF-8） | 路径之外的环境变量、参数 |
| `PathBuf` / `&Path` | 路径（内部是 `OsStr`） | 文件系统 |
| `CString` / `&CStr` | 以 NUL 结尾 | FFI |

```rust
let s = String::from("hello");
let slice: &str = &s;                 // 借用
let owned: String = slice.to_owned(); // 或 .to_string()
```

⚠️ **不能用下标取字符**：UTF-8 是变长编码，`s[0]` 会编译失败。按字符遍历用：

```rust
for ch in s.chars() {}                // Unicode 标量
for b in s.bytes() {}                 // 原始字节
for (i, ch) in s.char_indices() {}    // 字节偏移 + 字符
let first: &str = &s[0..1];           // ⚠️ 若 1 不在字符边界，运行期 panic
```

常用转换：

```rust
String::from_utf8(bytes)?                   // 严格：非 UTF-8 报错
String::from_utf8_lossy(&bytes)            // 宽松：非法字节替换为 U+FFFD
std::str::from_utf8(&bytes)?                // 借用版本
"42".parse::<i32>()?                        // 解析
format!("{name}: {value}")                  // 内联格式变量
```

🧭 读文件用 `std::fs::read_to_string`（要求 UTF-8）；处理用户输入/网络数据时先按字节读，再决定是否 `from_utf8_lossy`，否则一个坏字节就让整个流程失败。

### 2.5 常量、静态与一次性初始化

```rust
const MAX_RETRY: u32 = 3;                  // 编译期内联，无地址
static GREETING: &str = "hi";              // 有固定地址，'static 生命周期
static COUNTER: AtomicU64 = AtomicU64::new(0);
```

- `const` 可以出现在类型位置（`[u8; MAX_LEN]`），`static` 不行；
- `const fn` 允许在编译期求值，但只能调用 `const` 上下文允许的操作；
- `static mut` 基本被淘汰：需要全局可变状态时用 `AtomicXxx`、`Mutex<T>` 或一次性初始化。

```rust
use std::sync::{LazyLock, OnceLock};

static CONFIG: OnceLock<Config> = OnceLock::new();
fn config() -> &'static Config { CONFIG.get_or_init(load_config) }

// 需要"首次访问即初始化"，且初始化可内联成表达式
static TABLE: LazyLock<HashMap<&str, u8>> = LazyLock::new(|| HashMap::from([("a", 1)]));
```

区别：`OnceLock` 允许传入参数、失败可重试；`LazyLock` 是 `Deref` 到内部值的语法糖，初始化 panic 会毒化。

### 2.6 类型转换的四个层次

| 机制 | 触发方式 | 适用 |
|---|---|---|
| `as` | 显式 | 数值位转换、指针转换；不做检查 |
| `From`/`Into` | `.into()`、`?`、`impl From` | 无失败的转换 |
| `TryFrom`/`TryInto` | `.try_into()?` | 可能失败的转换 |
| `Deref` 强制转换 | 隐式 | 方法调用与函数实参上的自动解引用 |

隐式解引用链（`&String → &str`、`&Vec<T> → &[T]`、`&Box<T> → &T`）只在**方法接收者**和**函数实参**处发生，不会发生在泛型约束匹配上：

```rust
fn takes_str(s: &str) {}
let s = String::from("x");
takes_str(&s);        // ✅ Deref coercion

fn takes_generic<T: AsRef<str>>(t: T) {}
takes_generic(&s);    // &String 实现了 AsRef<str>，也可
```

💡 API 设计上，**参数用 `&str`/`&[T]`/`impl AsRef<Path>`，返回用具体拥有类型**，能让调用方少写转换，同时不丢失类型信息。

---

## 3. 表达式、控制流与模式匹配

### 3.1 一切皆表达式

块、`if`、`match`、`loop` 都有值；函数体最后一个表达式（无分号）是返回值。

```rust
let level = if retry > 3 { "high" } else { "low" };

let parsed = {
    let trimmed = input.trim();
    trimmed.parse::<i32>()          // 块的值
};

let n = loop {
    if ready() { break 42; }        // break 可带值
};
```

分号决定语义：`expr;` 丢弃值（要求类型为 `()`），`expr` 作为块的值。

⚠️ 常见编译错误 `expected (), found i32` 几乎都是"多写/少写了一个分号"。

`!`（never 类型）可与任意类型统一，因此 `panic!`、`return`、`break` 能出现在任何分支位置。

### 3.2 控制流

```rust
for item in &items { }                 // 借用迭代
for item in items { }                  // 移动迭代（之后 items 不可用）
while let Some(x) = iter.next() { }
'outer: for i in 0..3 {                // 标签跳出外层
    for j in 0..3 {
        if i * j > 4 { break 'outer; }
    }
}
```

| 写法 | 说明 |
|---|---|
| `0..n` / `0..=n` | 左闭右开 / 闭区间；`n` 为 `usize` 时注意 `n-1` 下溢 |
| `for (i, v) in v.iter().enumerate()` | 带下标，优先于手写 `i += 1` |
| `while cond` | 条件在循环前后各求值一次的场景注意副作用 |

🧭 能用迭代器链表达就少写 `while`：前者更容易保证边界正确，且通常编译结果等价。

### 3.3 模式匹配

```rust
match msg {
    Msg::Quit => return,
    Msg::Move { x, y } if x > 0 => move_right(y),      // 守卫
    Msg::Move { x, .. } => move_left(x),               // 忽略其余字段
    Msg::Write(text) => write(&text),
    other => log_unknown(&other),                      // 兜底绑定
}
```

进阶形态：

```rust
// or-pattern：多个模式共享分支（绑定必须一致）
match code {
    401 | 403 => "auth",
    500..=599 => "server",
    _ => "other",
}

// 切片模式
match &items[..] {
    [] => "empty",
    [one] => "single",
    [first, rest @ ..] => "many",
}

// @ 绑定：既保留整体又解构
match age {
    n @ 0..=17 => println!("minor {n}"),
    n => println!("adult {n}"),
}

// ref / ref mut：匹配时借用而非移动（match ergonomics 多数情况可省略）
match &mut config {
    Config { timeout: ref mut t, .. } => *t += 1,
}
```

⚠️ **穷尽性**：`match` 不覆盖所有变体就编译失败，这是 Rust 相对 `switch` 的核心优势。公开枚举的新增变体会破坏下游的穷尽匹配，因此**库的公开枚举建议加 `#[non_exhaustive]`**，并在下游用 `_ => ...` 兜底。

match ergonomics（默认绑定模式）在 edition 2024 有调整：对「引用 + 非引用模式」混写更严格。迁移时若遇到 "binding mode" 报错，显式加 `&`/`ref` 即可。

### 3.4 let-else、if let 链与 while let

```rust
// let-else：不匹配就退出当前作用域（必须有 else 且发散）
let Some(config) = load() else { return Err(Error::NoConfig) };

// if let：单分支解构
if let Some(id) = user.id { fetch(id).await; }

// while let：消费式循环
while let Some(item) = queue.pop() { handle(item); }

// let-chains（edition 2024）：多个模式条件串联，避免嵌套
if let Some(user) = session.user()
    && let Some(token) = user.token()
    && token.is_valid()
{
    grant_access();
}
```

💡 何时用哪个：**能提前返回用 `let-else`；只关心一个分支用 `if let`；需要同时满足多个条件用 let-chains；分支多且要返回值用 `match`。**

### 3.5 属性与条件编译

```rust
#[derive(Debug, Clone)]                  // 派生 trait
#[cfg(test)] mod tests;                  // 条件编译：仅测试
#[cfg(feature = "metrics")] fn emit();   // cargo feature 开关
#[cfg_attr(windows, path = "win.rs")]    // 条件属性
#[allow(dead_code)]                      // 豁免 lint（可审计性差）
#[expect(unused_variables, reason = "占位")]  // 预期会有该 lint
#[must_use]                              // 忽略返回值时告警
#[deprecated(since = "1.2.0", note = "用 new_api 替代")]
#[non_exhaustive]                        // 外部 crate 不能穷尽构造/匹配
```

常见 `cfg` 谓词：`test`、`debug_assertions`、`feature = "x"`、`target_os = "linux"`、`target_arch = "aarch64"`、`unix`/`windows`。

🧭 组合条件用 `all(...)` / `any(...)` / `not(...)`：

```rust
#[cfg(all(unix, not(target_os = "macos")))]
```

⚠️ `#[cfg]` 作用于**下一个 item**，写错位置会让整块代码被静默移除；加 `#[cfg(test)]` 的模块若写成 `mod tests;` 而没有对应文件，报错信息会指向 cfg 而不是文件缺失。

---

## 4. 所有权、借用与生命周期

### 4.1 三条规则

1. **每个值有且只有一个所有者**；
2. **离开作用域时，所有者负责释放（drop）**；
3. **所有权可以转移（move），但不能被复制**——除非类型实现了 `Copy`。

```rust
let s1 = String::from("hi");
let s2 = s1;                    // 所有权转移，s1 失效
// println!("{s1}");            // ❌ E0382: borrow of moved value

let n = 5;                      // 整数实现了 Copy
let m = n;                      // 复制，n 仍可用
```

这三条决定了 Rust 不需要 GC 也能保证内存安全：释放点唯一且静态可推导。

### 4.2 move、Copy、Clone 与 partial move

| 类型 | 赋值/传参行为 | 要求 |
|---|---|---|
| 非 `Copy` | 移动，原变量失效 | — |
| `Copy` | 按位复制，两边都可用 | 所有字段都是 `Copy`，且无 `Drop` |
| `Clone` | 显式调用 `.clone()` 深拷贝 | 手动或 derive 实现 |

⚠️ `Copy` 与 `Drop` 互斥：一个类型要么"能随便复制"，要么"有清理逻辑"，不能兼得（否则会二次释放）。

**partial move**：结构体可以只移动其中一个字段，之后整体不可用，但未移动的字段仍可用：

```rust
struct User { name: String, age: u8 }

let u = User { name: "a".into(), age: 1 };
let name = u.name;              // 移动 name 字段
println!("{}", u.age);          // ✅ 未移动的字段仍可读
// println!("{:?}", u);         // ❌ 整体已被部分移动
```

需要"取出字段但保留整体"时，用 `std::mem::take` 或 `replace`（见 4.7）。

### 4.3 借用与 NLL

> 同一时刻，要么有任意多个 `&T`，要么有唯一一个 `&mut T`。

```rust
let mut v = vec![1, 2, 3];
let first = &v[0];              // 不可变借用
println!("{first}");
v.push(4);                      // ✅ 上一行的借用到这里已经结束
```

**NLL（non-lexical lifetimes）**：借用的有效期算到"最后一次使用"，而不是作用域末尾。

```rust
let mut s = String::from("a");
let r = &s;
println!("{r}");
s.push('b');                    // ✅ NLL 允许
```

借用分裂（同一结构的不同字段可以同时可变借用，前提是编译器能看出来）：

```rust
struct Pair { a: i32, b: i32 }
let mut p = Pair { a: 1, b: 2 };
let Pair { a, b } = &mut p;     // 两个独立可变借用
*a += 1; *b += 1;
```

切片同理用 `split_at_mut`；容器同时要"读某元素 + 改另一元素"时，先取索引再操作，避免同时持两个借用。

### 4.4 生命周期

生命周期不是"让值活更久"的语法，而是**给编译器描述引用之间关系**的标注。

**省略规则（三条）**：

1. 每个输入引用各自获得一个独立生命周期参数；
2. 只有一个输入引用时，输出引用沿用它；
3. 有 `&self`/`&mut self` 时，输出引用沿用 `self` 的生命周期。

```rust
fn first_word(s: &str) -> &str            // 规则 2：输出 = 输入
fn get(&self) -> &str                     // 规则 3：输出 = self
fn pick<'a>(x: &'a str, y: &'a str) -> &'a str   // 两个输入：必须显式统一
```

`'static` 有两种含义，别混：

```rust
let s: &'static str = "literal";          // ① 数据活到程序结束
fn spawn<F: Fn() + 'static>(f: F) {}      // ② 不含短生命周期借用（可跨线程/长期持有）
```

**结构体持引用**：必须显式标注，且结构体实例不能超过被借用数据的寿命。

```rust
struct Parser<'a> { input: &'a str }
impl<'a> Parser<'a> {
    fn rest(&self) -> &'a str { &self.input[1..] }   // 注意返回 'a，不是 &self
}
```

**HRTB（高阶生命周期）**：约束"对任意生命周期都成立"，常见于回调。

```rust
fn with_tmp<F>(f: F) where F: for<'a> Fn(&'a str) -> usize { f("x") }
```

**子类型关系**：`'long: 'short`（长寿命可当短的用），反之不行。

### 4.5 变型与 `PhantomData`

| 位置 | 变型 | 含义 |
|---|---|---|
| `&'a T` | 协变 | 长寿命可当短寿命用 |
| `&'a mut T` | 对 T 不变、对 'a 协变 | 可变引用不允许替换类型 |
| `fn(T)` | 逆变 | 少见，回调参数位置 |
| `Cell<T>`、`&mut T` 内的 T | 不变 | 保证内部可变性安全 |

`PhantomData` 的三种用途：

```rust
use std::marker::PhantomData;

struct Id<T> { raw: u64, _marker: PhantomData<T> }          // ① 类型参数未出现在字段里
struct Borrowed<'a> { ptr: *const u8, _marker: PhantomData<&'a u8> }  // ② 声明借用关系
struct NotSend { _marker: PhantomData<*const ()> }          // ③ 主动放弃 Send/Sync
```

⚠️ 手写 `unsafe impl Send` 前先想清楚：绝大多数"需要手动实现"的场景，其实是结构里混进了 `Rc`/裸指针，应该改数据结构而不是给编译器打补丁。

### 4.6 Drop 与 RAII

```rust
struct Guard<'a> { log: &'a str }
impl Drop for Guard<'_> {
    fn drop(&mut self) { println!("exit {}", self.log); }
}
```

- 字段按**声明顺序** drop，结构体本身在字段之后；
- 提前释放用 `drop(value)`，不要依赖作用域结束；
- `ManuallyDrop<T>` 跳过自动 drop（FFI、手写容器时用）；
- `std::mem::forget` 会泄漏（合法但通常意味着设计有问题）。

🧭 RAII 是 Rust 工程化的核心模式：锁守卫、文件句柄、临时目录、取消注册、指标计时都靠 `Drop` 自动完成，不需要 try/finally。

```rust
// 典型：作用域内自动恢复状态
struct Restore<'a> { slot: &'a mut bool }
impl Drop for Restore<'_> { fn drop(&mut self) { *self.slot = false; } }
```

### 4.7 `mem::replace` / `take` / `swap`

borrow checker 不允许"从借用里搬走东西"，这时用这三个函数：

```rust
use std::mem;

let mut slot: Option<String> = Some("x".into());

// 取出并留下 None
let taken = slot.take();

// 换成新值，返回旧值
let old = mem::replace(&mut slot, Some("y".into()));

// 交换两个可变引用的内容
let (mut a, mut b) = (1, 2);
mem::swap(&mut a, &mut b);
```

💡 在 `&mut self` 方法里需要"消费掉自己某个字段但又不能 move"时，`take` 是最常见的解法。

### 4.8 `Deref` 与强制转换

`Deref` 让自定义类型获得"自动解引用"能力，也是智能指针的基础：

```rust
use std::ops::Deref;
struct Wrapper(String);
impl Deref for Wrapper {
    type Target = str;
    fn deref(&self) -> &str { &self.0 }
}
let w = Wrapper("hi".into());
println!("{}", w.len());     // 自动 &Wrapper -> &str
```

强制转换的规则：`&T` → `&U`，当 `T: Deref<Target = U>`；可链式（`&Box<String>` → `&str`）。**只发生在方法接收者和函数实参**，不影响泛型匹配与 `==` 比较。

⚠️ 不要为了"少写一次 `.0`"而给业务类型实现 `Deref`：它会让方法解析变得隐式、破坏封装。`Deref` 只该用于**智能指针/包装器**语义。

### 4.9 内部可变性

| 类型 | 线程 | 检查时机 | 用途 |
|---|---|---|---|
| `Cell<T>` | 单线程 | 无（Copy 进出） | 小值计数器、标志位 |
| `RefCell<T>` | 单线程 | 运行期 | 图结构、缓存、需要 `&self` 里改状态 |
| `Mutex<T>` | 多线程 | 阻塞 | 跨线程独占 |
| `RwLock<T>` | 多线程 | 阻塞 | 读多写少 |
| `OnceLock` / `LazyLock` | 多线程 | 无 | 一次性初始化 |
| `AtomicXxx` | 多线程 | 无锁 | 计数、标志、简单状态机 |

```rust
use std::cell::RefCell;
let cache = RefCell::new(Vec::new());
cache.borrow_mut().push(1);       // ⚠️ 同一作用域内再 borrow() 会 panic
```

⚠️ `RefCell` 的借用检查在运行期，**panic 而不是编译错误**：`borrow_mut()` 未释放时再 `borrow()` 就会崩。调试时看到 "already borrowed" 基本都能定位到嵌套调用。

🧭 优先用"把状态作为参数传递"的设计，`RefCell` 是逃逸手段而非首选。

### 4.10 临时值的生命周期

```rust
let s = String::from("a b");
let parts: Vec<&str> = s.split(' ').collect();   // ✅ 借用源 s 仍活
// let p = s.split(' ').next().unwrap();         // ✅ 借用 s
```

典型的 E0716：

```rust
// ❌ 临时 String 在语句结束即释放，引用悬空
// let x: &str = String::from("tmp").as_str();
let owned = String::from("tmp");                  // ✅ 先绑定
let x: &str = owned.as_str();
```

`match`/`if let`/`while let` 的临时值寿命有特殊规则（scrutinee 的临时值活到整个 match 结束），这也是"为什么有些链式调用必须拆成两行"的原因。edition 2024 调整了尾表达式临时值的 drop 时机，升级后个别代码的借用期会变短——遇到报错先看 Edition Guide 的 "temporary scope" 一节。

---

## 5. 结构体、枚举、trait 与泛型

### 5.1 struct 三种形态与接收者

```rust
struct User { name: String, age: u8 }   // 命名字段
struct Meters(f64);                      // tuple struct / newtype
struct Marker;                           // unit struct
```

| 接收者 | 语义 | 何时用 |
|---|---|---|
| `self` | 消费，方法结束后值被 drop | 转换、销毁（`into_*`） |
| `&self` | 只读借用 | 查询（`as_*`、`len`） |
| `&mut self` | 可变借用 | 原地修改 |
| `self: Arc<Self>` | 任意 self 类型（需 `arbitrary_self_types` 之外的标准写法） | 共享所有权对象的方法 |

```rust
impl User {
    fn new(name: impl Into<String>) -> Self { Self { name: name.into(), age: 0 } }
    fn name(&self) -> &str { &self.name }
    fn rename(&mut self, n: impl Into<String>) { self.name = n.into(); }
    fn into_name(self) -> String { self.name }
}
```

### 5.2 构造、更新语法与 `Default`

```rust
let u = User { name: "a".into(), age: 1 };
let u2 = User { age: 2, ..u };        // 结构体更新语法：剩余字段从 u 移动
```

- 构造器习惯叫 `new`，可失败的用 `try_new`；带参数的用 `with_capacity`/`with_timeout`；
- 字段多时用 **builder 模式**（见 18 章）：`Config::builder().timeout(3).build()?`；
- `#[derive(Default)]` 给每个字段取默认值，枚举可用 `#[default]` 标记变体：

```rust
#[derive(Default)]
enum Level { #[default] Low, High }
```

### 5.3 enum 与和类型

```rust
enum Event {
    Quit,                                   // unit
    Move { x: i32, y: i32 },                // struct
    Write(String),                           // tuple
}
```

- enum 的布局是 tag + 最大载荷；`Option<&T>`、`Option<Box<T>>`、`Option<NonZeroU32>` 等可零开销优化（niche optimization）；
- 需要与 C 交互或控制布局时加 `#[repr(C)]` / `#[repr(u8)]`；
- 公开 enum 想让下游"能加变体而不断链"就加 `#[non_exhaustive]`。

🧭 设计经验：**用 enum 表达"互斥状态"，用 struct 表达"组合属性"**。把状态机写成 enum，把非法状态排除在类型之外（typestate），能消灭大量运行期断言。

### 5.4 trait：定义、默认方法、关联类型

```rust
pub trait Repository {
    type Id;                                       // 关联类型：每个实现固定
    const NAME: &'static str = "repo";             // 关联常量（可带默认）

    fn get(&self, id: Self::Id) -> Option<Record>; // 必须实现
    fn exists(&self, id: Self::Id) -> bool {       // 默认方法
        self.get(id).is_some()
    }
}
```

- **supertrait**：`trait Store: Send + Sync` 要求实现者同时满足这些约束；
- **关联类型 vs 泛型参数**：一个实现只能有一种 `Id` 用关联类型；同一类型要支持多种参数用泛型 `trait Convert<T>`；
- trait 方法可以有 `where Self: Sized` 来把某个方法排除出 `dyn` 兼容范围。

### 5.5 泛型、const 泛型、GAT 与单态化

```rust
fn first<T: Clone>(v: &[T]) -> Option<T> { v.first().cloned() }

struct Cache<K, V> { map: std::collections::HashMap<K, V> }

impl<K: Eq + std::hash::Hash, V> Cache<K, V> {
    fn get(&self, k: &K) -> Option<&V> { self.map.get(k) }
}
```

复杂约束用 `where`，可读性更好：

```rust
fn merge<K, V>(a: Map<K, V>, b: Map<K, V>) -> Map<K, V>
where
    K: Eq + std::hash::Hash + Clone,
    V: Clone,
{ /* ... */ }
```

**const 泛型**（长度/大小作为类型参数）：

```rust
fn sum<const N: usize>(xs: [u32; N]) -> u32 { xs.iter().sum() }
let total = sum([1, 2, 3]);          // N 自动推断为 3
```

**GAT**（关联类型带生命周期，写"借贷式迭代器"必需）：

```rust
trait Lender {
    type Item<'a> where Self: 'a;
    fn next(&mut self) -> Option<Self::Item<'_>>;
}
```

**单态化代价**：每个泛型实例都会生成一份机器码。滥用泛型会导致编译变慢、二进制膨胀；对"只在少数几处使用"的抽象，`dyn` 反而更省。

### 5.6 `impl Trait`、`dyn Trait` 与对象安全

```rust
// 参数位置（APIT）：等价于泛型，静态分发
fn notify(h: impl Handler) {}

// 返回位置（RPIT）：返回"某个实现了该 trait 的具体类型"
fn handlers(&self) -> impl Iterator<Item = &Handler> { self.handlers.iter() }

// trait object：动态分发、可异构容器
let hs: Vec<Box<dyn Handler>> = vec![Box::new(A), Box::new(B)];
```

| 维度 | `impl Trait`（返回） | `dyn Trait` |
|---|---|---|
| 分发 | 编译期单态化 | 运行期 vtable |
| 异构集合 | 不支持 | 支持 |
| 二进制体积 | 每实例一份 | 一份 |
| 能否跨 crate 扩展 | 不能（返回类型不透明） | 能（新增实现即可） |

**对象安全（dyn 兼容）完整清单**——不满足任一条就不能 `dyn`：

1. 方法不能有泛型参数；
2. 方法不能返回 `Self`（`where Self: Sized` 的方法除外）；
3. 不能有关联常量（关联类型可以）；
4. 方法的接收者必须是 `self`/`&self`/`&mut self`/`Box<Self>` 等可分派形式；
5. 不能有 `where Self: Sized` 之外的 `Self` 约束。

`dyn` 还要求指针包装：`&dyn T`、`Box<dyn T>`、`Arc<dyn T>`（裸 `dyn T` 是 unsized，不能作为值）。

trait upcasting（1.86+）允许把 `Arc<dyn Sub>` 当 `Arc<dyn Super>` 用，之前需要手写桥接方法。

### 5.7 RPITIT 与手写擦除层

Rust 1.75 起 trait 方法可以直接返回 `impl Future`（RPITIT），不必再用 `#[async_trait]` 的装箱：

```rust
pub trait Handler: Send + Sync + 'static {
    fn handle(&self, req: Request) -> impl std::future::Future<Output = Response> + Send;
}

impl Handler for MyHandler {
    async fn handle(&self, req: Request) -> Response { /* ... */ }
}
```

代价与限制：

- **RPITIT 的 trait 不是 dyn 兼容的**；
- `async fn` in trait 的返回值不能额外加 auto-trait 约束（比如要求 `Send` 时，需要手写 `-> impl Future + Send` 形式）；
- 需要异构集合时，必须自己写一层擦除适配：

```rust
use std::future::Future;
use std::pin::Pin;

pub type BoxFuture<'a, T> = Pin<Box<dyn Future<Output = T> + Send + 'a>>;

pub trait DynHandler: Send + Sync + 'static {
    fn handle(&self, req: Request) -> BoxFuture<'_, Response>;
}

impl<T: Handler> DynHandler for T {
    fn handle(&self, req: Request) -> BoxFuture<'_, Response> {
        Box::pin(Handler::handle(self, req))     // 把 RPITIT 的 future 装箱
    }
}
```

💡 判断标准：**默认 RPITIT（零分配）；只有确实要把不同类型塞进同一个容器/字段时，才付出 `Box::pin` 的代价。**

### 5.8 孤儿规则、newtype 与转换 trait

孤儿规则：`impl Trait for Type` 必须在 `Trait` 或 `Type` **至少一个属于当前 crate** 时才合法；外部 trait 作用于外部类型时需要"本地类型出现在 trait 的类型参数里"。

newtype 是绕过孤儿规则、同时获得类型安全的标准手段：

```rust
pub struct UserId(u64);

impl From<u64> for UserId { fn from(v: u64) -> Self { Self(v) } }
impl UserId { pub fn get(self) -> u64 { self.0 } }

let id: UserId = 42.into();
```

- `From`/`Into`：无失败转换，`Into` 由 `From` 自动获得，**只实现 `From` 就够了**；
- `TryFrom`：可能失败，`?` 可直接用（需错误类型可转换）；
- `FromIterator`：让类型支持 `.collect()`；
- `ToString`：实现了 `Display` 就自动获得，不要手写。

```rust
impl FromIterator<Permission> for Permissions {
    fn from_iter<I: IntoIterator<Item = Permission>>(iter: I) -> Self {
        iter.into_iter().fold(Self::EMPTY, |acc, p| acc | p)
    }
}
```

### 5.9 derive 与常见 trait 语义

| 派生 | 语义 | 注意 |
|---|---|---|
| `Debug` | 调试打印 `{:?}` | 含密钥/令牌的类型**不要** derive |
| `Clone` | 显式深拷贝 | 与 `Copy` 二选一（有 `Drop` 不能 `Copy`） |
| `PartialEq` / `Eq` | 相等 | 浮点只能 `PartialEq` |
| `PartialOrd` / `Ord` | 排序 | `Ord` 要求全序，冲突时 `cmp` 必须与 `eq` 一致 |
| `Hash` | 哈希 | 必须与 `Eq` 保持一致，否则 `HashMap` 出错 |
| `Default` | 零值/默认值 | 枚举用 `#[default]` 标记变体 |
| `Serialize`/`Deserialize` | serde | 见 13.5 |

⚠️ 手写 `Hash` 时忘了让 `Eq` 与之一致，是 `HashMap` "插入后查不到"的经典原因。

### 5.10 运算符重载与 `Deref` 滥用

实现标准 trait 即获得语法：

```rust
impl std::ops::Add for Meters {
    type Output = Meters;
    fn add(self, rhs: Meters) -> Meters { Meters(self.0 + rhs.0) }
}

impl std::fmt::Display for Meters {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        write!(f, "{}m", self.0)
    }
}
```

⚠️ 不要给业务类型实现 `Deref` 来"继承方法"，这会破坏封装、让方法解析隐式；`Deref` 只用于智能指针。同理，不要滥用 `Index`/`Add` 去表达"业务运算"，语义会漂移。

---

## 6. 闭包与迭代器

这两块是 Rust 日常代码的 70%，也是从"会语法"到"写得顺手"的分水岭。

### 6.1 闭包的三种 trait

闭包本质是**编译器生成的匿名结构体**，实现了以下至多三种 trait：

| trait | 调用方式 | 捕获方式 | 典型场景 |
|---|---|---|---|
| `FnOnce` | `f()` 只能调用一次 | 可能移动捕获值 | 消费型回调（`spawn(move \|\| ...)`） |
| `FnMut` | `f()` 可多次，需 `&mut` | 可变借用捕获 | 累加器、`for_each` |
| `Fn` | `f()` 可多次，只需 `&` | 不可变借用 | 谓词、映射函数 |

关系是包含的：**实现 `Fn` 的一定实现 `FnMut` 和 `FnOnce`；实现 `FnMut` 的一定实现 `FnOnce`。** 所以参数写 `FnOnce` 最宽松，写 `Fn` 最严格。

```rust
fn call_once<F: FnOnce() -> i32>(f: F) -> i32 { f() }
fn call_many<F: FnMut() -> i32>(mut f: F) -> i32 { f() + f() }
fn call_ref<F: Fn() -> i32>(f: F) -> i32 { f() }
```

**调用语义决定推断**：编译器根据闭包体内怎么用捕获值，推断它最少实现了哪个 trait。

```rust
let s = String::from("x");
let by_ref = || s.len();          // 只读 -> Fn
let by_move = move || s;          // 返回 s -> FnOnce（且 s 被移动进闭包）
```

### 6.2 `move` 与捕获粒度

```rust
let name = String::from("a");
let f = move || println!("{name}");     // 所有权移入闭包，name 之后不可用
```

edition 2021 起闭包按**字段级**捕获（disjoint capture），因此下面两个闭包可以同时存在：

```rust
struct S { a: String, b: String }
let mut s = S { a: "1".into(), b: "2".into() };
let fa = || &s.a;
let fb = || &s.b;      // 2021+ 合法：捕获的是不同字段
```

返回闭包必须用装箱或 `impl`：

```rust
fn make_adder(n: i32) -> impl Fn(i32) -> i32 {
    move |x| x + n
}

fn make_handler(kind: Kind) -> Box<dyn Fn(&str) -> String> {
    match kind {
        Kind::Upper => Box::new(|s: &str| s.to_uppercase()),
        Kind::Lower => Box::new(|s: &str| s.to_lowercase()),
    }
}
```

### 6.3 闭包 vs 函数指针

```rust
let f: fn(i32) -> i32 = |x| x + 1;      // 无捕获 -> 可强制转成 fn 指针
let g: fn(i32) -> i32 = double;          // 普通函数也能当指针
```

- 无捕获闭包可以传给需要 `fn` 的地方（C 回调、`sort_by_key` 的旧写法）；
- 有捕获就必须用泛型 `F` 或 `dyn Fn`；
- `dyn Fn` 有间接调用与装箱成本，热路径优先泛型。

💡 参数写 `impl Fn(...)` 而不是 `fn(...)`，除非确实需要函数指针（如存进 `struct` 字段且不想引入泛型参数）。

### 6.4 迭代器协议与惰性

```rust
pub trait Iterator {
    type Item;
    fn next(&mut self) -> Option<Self::Item>;
    // 其余几十个方法都有默认实现
}
```

**迭代器是惰性的**：适配器只构造状态，直到遇到消费者才真正执行。

```rust
let v = vec![1, 2, 3, 4];
let lazy = v.iter().map(|x| { println!("map {x}"); x * 2 });   // 什么也不打印
let sum: i32 = lazy.sum();                                      // 这时才逐项执行
```

### 6.5 适配器与消费者速查

| 适配器 | 作用 |
|---|---|
| `map` / `filter` / `filter_map` | 转换 / 谓词 / 转换+过滤 |
| `flat_map` / `flatten` | 展平 |
| `fold` / `scan` | 带状态的归约 |
| `zip` / `chain` / `enumerate` | 组合 / 拼接 / 带上标 |
| `take` / `take_while` / `skip` / `skip_while` | 截断 |
| `peekable` | 预读一个元素 |
| `rev` / `step_by` | 逆向 / 步进（需 `DoubleEndedIterator`） |
| `cloned` / `copied` | 把 `&T` 变成 `T` |
| `inspect` | 调试时插入副作用（不改变值） |
| `collect` / `sum` / `product` / `count` | 消费者 |
| `find` / `position` / `any` / `all` | 查找与判定 |
| `min_by_key` / `max_by_key` | 取极值 |

```rust
let words: Vec<String> = text
    .lines()
    .filter(|l| !l.trim().is_empty())
    .filter_map(|l| l.split_once(':'))
    .map(|(k, v)| format!("{k}={}", v.trim()))
    .collect();
```

### 6.6 `collect` 与自定义迭代器

`collect` 的目标类型决定实现，因此必须能推断：

```rust
let v: Vec<i32> = iter.collect();
let s: String = chars.collect();
let m: HashMap<_, _> = pairs.collect();
let r: Result<Vec<_>, _> = results.collect();   // 遇到第一个 Err 就短路
```

自定义迭代器只需实现 `next`：

```rust
struct Countdown(u32);
impl Iterator for Countdown {
    type Item = u32;
    fn next(&mut self) -> Option<u32> {
        if self.0 == 0 { return None; }
        self.0 -= 1;
        Some(self.0)
    }
}
```

让自定义容器支持 `for`：

```rust
impl<'a> IntoIterator for &'a Store {
    type Item = &'a Record;
    type IntoIter = std::slice::Iter<'a, Record>;
    fn into_iter(self) -> Self::IntoIter { self.items.iter() }
}
```

### 6.7 性能：零成本的边界

- 迭代器链通常与手写 `for` **编译成同样的机器码**（内联 + 消除中间结构）；
- 会破坏优化的写法：链中夹 `Box<dyn Iterator>`、频繁 `collect::<Vec<_>>()` 中转、在热循环里 `format!`；
- `.cloned()`/`.copied()` 优于 `.map(|x| x.clone())`；
- 需要返回"惰性序列"时，返回 `impl Iterator<Item = &T> + '_`，比返回 `Vec` 少一次分配，但要注意与 `&self` 的借用关系。

```rust
// 好：一次分配，无中间 Vec
let total: u64 = records.iter().filter(|r| r.ok).map(|r| r.size).sum();

// 差：两次分配
let sizes: Vec<u64> = records.iter().filter(|r| r.ok).map(|r| r.size).collect();
let total: u64 = sizes.iter().sum();
```

---

## 7. 错误处理

### 7.1 panic 与 `Result` 的边界

| 场景 | 建议 |
|---|---|
| 可预期的失败（IO、解析、网络、校验） | `Result` |
| 程序 bug（越界、违反不变量、未实现分支） | `panic!` / `unreachable!` |
| 库的公开 API | 一律 `Result`，不 panic |
| 测试 / 原型 / 示例 | `unwrap()`/`expect()` 可接受 |
| 启动期配置非法 | 早 panic（fail fast），但要打印清晰信息 |

```rust
let n: i32 = input.parse().expect("input must be an integer");
```

⚠️ `expect` 的字符串要写"为什么这里不可能失败"或"用户该怎么做"，不要写 "failed"。

### 7.2 `?` 与 `From`

```rust
use std::fs;

fn load(path: &str) -> Result<Config, AppError> {
    let text = fs::read_to_string(path)?;    // io::Error -> AppError（有 From impl）
    let cfg = toml::from_str(&text)?;        // toml::de::Error -> AppError
    Ok(cfg)
}
```

规则：`?` 对 `Err(e)` 执行 `From::from(e)`；若没有对应 `From`，可先 `map_err`。

`?` 也能用于 `Option`（在返回 `Option` 的函数里）：

```rust
fn first_char(s: &str) -> Option<char> { Some(s.chars().next()?) }
```

### 7.3 `thiserror`：库层错误类型

```rust
use thiserror::Error;

#[derive(Debug, Error)]
pub enum StoreError {
    #[error("record {id} not found")]
    NotFound { id: u64 },

    #[error("io error")]
    Io(#[from] std::io::Error),              // 自动生成 From

    #[error("decode failed")]
    Decode(#[source] serde_json::Error),     // 保留错误链但不自动 From

    #[error(transparent)]                    // 直接透传底层 Display
    Other(#[from] anyhow::Error),
}
```

要点：

- `#[from]` 生成 `From`，让 `?` 直接可用；一个枚举里同类型只能有一个 `#[from]`；
- `#[source]` 只标注错误链，不生成 `From`；
- `#[error(transparent)]` 让外层错误完全等价于内层（包装型错误用）；
- 错误类型应实现 `Debug + Display + Error`，`thiserror` 全部代劳。

### 7.4 `anyhow`：应用层错误

```rust
use anyhow::{Context, Result};

fn run() -> Result<()> {
    let cfg = load("app.toml").context("加载配置失败")?;
    start(cfg).with_context(|| format!("启动失败，pid={}", std::process::id()))?;
    Ok(())
}
```

- `anyhow::Error` 是类型擦除的错误，适合**应用/二进制**，不适合库的公开 API（下游无法匹配具体错误）；
- 打印完整错误链用 `{e:#}` 或 `{:?}`；只打印最外层用 `{e}`；
- 需要向下转型时用 `e.downcast_ref::<MyError>()`。

🧭 **分层原则**：`thiserror` 定义边界（库的公开错误、模块间契约），`anyhow` 在应用内自由传播。一个项目里两者共存是正常的。

### 7.5 错误边界与对外映射

```rust
pub fn handle(req: Request) -> Response {
    match service::process(&req) {
        Ok(v) => Response::ok(v),
        Err(e) => {
            tracing::warn!(error = %e, "request failed");     // 内部细节进日志
            Response::error(e.public_code())                  // 外部只见稳定错误码
        }
    }
}
```

原则：**内部错误可以很细，对外错误必须稳定且不泄漏实现细节**（路径、SQL、堆栈）。

### 7.6 `main`、panic 与 `catch_unwind`

```rust
fn main() -> Result<(), Box<dyn std::error::Error>> {   // 返回 Err 会打印 Debug 并以非零码退出
    run()?;
    Ok(())
}
```

- `panic = "abort"`（见 14.4）能减小体积、加快启动，但会失去 `catch_unwind` 与 `Drop` 保证；
- `catch_unwind` 只能捕获 `UnwindSafe` 的 panic，用于"单个请求出错不影响整体"（如 Web 框架、测试框架）；
- 自定义 `panic::set_hook` 可以把 panic 写进日志/上报，但**不要在里面做复杂分配或加锁**。

```rust
let result = std::panic::catch_unwind(|| risky());
if result.is_err() { /* 记录并继续 */ }
```

### 7.7 测试中的错误

```rust
#[test]
fn rejects_negative() {
    let err = parse("-1").unwrap_err();
    assert!(matches!(err, ParseError::Negative { .. }));
}

#[test]
#[should_panic(expected = "input must be")]
fn panics_on_garbage() {
    parse("abc").unwrap();
}
```

⚠️ 不要用 `assert_eq!(format!("{err}"), "...")` 断言错误文案——文案会变；用 `matches!` 匹配变体，必要时单独测 `Display`。

---

## 8. 智能指针与内部可变性

### 8.1 选型决策树

```text
需要放到堆上 / 递归类型 / trait object ?
  └── Box<T>

需要多处共享所有权 ?
  ├── 单线程 → Rc<T>（配 Cell/RefCell）
  └── 多线程 → Arc<T>（配 Mutex/RwLock/原子）

需要打破循环引用 ? → Weak<T>
需要"借用优先、必要时克隆" ? → Cow<'a, T>
需要固定地址（自引用 / async） ? → Pin<P>
需要 &self 里改状态 ?
  ├── 单线程、Copy 小值 → Cell<T>
  ├── 单线程、任意类型 → RefCell<T>
  └── 多线程 → Mutex/RwLock/AtomicXxx
```

### 8.2 `Box<T>`

```rust
enum List { Cons(i32, Box<List>), Nil }        // 递归类型必须有间接层

let f: Box<dyn Fn(i32) -> i32> = Box::new(|x| x + 1);
let big = Box::new([0u8; 1 << 20]);            // 大对象避免栈拷贝
```

`Box` 是唯一开销为"一次分配 + 一次间接"的智能指针；`Box<[T]>`（胖指针）比 `Vec<T>` 少 8 字节且不可增长。

### 8.3 `Rc` / `Arc` / `Weak`

```rust
use std::sync::{Arc, Weak};

let shared = Arc::new(vec![1, 2, 3]);
let clone = Arc::clone(&shared);        // 引用计数 +1，不复制数据
```

⚠️ 写 `shared.clone()` 也能编译，但语义上可能被误读成深拷贝；`Arc::clone(&shared)` 更明确。开启 `clippy::clone_on_ref_ptr` 可强制这个习惯。

**循环引用与 `Weak`**：

```rust
struct Node {
    next: Option<Arc<Node>>,
    parent: Option<Weak<Node>>,          // 反向边用 Weak
}
// 使用前先升级
if let Some(p) = self.parent.as_ref().and_then(Weak::upgrade) { /* ... */ }
```

`Arc<T>` 要求 `T: Send + Sync` 才能跨线程共享；`Arc<RefCell<T>>` 不是 `Sync`，编译器会拦住你。**`Arc<Mutex<T>>` 不是默认答案**，见 8.7。

### 8.4 `Cell` 与 `RefCell`

```rust
use std::cell::{Cell, RefCell};

let counter = Cell::new(0);
counter.set(counter.get() + 1);          // 无借用检查，靠 Copy 进出

let cache = RefCell::new(Vec::new());
cache.borrow_mut().push(1);              // 运行期借用
```

| | 适用 | 风险 |
|---|---|---|
| `Cell<T>` | `T: Copy` 的小值、标志位、计数 | 不能取内部引用 |
| `RefCell<T>` | 任意类型、需要 `&mut` 语义 | 借用冲突 → **运行期 panic** |
| `OnceCell` | 只写一次 | 初始化失败需自己处理 |

💡 需要"多线程版的 RefCell"用 `RwLock`；需要"多线程版的 Cell<Copy>"用 `AtomicXxx`。

### 8.5 `Cow`：借用优先，必要时克隆

```rust
use std::borrow::Cow;

fn normalize(s: &str) -> Cow<'_, str> {
    if s.contains(' ') { Cow::Owned(s.replace(' ', "_")) } else { Cow::Borrowed(s) }
}
```

用于"大多数输入不需要修改"的函数（解析、清洗、脱敏），避免无谓分配。`Cow<T>` 要求 `T: ToOwned`。

### 8.6 `Pin` 与 `Unpin`

`Pin<P>` 保证被指向的值**不会被移动**，因为自引用结构（async 状态机是典型）一旦移动，内部指针就会悬空。

```rust
use std::pin::Pin;
use std::future::Future;

fn run(fut: Pin<Box<dyn Future<Output = ()>>>) { /* ... */ }
let fut = Box::pin(async { /* ... */ });      // 装箱 + 固定，一步到位
```

- 绝大多数类型是 `Unpin` 的（可以随便移动），`Pin<&mut T>` 对它们没有额外约束；
- `async` 块/函数生成的 future 默认 `!Unpin`，所以跨 `await` 之后不能再移动；
- 手写 `Future` 时若字段含自引用，必须用 `unsafe` 保证不被移动（`PhantomPinned` 可主动放弃 `Unpin`）。

⚠️ 常见误解：`Pin` 不等于"装箱"。`Box::pin(x)` 是"装箱并固定"；`Pin<&mut T>` 只是固定，没有分配。

### 8.7 常见反模式

| 反模式 | 问题 | 替代 |
|---|---|---|
| 到处 `Arc<Mutex<T>>` | 锁粒度粗、易死锁、读并发差 | 缩小锁范围、`RwLock`、把状态作为参数传递 |
| 无脑 `.clone()` 消 borrow 报错 | 掩盖设计问题、性能损失 | 调整所有权、用引用或 `Cow` |
| `Rc<RefCell<T>>` 建图 | 环导致泄漏、运行期 panic | `Arc` + `Weak`，或索引式（arena + id） |
| `Arc<Vec<u8>>` 反复切片 | 无法释放前半段内存 | `bytes::Bytes` 或 `Arc<[u8]>` + 偏移 |

---

## 9. 同步并发

### 9.1 线程与作用域线程

```rust
let data = vec![1, 2, 3];
std::thread::scope(|s| {
    s.spawn(|| println!("{}", data.len()));   // 借用，无需 Arc
});
```

`thread::scope`（1.63+）保证所有子线程在作用域结束前 join，因此可以安全借用栈上数据。适合"分片并行处理一个切片"。

### 9.2 `Send` / `Sync` 判定

| 类型 | `Send` | `Sync` | 原因 |
|---|---|---|---|
| `T`（普通数据） | ✅ | ✅ | 无共享可变状态 |
| `Rc<T>` | ❌ | ❌ | 非原子引用计数 |
| `Arc<T>` | ✅（T: Send+Sync） | ✅ | 原子计数 |
| `RefCell<T>` | ✅ | ❌ | 非线程安全借用检查 |
| `Mutex<T>` | ✅（T: Send） | ✅ | 内部同步 |
| `Cell<T>` | ✅ | ❌ | 非同步 |
| `*const T` / `*mut T` | ❌ | ❌ | 裸指针默认不跨线程 |

记忆法：**`Send` = 能安全移动到别的线程；`Sync` = 能被多个线程同时引用（`&T: Send`）。**

### 9.3 锁选型

| 场景 | 选择 |
|---|---|
| 极短临界区、无 `await` | `std::sync::Mutex`（更快） |
| 临界区内有 `await` | `tokio::sync::Mutex`（可跨 await 持锁） |
| 读多写少 | `RwLock`（注意写者饥饿） |
| 高争用、需要公平/性能 | `parking_lot::Mutex`（更快、无中毒） |
| 只需计数/标志 | `AtomicXxx` |

⚠️ **永远不要跨 `.await` 持有 `std::sync::MutexGuard`**：它可能阻塞 executor 线程，且守卫不是 `Send`，会直接编译失败。

```rust
// 错误：guard 跨 await（编译报 future is not Send）
// let g = self.state.lock().unwrap();
// do_async().await;

// 正确：缩小作用域
{
    let mut g = self.state.lock().unwrap();
    g.count += 1;
}
do_async().await;
```

### 9.4 原子与 `Ordering`

```rust
use std::sync::atomic::{AtomicUsize, Ordering};

static HITS: AtomicUsize = AtomicUsize::new(0);
HITS.fetch_add(1, Ordering::Relaxed);          // 只统计，不用于同步
```

| `Ordering` | 保证 | 何时用 |
|---|---|---|
| `Relaxed` | 原子性，无顺序保证 | 计数器、指标 |
| `Acquire` / `Release` | 配对建立 happens-before | 发布/读取共享数据（`load(Acquire)` / `store(Release)`） |
| `AcqRel` | 读改写同时具备两者 | `compare_exchange` |
| `SeqCst` | 全局总顺序 | 默认安全选择，性能略差 |

🧭 不确定时先用 `SeqCst`；只有基准测试证明有瓶颈时才降级，并且必须写清不变量。

### 9.5 channel 选型

| 类型 | 语义 | 场景 |
|---|---|---|
| `std::sync::mpsc` | 多生产者单消费者 | 线程池任务投递 |
| `crossbeam::channel` | 多生产者多消费者、select | 高性能线程间通信 |
| `tokio::sync::mpsc` | 异步多生产者单消费者，有界/无界 | actor 模型、任务队列 |
| `tokio::sync::oneshot` | 一次性回执 | 请求-响应配对 |
| `tokio::sync::watch` | 只保留最新值 | 配置热更新、状态广播 |
| `tokio::sync::broadcast` | 多消费者广播 | 事件总线 |

🧭 **有界队列是背压的实现方式**：无界队列在生产者快于消费者时会吃光内存。生产环境默认选有界，并明确"满了怎么办"（阻塞、丢弃、拒绝）。

### 9.6 CPU 并行用 rayon

```rust
use rayon::prelude::*;
let sum: u64 = data.par_iter().map(|x| x * 2).sum();
```

区分两类并发：

- **IO 并发**（等待网络/磁盘）→ `tokio`，任务是 future；
- **CPU 并行**（计算密集）→ `rayon`，线程池按核数切分。

⚠️ 在 `tokio` 里跑 CPU 密集任务要用 `spawn_blocking`，否则会饿死 executor（见 10.6）。

### 9.7 死锁与排查

死锁的三种典型：**锁顺序不一致**、**同线程重复加锁**、**持锁等待另一个持锁者**。

- 约定全局锁顺序，并在文档里写清；
- 临界区内不发网络请求、不做阻塞 IO；
- 多个锁需要同时持有时，用 `Mutex::try_lock` 或重构为单一锁；
- 排查手段：`tokio-console`、`gdb` 抓栈、给锁加 tracing span 记录等待时长。

---

## 10. 异步

### 10.1 `Future` 是什么

`Future` 是一个可以"被轮询"的状态机：

```rust
pub trait Future {
    type Output;
    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output>;
}
```

- `Poll::Pending` 表示"现在没好"，返回前必须通过 `cx.waker()` 注册唤醒，否则任务永远不会被再次调度；
- `Poll::Ready(v)` 表示完成，之后不应再 poll（再次 poll 是逻辑错误，可能 panic）。

最小手写实现（理解用，实际很少写）：

```rust
struct Ready(Option<u32>);
impl Future for Ready {
    type Output = u32;
    fn poll(mut self: Pin<&mut Self>, _cx: &mut Context<'_>) -> Poll<u32> {
        Poll::Ready(self.0.take().expect("polled after completion"))
    }
}
```

### 10.2 执行器与 runtime

`async fn` 只生成状态机，**必须有人驱动**。tokio 的常用形态：

```rust
#[tokio::main]                       // 等价于手动建 runtime + block_on
async fn main() { /* ... */ }
```

| 形态 | 用途 |
|---|---|
| `#[tokio::main]` | 应用入口 |
| `#[tokio::test]` | 异步测试（默认 current_thread） |
| `Builder::new_multi_thread()` | 自定义 worker 数、线程名、阻塞池 |
| `Runtime::block_on` | 在同步代码里驱动异步（不能嵌套在 runtime 内） |

### 10.3 `spawn` 的约束

```rust
tokio::spawn(async move { ... });     // 要求 future: Send + 'static
```

- `'static`：不能捕获短生命周期的借用 → 用 `Arc` 或 `move` 拥有数据；
- `Send`：future 在任意线程间迁移 → 跨 `await` 不能持有 `Rc`、`RefCell` 借用、`std::sync::MutexGuard`。

这就是异步代码里 `Arc` 满天飞的真正原因：不是"共享状态需要"，而是"spawn 要求拥有 + 可跨线程"。

### 10.4 `select!` 的三个陷阱

```rust
tokio::select! {
    _ = token.cancelled() => return Err(Cancelled),
    res = work() => res,
}
```

1. **未选中的分支 future 会被 drop**：如果它已产生副作用（写了半条消息），状态就坏了 → 优先选"取消安全"的 future；
2. **默认随机轮询**：需要确定性优先级时加 `biased;`，并把高优先级分支放前面；
3. **没有 `else` 分支时全 Pending 会永久挂起**：需要"都不就绪就做别的"时加 `else => { ... }`。

### 10.5 取消安全（cancel safety）

一个 future 被 drop 时"什么都没发生"才算取消安全。经验清单：

| API | 取消安全 | 说明 |
|---|---|---|
| `mpsc::Receiver::recv` | ✅ | 未收到消息时 drop 无副作用 |
| `AsyncReadExt::read` | ✅ | 未读到数据不消耗 |
| `AsyncWriteExt::write_all` | ❌ | 可能写了半截 |
| `sleep` / `timeout` | ✅ | — |
| 自定义 `async fn` | 取决于内部 | 有副作用的位置要单独设计 |

🧭 需要"要么全做完、要么不做"的操作，包一层 `tokio::spawn` 拿到 `JoinHandle`，或改用能回滚的协议。

### 10.6 阻塞操作的正确写法

```rust
// CPU 密集 / 同步 IO
let out = tokio::task::spawn_blocking(move || heavy_compute(input)).await??;

// 已经在 runtime 线程里且必须阻塞（少用）
tokio::task::block_in_place(|| std::fs::read(path));
```

⚠️ `block_in_place` 只在 multi-thread runtime 可用；在 current_thread runtime 里调用会 panic。真正该做的是把阻塞代码移出异步路径。

### 10.7 时间相关

```rust
tokio::time::sleep(Duration::from_millis(100)).await;
let r = tokio::time::timeout(Duration::from_secs(5), fut).await;   // Err(Elapsed)

let mut tick = tokio::time::interval(Duration::from_secs(1));
tick.tick().await;                       // 第一次立即返回
```

测试里用 `tokio::time::pause()` + `advance()`，让超时测试瞬间完成，不要真的 `sleep`。

### 10.8 `Stream` 与背压

`Stream` 是异步版迭代器（`poll_next`），来自 `futures` 或 `tokio-stream`。

```rust
use tokio_stream::{wrappers::ReceiverStream, StreamExt};

let mut stream = ReceiverStream::new(rx);
while let Some(item) = stream.next().await { handle(item).await?; }
```

背压来自**有界 channel**：生产者 `send().await` 在队列满时挂起，天然限速。把无界 channel 当队列用，等于放弃背压。

### 10.9 结构化并发与 `abort`

```rust
let handle = tokio::spawn(worker());
// 需要一个"drop 即取消"的句柄
let handle = AbortOnDrop(handle);        // 自行实现的 RAII 包装
```

- 裸 `tokio::spawn` 的任务在句柄 drop 后继续运行（"孤儿任务"），退出时可能丢数据；
- 需要"父任务结束，子任务必然结束"时，用 `JoinSet`、`TaskTracker`（tokio-util）或自己包一层 `Drop` 中 `abort()`；
- 优雅退出：`CancellationToken` 广播 → 各任务检查 → 等待 `JoinSet` 清空 → 超时强制 abort。

```rust
let mut set = tokio::task::JoinSet::new();
set.spawn(async { 1 });
while let Some(res) = set.join_next().await { res??; }
```

### 10.10 常见 `!Send` / 递归问题

**`future is not Send` 的三大来源**：

1. 跨 `await` 持有 `std::sync::MutexGuard` → 缩小作用域或换 tokio 锁；
2. 跨 `await` 持有 `RefCell` 的 `Ref`/`RefMut` → 提前 drop；
3. 捕获了 `Rc` / `Rc<RefCell<T>>` → 换 `Arc`。

排查方法：把 `async fn` 里每个 `let` 的作用域收紧，或临时用 `tokio::task::spawn` 编译错误定位到具体行。

**异步递归**：`async fn f() { f().await }` 无法计算大小，必须装箱：

```rust
fn f() -> BoxFuture<'static, ()> { Box::pin(async { f().await }) }
```

---

## 11. 宏与元编程

### 11.1 `macro_rules!` 基础

```rust
#[macro_export]
macro_rules! hashmap {
    () => { std::collections::HashMap::new() };
    ($($k:expr => $v:expr),+ $(,)?) => {{
        let mut m = std::collections::HashMap::new();
        $( m.insert($k, $v); )+
        m
    }};
}

let m = hashmap! { "a" => 1, "b" => 2 };
```

- `$name:expr` 是片段匹配器，常用：`expr`、`ty`、`pat`、`ident`、`path`、`item`、`tt`、`literal`、`block`；
- `$( ... ),+` 表示"一到多个，用逗号分隔"；`$(,)?` 允许尾随逗号；
- `$( ... )*` / `$( ... )?` 分别表示零或多个 / 零或一个。

⚠️ 宏不是函数：展开发生在语法层，**参数会被求值多次**（如果写了两次），且错误信息指向调用点而非定义处。

### 11.2 hygiene 与 `$crate`

宏默认是"卫生的"：宏内部引入的变量不会与调用处的同名变量冲突。

```rust
#[macro_export]
macro_rules! log_scope {
    ($name:ident, $body:block) => {{
        let $name = "scope";          // 不会污染调用处的 $name
        $body
    }};
}
```

**跨 crate 引用自身条目必须用 `$crate`**，否则在下游展开时会找不到路径：

```rust
#[macro_export]
macro_rules! make_counter {
    () => { $crate::Counter::new() };
}
```

### 11.3 `tt` 吞入（tt munching）

处理"可变数量的复杂语法"时的经典写法：

```rust
macro_rules! count {
    () => { 0usize };
    ($head:tt $($rest:tt)*) => { 1usize + count!($($rest)*) };
}
let n = count!(a b c d);      // 4
```

### 11.4 proc-macro 三类

| 类型 | 声明 | 用途 |
|---|---|---|
| derive | `#[proc_macro_derive(Name, attributes(attr))]` | 为 `#[derive(Name)]` 生成 impl |
| 属性宏 | `#[proc_macro_attribute]` | 改写被标注的 item（如 `#[instrument]`） |
| 函数式宏 | `#[proc_macro]` | 像函数一样调用（如 `sql!(...)`） |

必须在独立 crate 里，且 `Cargo.toml` 声明：

```toml
[lib]
proc-macro = true
```

最小 derive 示例（依赖 `syn` + `quote`）：

```rust
use proc_macro::TokenStream;
use quote::quote;
use syn::{parse_macro_input, DeriveInput};

#[proc_macro_derive(Describe)]
pub fn describe(input: TokenStream) -> TokenStream {
    let ast = parse_macro_input!(input as DeriveInput);
    let name = &ast.ident;
    quote! {
        impl #name {
            pub fn describe() -> &'static str { stringify!(#name) }
        }
    }
    .into()
}
```

### 11.5 调试与取舍

- `cargo expand`（外部工具）把宏展开成普通代码，是排查 derive 问题的第一手段；
- `cargo rustc -- -Zunpretty=expanded` 是 std 侧替代（需 nightly）；
- 🧭 **能用泛型/函数解决就不写宏**：宏的错误信息差、IDE 支持弱、无法单独测试；宏适合"必须操作语法"的场景（derive、DSL、变参构造）。

---

## 12. `unsafe` 与 FFI

### 12.1 什么算 `unsafe`，边界在哪

`unsafe` 块只是"把编译器的一部分检查交给你"，它不会关掉借用检查，也不会让代码变快。

合法的 `unsafe` 场景只有四类：**调用外部 C 代码、解引用裸指针、访问 `union` 字段、实现 `Send`/`Sync`/`Drop` 里的不变量**。

🧭 规范：每个 `unsafe` 块上方写 `// SAFETY:` 注释，说明"为什么这里的先决条件成立"。这是代码审查能看懂的唯一依据。

```rust
// SAFETY: ptr 来自 Vec::as_ptr，长度与 vec 一致，且未与其他可变借用共存
let slice = unsafe { std::slice::from_raw_parts(ptr, len) };
```

### 12.2 裸指针、`MaybeUninit`、`NonNull`

```rust
let mut x = 5;
let p: *mut i32 = &mut x;
unsafe { *p += 1; }                 // 解引用裸指针

use std::mem::MaybeUninit;
let mut buf: MaybeUninit<[u8; 1024]> = MaybeUninit::uninit();   // 避免先零初始化
let buf = unsafe { buf.assume_init() };                          // 仅当已完整写入
```

`NonNull<T>` 表示"非空指针"，可用于优化 `Option`（零开销），也常用于链表/树实现。

⚠️ 三个高频 UB：`assume_init` 在未完整初始化时调用、把 `&mut` 别名成两个 `&mut`、`transmute` 尺寸/对齐不匹配的类型。

### 12.3 `repr`

```rust
#[repr(C)]           // 与 C 一致的字段布局
#[repr(transparent)] // 单字段包装，ABI 与内部类型一致（newtype 传 FFI 用）
#[repr(u8)]          // 枚举判别值类型
```

⚠️ 默认 `repr(Rust)` 的字段顺序**不保证**，不能作为二进制协议或 FFI 布局。

### 12.4 FFI 最小示例

```rust
use std::ffi::{c_char, CStr, CString};

unsafe extern "C" {                      // edition 2024 要求 unsafe extern
    fn strlen(s: *const c_char) -> usize;
}

fn c_len(s: &str) -> usize {
    let c = CString::new(s).expect("no interior NUL");
    // SAFETY: c 在本调用期间有效，且以 NUL 结尾
    unsafe { strlen(c.as_ptr()) }
}
```

- 传入 Rust 侧：`*const T` / `*mut T` / `extern "C" fn`；
- 字符串用 `CString`/`CStr`，不要直接传 `&str`；
- 回调要 `extern "C"`，且不能 panic 穿过 FFI 边界（用 `catch_unwind` 包住）；
- 绑定生成：`bindgen`（C 头文件 → Rust），导出：`cbindgen`（Rust → C 头文件）。

### 12.5 用工具兜底

| 工具 | 能查什么 |
|---|---|
| `cargo miri test` | 未定义行为：越界、悬垂、数据竞争、对齐错误 |
| `-Zsanitizer=address/thread`（nightly） | 与 C 交互的内存/线程错误 |
| `loom` | 并发算法的交错穷举（验证 `Ordering` 正确性） |
| `cargo geiger` | 统计依赖树里 `unsafe` 的用量 |

🧭 含 `unsafe` 的 crate 应在 CI 里至少跑一次 `miri`（可限定到相关测试子集），否则"看起来能跑"的 UB 会在优化后变成随机崩溃。

---

## 13. 标准库与生态速查

### 13.1 集合选型

| 需求 | 类型 | 复杂度 |
|---|---|---|
| 顺序存储、尾部增删 | `Vec<T>` | 尾部 O(1)，中间 O(n) |
| 双端队列 | `VecDeque<T>` | 两端 O(1) |
| 键值查找（无序） | `HashMap<K, V>` | 平均 O(1) |
| 有序键值 / 范围查询 | `BTreeMap<K, V>` | O(log n) |
| 去重集合 | `HashSet` / `BTreeSet` | 同上 |
| 优先队列 | `BinaryHeap<T>` | push/pop O(log n) |
| 需要稳定顺序的小集合 | `Vec` + 线性查找 | 常数因子更低 |

```rust
use std::collections::HashMap;

// Entry API：避免两次查找
let counter = map.entry(key).or_insert(0);
*counter += 1;
```

🧭 `HashMap` 默认用 SipHash（抗哈希洪水攻击但偏慢）；内部热路径可换 `ahash`/`rustc-hash`，对外暴露的 API 保持默认。

### 13.2 `Option` / `Result` 组合子

| 方法 | 作用 |
|---|---|
| `map` / `map_or` / `map_or_else` | 映射内部值 |
| `and_then` | 链式可能失败的转换 |
| `ok_or` / `ok_or_else` | `Option` → `Result` |
| `filter` | 条件保留 |
| `unwrap_or` / `unwrap_or_default` / `unwrap_or_else` | 提供兜底值（后者可延迟计算） |
| `is_some_and` / `is_none_or` | 带谓词判断 |
| `take` / `replace` / `get_or_insert` | 原地取出/替换 |
| `transpose` | `Option<Result<T, E>>` ↔ `Result<Option<T>, E>` |
| `Result::inspect` / `inspect_err` | 观测而不改变值 |
| `and_then` + `?` | 优先于层层 `match` |

```rust
let port = env::var("PORT").ok()
    .and_then(|s| s.parse::<u16>().ok())
    .unwrap_or(8080);
```

### 13.3 排序与比较

```rust
v.sort();                              // 稳定排序，可能分配
v.sort_unstable();                     // 不稳定但更快，无分配
v.sort_by_key(|x| x.priority);         // 按键排序
v.sort_by(|a, b| a.score.total_cmp(&b.score));   // 浮点安全比较
v.dedup();                             // 相邻去重（先排序）
v.binary_search(&target);              // 要求已排序
```

⚠️ `sort_by` 的比较函数必须满足全序，否则 1.81+ 起会 panic（此前是未定义行为）。

### 13.4 格式化

```rust
println!("{name} = {value:>8.2}");      // 内联变量 + 宽度/精度
println!("{:#?}", config);              // 多行 Debug
println!("{:?}", err);                  // Debug 可能不带错误链，用 anyhow 时改用 {:#}
write!(&mut buf, "{:.3}", ratio)?;      // 写入任意实现 fmt::Write 的目标
```

- `Display`（`{}`）给最终用户看；`Debug`（`{:?}`）给开发者看；
- 自定义错误打印链式信息时，实现 `Display` 里逐个遍历 `source()`。

### 13.5 serde 惯用法

```rust
#[derive(Serialize, Deserialize)]
#[serde(rename_all = "camelCase", deny_unknown_fields)]
pub struct Config {
    pub listen_addr: String,
    #[serde(default = "default_timeout")]
    pub timeout_ms: u64,
    #[serde(skip_serializing_if = "Option::is_none")]
    pub tls: Option<TlsConfig>,
}

fn default_timeout() -> u64 { 3_000 }
```

四种枚举表示：

```rust
#[serde(tag = "type")]                       // 内部标签 {"type":"a", ...}
#[serde(tag = "t", content = "c")]           // 相邻标签
#[serde(untagged)]                           // 按形状尝试
#[serde(rename_all = "snake_case")]          // 配合上面任意一种
```

**演进兼容策略**（对外协议必须考虑）：

| 变更 | 是否兼容 | 做法 |
|---|---|---|
| 加字段 | ✅（接收方需 `#[serde(default)]`） | 老数据缺字段用默认值 |
| 删字段 | ❌（发送方可能仍在发） | 先标记 `#[serde(skip_serializing)]` 保留解析 |
| 加枚举变体 | ⚠️ | 接收方加 `#[serde(other)]` 兜底或 `untagged` |
| 改字段类型 | ❌ | 新增字段 + 迁移逻辑 |
| 重命名字段 | ❌ | 同时接受新旧名（自定义 `Deserialize` 或用别名） |

`#[serde(alias = "old_name")]` 是重命名时最省事的过渡手段。

### 13.6 常用 crate 地图

| 领域 | crate | 说明 |
|---|---|---|
| 异步运行时 | `tokio` | 事实标准；`smol`/`async-std` 更轻 |
| 序列化 | `serde` + `serde_json`/`toml`/`bincode` | — |
| 错误 | `thiserror`（库）/ `anyhow`（应用） | — |
| 日志/追踪 | `tracing` + `tracing-subscriber` | 结构化 span |
| CLI | `clap`（derive） | 参数解析 |
| HTTP 客户端 | `reqwest` | — |
| HTTP 服务 | `axum` / `actix-web` | — |
| 并行计算 | `rayon` | CPU 并行 |
| 时间 | `chrono` / `time` | 需要时区用 `chrono-tz` |
| 随机 | `rand` | — |
| 正则 | `regex` | — |
| 测试增强 | `rstest`、`proptest`、`insta`、`mockall` | 见 16 章 |

🧭 选 crate 的三条硬指标：最近一年有提交、下载量在同类别前列、`cargo deny` 无高危 license/漏洞。

---

# 第二部分　工程实践

## 14. Cargo 工程化

### 14.1 单 crate 与 workspace

单 crate 适合小工具；一旦出现"库 + 二进制 + 共享类型"就上 workspace：

```toml
# 根 Cargo.toml
[workspace]
members = ["crates/core", "crates/api", "crates/cli"]
resolver = "2"
```

```text
repo/
├── Cargo.toml          # 只有 [workspace]
├── Cargo.lock          # 全 workspace 共享
└── crates/
    ├── core/           # 纯逻辑，无 IO
    ├── api/            # 依赖 core，暴露接口
    └── cli/            # bin，依赖 api
```

分层原则：**core 不依赖任何基础设施**（便于测试与复用），依赖方向单向向下。

### 14.2 workspace 继承

```toml
# 根
[workspace.dependencies]
serde = { version = "1", features = ["derive"] }
tokio = { version = "1", features = ["rt-multi-thread", "macros"] }

[workspace.package]
edition = "2021"
rust-version = "1.75"
license = "MIT"

[workspace.lints.rust]
unsafe_code = "warn"

[workspace.lints.clippy]
all = "warn"
```

```toml
# 子 crate
[dependencies]
serde.workspace = true
tokio.workspace = true

[package]
edition.workspace = true

[lints]
workspace = true
```

好处：版本与 lint 规则单点维护，避免 20 个 crate 里 20 份 tokio 版本。

### 14.3 features 设计

```toml
[features]
default = ["json"]
json = ["dep:serde_json"]
metrics = ["dep:prometheus"]
```

规则：

- **feature 必须是加性的**：开启任意组合都应能编译通过，不能出现"开了 A 就必须关 B"；
- 用 `dep:` 语法隐式引入依赖，避免"feature 名 = 依赖名"造成的隐式开启；
- 谨慎开重 feature：`tokio = { features = ["full"] }` 会拖慢编译、增大体积；
- 下游 `default-features = false` 时应仍能构建最小功能集；
- 检查 feature 组合：`cargo hack check --feature-powerset`。

⚠️ **feature 统一**：同一 workspace 里多个 crate 依赖同一 crate 时，feature 会取并集。所以"某个 crate 悄悄开了 `full`"会让所有人的构建都变重。

### 14.4 profiles

```toml
[profile.release]
lto = "thin"          # 平衡编译时间与性能；"fat" 更极致
codegen-units = 1     # 更好的优化，编译更慢
panic = "abort"       # 体积更小；会失去 catch_unwind 与 unwind 时的 Drop
strip = "symbols"     # 去掉符号，减小二进制

[profile.dev]
opt-level = 0
debug = 1             # 降为 line-tables，加快链接（大项目明显）

[profile.bench]
lto = "thin"
```

### 14.5 `build.rs`

用于：生成代码（protobuf、bindgen）、嵌入版本信息、检测平台能力。

```rust
// build.rs
fn main() {
    println!("cargo:rerun-if-changed=proto/");
    println!("cargo:rustc-env=BUILD_VERSION={}", std::env::var("CARGO_PKG_VERSION").unwrap());
}
```

- 产物写到 `OUT_DIR`，用 `include!(concat!(env!("OUT_DIR"), "/gen.rs"))` 引入；
- `println!("cargo:rerun-if-changed=...")` 决定何时重新执行，漏写会导致缓存不刷新；
- `links` 字段表示"该 crate 链接了某个原生库"，同名的 `links` 全局唯一。

⚠️ `build.rs` 运行在宿主机（不是目标机），跨平台交叉编译时不能用它探测目标平台能力。

### 14.6 依赖治理

```bash
cargo tree -d                  # 同一 crate 的多个版本
cargo tree -i openssl          # 谁引入了它
cargo update -p time --precise 0.3.36   # 精确升级以绕开已知问题
cargo vendor                   # 离线分发
```

- `[patch.crates-io]` 临时指向 fork 或本地路径，修完记得删；
- `Cargo.lock`：**bin 提交、lib 不提交**（lib 的依赖由下游决定）；
- 定期 `cargo deny check`（漏洞、license、重复版本）与 `cargo audit`。

### 14.7 版本、MSRV 与发布前检查

```toml
[package]
version = "0.3.1"
rust-version = "1.75"
```

发布前清单：

1. `cargo test`、`cargo clippy -- -D warnings`、`cargo fmt --check` 全绿；
2. MSRV job 通过（用 `rust-version` 指定的工具链 `cargo check`）；
3. `cargo semver-checks` 确认没有意外的破坏性变更；
4. `cargo package --list` 检查打包内容（别把测试数据/密钥塞进去）；
5. `cargo doc` 无 broken intra-doc link；
6. README 的示例代码能跑（`cargo test --doc` 覆盖）。

---

## 15. 质量闸门与 CI

### 15.1 格式化

```toml
# rustfmt.toml
edition = "2021"
max_width = 100
imports_granularity = "Module"    # 需 nightly 的部分选项慎用
```

🧭 统一由 `cargo fmt` 决定格式，代码评审不再讨论空格与换行；CI 用 `cargo fmt --check` 阻断未格式化的提交。

### 15.2 clippy 分层策略

| 层级 | 建议 | 说明 |
|---|---|---|
| `clippy::all`（默认） | 全开 + `-D warnings` | 正确性与常见坑 |
| `clippy::pedantic` | 选择性开启 | 噪音大，挑有价值的规则 |
| `clippy::nursery` | 一般不开 | 不稳定、变动频繁 |
| `clippy::restriction` | 按规则单独开 | 每条都要团队讨论 |

```toml
# Cargo.toml（1.74+）
[lints.clippy]
all = "warn"
unwrap_used = "warn"
clone_on_ref_ptr = "warn"
```

常用高价值 lint：`unwrap_used`、`expect_used`、`clone_on_ref_ptr`、`redundant_clone`、`needless_collect`、`large_enum_variant`、`manual_let_else`。

### 15.3 `allow` vs `expect`

```rust
#[allow(dead_code)]                       // 永久豁免，后续不会提醒
#[expect(dead_code, reason = "预留接口")]  // 如果 lint 不再触发，会反过来报警告
```

🧭 优先 `#[expect]`（1.81+）：它自带"豁免过期检测"，避免遗留一堆无用的 `allow`。

### 15.4 供应链与兼容性检查

| 工具 | 作用 | CI 位置 |
|---|---|---|
| `cargo deny check` | license、漏洞公告、重复版本、来源白名单 | PR |
| `cargo audit` | 只查漏洞（比 deny 轻） | 每日定时 |
| `cargo semver-checks` | 公开 API 的破坏性变更检测 | 发布前 |
| `cargo machete` | 未使用的依赖 | PR |

### 15.5 最小 CI 矩阵

```yaml
# .github/workflows/ci.yml（节选）
jobs:
  test:
    strategy:
      matrix:
        os: [ubuntu-latest, macos-latest]
    runs-on: ${{ matrix.os }}
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
        with: { components: "clippy,rustfmt" }
      - uses: Swatinem/rust-cache@v2
      - run: cargo fmt --check
      - run: cargo clippy --all-targets -- -D warnings
      - run: cargo test --all-features
      - run: cargo doc --no-deps

  msrv:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@1.75.0
      - run: cargo check --all-features
```

要点：**fmt / clippy / test / doc 四条独立失败信号**；`rust-cache` 复用 `target/`；MSRV 单列 job，别混在主矩阵里。

### 15.6 本地一键闸门

```makefile
# justfile 或 Makefile
check:
    cargo fmt --check
    cargo clippy --all-targets -- -D warnings
    cargo test --all-features
    cargo doc --no-deps
```

🧭 用 `just`（或 `xtask`）统一团队命令，避免每人记住不同参数；CI 与本地跑的是同一条命令，才不会出现"本地过、CI 挂"。

---

## 16. 测试体系

### 16.1 三类测试的分工

| 类型 | 位置 | 能访问 | 适合测什么 |
|---|---|---|---|
| 单元测试 | 源码内 `#[cfg(test)] mod tests` | 私有项 | 算法、边界、错误分支 |
| 集成测试 | `tests/*.rs` | 仅公开 API | 对外契约、端到端流程 |
| 文档测试 | `///` 注释里的代码块 | 同外部使用者 | 示例正确性 + 用法说明 |

```rust
// src/lib.rs 末尾
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn parses_positive() {
        assert_eq!(parse("42").unwrap(), 42);
    }
}
```

集成测试每个文件是独立 crate，只 `use my_crate::...`，因此能发现"API 设计对外不好用"的问题。

### 16.2 断言与失败信息

```rust
assert_eq!(actual, expected);                       // 失败时打印两边
assert_eq!(actual, expected, "input was {input}");  // 带上下文
assert!(matches!(err, Error::Timeout { .. }));
```

- 失败信息要能直接定位问题：把输入带进断言；
- 浮点比较用 `approx` 或手写 epsilon；
- 结构化对象优先整体比较（`assert_eq!` on struct），而不是逐字段断言——差别会一眼看出。

### 16.3 异步测试

```rust
#[tokio::test]                       // 默认 current_thread，够快
async fn handles_timeout() {
    let r = tokio::time::timeout(Duration::from_millis(10), slow()).await;
    assert!(r.is_err());
}

#[tokio::test(flavor = "multi_thread", worker_threads = 2)]
async fn concurrent_writes() { /* 需要真并发时 */ }
```

时间相关测试用**虚拟时钟**，不要真的 sleep：

```rust
#[tokio::test(start_paused = true)]
async fn retries_with_backoff() {
    // 这里的 sleep 会瞬间推进
}
```

### 16.4 参数化、属性测试、快照、打桩

| 工具 | 解决什么 | 示例场景 |
|---|---|---|
| `rstest` | 表驱动/参数化 | 一组输入输出用例 |
| `proptest` | 生成随机输入验证不变量 | 编解码往返、排序稳定性 |
| `insta` | 快照测试 | 渲染输出、错误文案 |
| `mockall` | 自动生成 mock | 隔离外部依赖 |
| `wiremock` | HTTP 打桩 | 客户端测试 |
| `tempfile` | 临时目录夹具 | 文件系统逻辑 |

```rust
#[rstest]
#[case("1", 1)]
#[case(" 2 ", 2)]
fn parses(#[case] input: &str, #[case] expected: i32) {
    assert_eq!(parse(input).unwrap(), expected);
}

proptest! {
    #[test]
    fn roundtrip(v: Vec<u8>) {
        prop_assert_eq!(decode(&encode(&v)).unwrap(), v);
    }
}
```

### 16.5 覆盖率与 flaky 治理

```bash
cargo llvm-cov --html          # 行/分支覆盖率
cargo nextest run              # 更快的运行器，逐测试进程隔离
```

- 覆盖率只作参考，重点看"错误分支是否被测到"；
- flaky 的三大来源：真实时间依赖、端口/文件竞争、共享全局状态 → 分别用虚拟时钟、随机端口/临时目录、测试内独立实例解决；
- `#[ignore]` 标记慢测试，CI 用 `--ignored` 单独跑一轮。

---

## 17. 可观测性与运维

### 17.1 `tracing` 的 span 与 field

```rust
use tracing::{info, instrument};

#[instrument(skip(cfg), fields(user = %user_id))]
async fn handle(cfg: &Config, user_id: u64) -> Result<()> {
    info!(retry = 2, "processing");       // 结构化字段，不是字符串拼接
    Ok(())
}
```

- `#[instrument]` 放在**函数入口**，自动生成 span 并记录参数（大参数用 `skip`）；
- 字段优于文本：`info!(order_id, "created")` 能被日志系统索引；
- `%value` 用 `Display`，`?value` 用 `Debug`，`value` 要求实现了 `Value`。

### 17.2 日志层级与采样

| 级别 | 用途 |
|---|---|
| `error` | 需要人介入（数据不一致、依赖不可用） |
| `warn` | 可自愈但值得关注（重试成功、降级） |
| `info` | 关键业务节点（启动、请求完成） |
| `debug` | 排查用细节 |
| `trace` | 高频内部状态，默认关闭 |

🧭 高频路径（每请求多条）不要用 `info`，否则日志成本超过业务本身；必要时按 `RUST_LOG` 过滤或采样。

### 17.3 指标、崩溃与优雅退出

```rust
// 指标：计数器/直方图通常由 metrics 或 prometheus crate 提供
counter!("requests_total", "path" => path).increment(1);

// panic 上报
std::panic::set_hook(Box::new(|info| {
    eprintln!("panic: {info}");
    // 这里不要加锁、不要分配大对象
}));
```

优雅退出三步：**监听信号 → 取消令牌 → 等待在飞任务（带超时）**。

```rust
let token = CancellationToken::new();
tokio::spawn({
    let t = token.clone();
    async move {
        tokio::signal::ctrl_c().await.ok();
        t.cancel();
    }
});
// 主循环：token.cancelled() 时停止接新任务，等待 JoinSet 清空，超时则 abort
```

### 17.4 配置分层

优先级从高到低：**命令行参数 > 环境变量 > 配置文件 > 内置默认值**。每一层都要能回答"这个值从哪来"，否则线上排查会变成猜谜。

---

## 18. API 设计与发布

### 18.1 命名约定（RFC 430 摘要）

| 场景 | 命名 | 例子 |
|---|---|---|
| 构造 | `new` / `with_*` | `Vec::new`、`with_capacity` |
| 可能失败 | `try_new` | `try_from` 配套 |
| 转换（廉价借用） | `as_*` | `as_str`、`as_bytes` |
| 转换（有开销） | `to_*` | `to_string`、`to_vec` |
| 消费转换 | `into_*` | `into_inner`、`into_iter` |
| 查询 | `is_*` / `has_*` | `is_empty` |
| 迭代 | `iter` / `iter_mut` / `into_iter` | 三元组保持一致 |

### 18.2 四种常用模式

```rust
// 1) builder：字段多、可选参数多
let cfg = Config::builder().timeout(Duration::from_secs(3)).build()?;

// 2) newtype：类型安全 + 绕过孤儿规则
pub struct Port(u16);

// 3) typestate：用类型表达状态
struct Draft; struct Published;
impl Doc<Draft> { fn publish(self) -> Doc<Published> { /* ... */ } }

// 4) sealed trait：公开 trait 但不允许外部实现
mod private { pub trait Sealed {} }
pub trait Plugin: private::Sealed {}
```

### 18.3 关键属性

| 属性 | 作用 |
|---|---|
| `#[must_use]` | 忽略返回值时告警（`Result`、纯函数、builder 中间态） |
| `#[non_exhaustive]` | 外部不能穷尽匹配/构造，便于加字段变体 |
| `#[deprecated(since, note)]` | 渐进废弃，note 里写替代方案 |
| `#[doc(hidden)]` | 公开但不出现在文档里（内部宏辅助项） |
| `#[doc(alias = "...")]` | 改善搜索命中 |

### 18.4 公开 API 的取舍

| 决策 | 静态分发（泛型/`impl Trait`） | 动态分发（`dyn`） |
|---|---|---|
| 性能 | 更好（可内联） | 一次间接 + 无法内联 |
| 编译时间/体积 | 单态化膨胀 | 更小 |
| 可扩展性 | 调用方给类型 | 可运行时装配 |
| 对象安全 | 无限制 | 受对象安全约束 |

🧭 公开 API 建议：**参数用 `impl Trait`/泛型，返回用具体类型或 `impl Trait`，只在需要异构集合时暴露 `dyn`**。避免在公开签名里出现复杂的嵌套泛型。

### 18.5 semver 与破坏性变更

**破坏性**（需要升 major）：删除/重命名公开项、收紧泛型约束、加必填参数、改变错误类型、给公开 enum 加变体（未标 `non_exhaustive`）、改变 trait 方法签名、把 `pub` 改小。

**非破坏性**：加新函数/新类型、加带默认实现的方法、放宽约束、给 `non_exhaustive` 类型加字段。

🧭 0.x 版本可以自由破坏，但**每次破坏都要在 CHANGELOG 明确写"迁移方式"**。

### 18.6 发布流程

```bash
cargo publish --dry-run        # 先看打包内容
cargo package --list           # 逐项检查文件
cargo publish
```

`Cargo.toml` 里补齐 `description`、`license`、`repository`、`readme`、`keywords`、`categories`；docs.rs 的构建特性用 `[package.metadata.docs.rs] all-features = true` 控制。

---

## 19. 性能工程

### 19.1 先测量，再优化

```rust
// benches/parse.rs
use criterion::{criterion_group, criterion_main, Criterion};

fn bench_parse(c: &mut Criterion) {
    c.bench_function("parse_1k", |b| b.iter(|| parse(SAMPLE)));
}
criterion_group!(benches, bench_parse);
criterion_main!(benches);
```

🧭 没有基准的优化都是猜测。先确认瓶颈在 CPU、内存分配、锁竞争还是 IO，再动手。

### 19.2 分配的四个高频点

| 位置 | 问题 | 改法 |
|---|---|---|
| 循环里 `format!` | 每次分配 | 复用 `String` + `write!` |
| `collect::<Vec<_>>()` 中转 | 多余分配 | 直接链式消费 |
| `clone()` 消 borrow 报错 | 隐式深拷贝 | 传引用/`Cow` |
| `Vec::remove(0)` | O(n) 搬移 | `VecDeque` 或 `swap_remove` |

```rust
// 复用缓冲
let mut buf = String::with_capacity(64);
for item in items {
    buf.clear();
    write!(&mut buf, "{item}:{}", item.len())?;
    sink.write_all(buf.as_bytes())?;
}
```

### 19.3 内联、LTO 与 PGO

- `#[inline]` 主要影响**跨 crate** 内联（同 crate 内 LLVM 自己会判断）；不要到处加；
- `lto = "thin"` 通常是性价比最高的档位；`fat` 适合最终发布；
- `codegen-units = 1` 提升优化质量但显著增加编译时间；
- PGO：`-Cprofile-generate` 跑真实负载 → `-Cprofile-use` 重编译，常见 5–15% 提升。

### 19.4 剖析工具

| 工具 | 用途 |
|---|---|
| `perf record` / `flamegraph` | CPU 火焰图，找热点函数 |
| `heaptrack` / DHAT | 分配次数与峰值内存 |
| `cargo bloat` | 二进制里谁占体积 |
| `tokio-console` | 异步任务延迟、阻塞、waker 次数 |
| `cargo llvm-cov` | 覆盖率（顺带发现没跑到的热路径） |

### 19.5 常见反模式

| 反模式 | 后果 |
|---|---|
| 热路径 `Box<dyn Trait>` | 无法内联，间接调用 |
| 大 struct 按值传递 | 栈拷贝 |
| `Arc<Mutex<HashMap>>` 每次全锁 | 串行化 |
| 用 `String` 传只读文本 | 多一次分配 |
| 迭代器里 `println!` | IO 淹没计算 |

---

## 20. 项目骨架模板

### 20.1 目录结构

```text
myapp/
├── Cargo.toml
├── rust-toolchain.toml
├── rustfmt.toml
├── deny.toml
├── justfile
├── README.md
├── src/
│   ├── main.rs          # 只做参数解析、初始化、调用 run
│   ├── lib.rs           # 对外 API 门面（re-export）
│   ├── config.rs
│   ├── error.rs         # 统一错误类型 + 对外映射
│   ├── domain/          # 纯业务逻辑，无 IO
│   ├── infra/           # 数据库、HTTP、文件
│   └── api/             # 路由/命令层
├── tests/               # 集成测试
├── benches/
└── .github/workflows/ci.yml
```

### 20.2 模块切分经验

- 单文件超过 **500 行**就考虑拆分；超过 800 行基本是职责混在一起；
- 按"变化原因"切分，而不是按"类型"切分（`user/`、`order/` 优于 `models/`、`handlers/`）；
- `mod.rs` 与 `foo.rs` 两种风格选一种，团队内统一；
- 公开面收敛到 `lib.rs` 的 `pub use`，子模块默认 `pub(crate)`。

### 20.3 错误边界

```rust
// error.rs
#[derive(Debug, thiserror::Error)]
pub enum AppError {
    #[error("config invalid: {0}")]
    Config(String),
    #[error(transparent)]
    Internal(#[from] anyhow::Error),
}

impl AppError {
    pub fn code(&self) -> &'static str {
        match self { Self::Config(_) => "invalid_config", Self::Internal(_) => "internal" }
    }
}
```

内部用 `anyhow` 自由传播，进程边界统一转成 `AppError`，对外只暴露稳定错误码。

### 20.4 CLI 与配置

```rust
use clap::Parser;

#[derive(Parser)]
#[command(version, about)]
struct Cli {
    #[arg(short, long, default_value = "app.toml")]
    config: std::path::PathBuf,
    #[arg(long, env = "APP_PORT")]
    port: Option<u16>,
}
```

`env` 属性让"命令行 > 环境变量"的优先级由代码结构直接保证。

### 20.5 启动与退出骨架

```rust
#[tokio::main]
async fn main() -> anyhow::Result<()> {
    tracing_subscriber::fmt().with_env_filter("info").init();
    let cli = Cli::parse();
    let cfg = config::load(&cli.config)?;
    let token = CancellationToken::new();
    let app = App::new(cfg, token.clone())?;
    tokio::select! {
        r = app.run() => r?,
        _ = shutdown_signal() => { token.cancel(); app.wait().await?; }
    }
    Ok(())
}
```

### 20.6 这份骨架的验收清单

- [ ] `cargo fmt --check`、`cargo clippy -D warnings`、`cargo test` 全绿
- [ ] `cargo doc` 无警告，公开项都有文档
- [ ] MSRV job 通过
- [ ] 启动失败有清晰错误信息，退出码非零
- [ ] 有优雅退出，长任务可被取消
- [ ] 日志有结构（字段而非拼接），关键路径有 span
- [ ] `cargo deny check` 通过
- [ ] README 有"如何运行/如何测试/目录说明"三段

---

## 附录 A　编译错误速查

| 错误 | 含义 | 常见解法 |
|---|---|---|
| E0308 mismatched types | 类型不匹配 | 检查返回值/分支类型是否一致，注意多余分号 |
| E0382 use of moved value | 值已被移动 | 借用、`clone`、调整所有权顺序 |
| E0502 cannot borrow as mutable | 已有不可变借用 | 缩短借用作用域（NLL） |
| E0499 two mutable borrows | 同时两个可变借用 | 索引、`split_at_mut`、重构 |
| E0505 cannot move out while borrowed | 借用期间移动 | 先结束借用或用 `mem::take` |
| E0507 cannot move out of borrowed | 从借用里搬值 | `clone`、`take`、改 `&mut` |
| E0515 return reference to local | 返回局部变量引用 | 返回拥有类型 |
| E0597 borrowed value does not live long enough | 生命周期不够 | 提前绑定、调整结构体持引用 |
| E0716 temporary value dropped while borrowed | 临时值提前释放 | 先 `let` 绑定再用 |
| E0106 missing lifetime specifier | 无法推导生命周期 | 显式 `<'a>` |
| E0004 non-exhaustive patterns | match 未穷尽 | 补分支或 `_ =>` |
| E0061 wrong number of arguments | 参数数量不符 | 查签名（含 `self`） |
| E0277 trait bound not satisfied | 缺 trait 实现 | derive、手写 impl、检查 `Send`/`Sync` |
| E0599 no method found | 方法不存在/未导入 trait | `use` 对应 trait、检查解引用层级 |
| E0432/E0433 unresolved import | 路径错误 | 检查 crate 名（连字符→下划线）与 feature |
| E0658 feature not stable | 用了未稳定特性 | 换稳定写法或固定 nightly |
| future is not `Send` | 跨 await 持有非 Send 值 | 缩小作用域、换 tokio 锁、去 `Rc` |
| borrowed data escapes outside of method | 借用逃逸出函数 | 返回拥有类型或加生命周期参数 |
| async fn in trait is not dyn compatible | RPITIT 不能 dyn | 手写擦除层（见 5.7） |
| cannot infer type | 类型推断失败 | 显式标注 `collect::<Vec<_>>()` |
| use of unstable library feature | 调用了未稳定 API | 查稳定替代 |
| integer overflow（debug） | 算术溢出 | 显式 `checked_*`/`saturating_*` |
| already borrowed: BorrowMutError | RefCell 运行期冲突 | 缩小 `borrow_mut` 作用域 |
| attempt to subtract with overflow | 无符号下溢 | 判空或用 `saturating_sub` |

---

## 附录 B　Rust 与 C++/Go/Java 对照

| 概念 | C++ | Go | Java | Rust |
|---|---|---|---|---|
| 内存管理 | RAII/智能指针 | GC | GC | 所有权 + 编译期借用检查 |
| 多态 | 虚函数/模板 | interface | 继承/interface | trait（泛型静态 + `dyn` 动态） |
| 空值 | 指针/`std::optional` | `nil` | `null` | `Option<T>`（无 null） |
| 错误 | 异常/错误码 | `error` 返回值 | 异常 | `Result<T, E>` + `?` |
| 并发 | `std::thread`/mutex | goroutine/channel | 线程/`CompletableFuture` | 线程 + `Send/Sync`；async 需 runtime |
| 泛型实现 | 模板（头文件） | 1.18+ 泛型 | 类型擦除 | 单态化（零开销，体积换速度） |
| 继承 | 有 | 组合 | 有 | 无，用 trait + 组合 |
| 可变性 | 默认可变 | 默认可变 | 默认可变 | **默认不可变**，需 `mut` |

🧭 迁移时最容易带过来的三个坏习惯：到处 `clone()`（等价于到处 new）、用 `Rc<RefCell<>>` 模拟共享对象（应重构数据流）、把 `Result` 当异常层层包装（应设计错误边界）。

---

## 附录 C　Code Review 检查清单

- [ ] 公开 API 的 `pub` 面是否收敛（子模块默认 `pub(crate)`）
- [ ] 库代码里是否残留 `unwrap`/`expect`（除非能证明不可能失败）
- [ ] 是否跨 `.await` 持有 `std::sync::MutexGuard` 或 `RefCell` 借用
- [ ] `clone()` 是否都可以用引用替代
- [ ] 数值转换是否用了 `as` 而非 `TryFrom`
- [ ] 公开 enum 是否加了 `#[non_exhaustive]`
- [ ] `match` 是否避免了无谓的 `_` 通配（新增变体应触发编译错误）
- [ ] 错误类型是否区分"库内部错误"与"对外错误码"
- [ ] 新增依赖是否检查了 license、维护状态与体积
- [ ] 热路径是否避免 `format!`、`collect` 中转与 `Box<dyn>`
- [ ] `unsafe` 块是否有 `// SAFETY:` 说明
- [ ] 新增行为是否有对应测试（含错误分支）
- [ ] 日志是否用结构化字段，是否泄漏敏感信息
- [ ] 是否有 CHANGELOG 条目与 semver 判断

---

## 附录 D　延伸阅读

| 资源 | 用途 |
|---|---|
| The Rust Programming Language（官方书） | 系统入门，示例最权威 |
| Rust by Example | 快速查语法片段 |
| The Rust Reference | 语义细节的最终依据（生命周期、模式、类型布局） |
| Rustonomicon | `unsafe`、内存模型、FFI 必读 |
| Asynchronous Programming in Rust（async book） | `Future`/执行器原理 |
| Rust API Guidelines | 公开 API 设计规则（RFC 430 等） |
| Effective Rust | 惯用法与反模式，适合进阶 |
| The Rust Performance Book | 性能优化方法论 |
| Clippy lint 列表 | 每条 lint 都有正反例，是最好的风格教材 |
| Tokio Tutorial | 异步实战与 `select!`/取消语义 |

---

## 附录 E　术语表

| 术语 | 含义 |
|---|---|
| **所有权（ownership）** | 每个值有唯一所有者，所有者离开作用域时释放 |
| **借用（borrow）** | 通过 `&T`/`&mut T` 临时访问而不取得所有权 |
| **生命周期（lifetime）** | 引用有效性的区间，用 `'a` 标注 |
| **NLL** | 非词法生命周期，借用结束于最后一次使用 |
| **单态化（monomorphization）** | 为每个泛型实例生成专门代码 |
| **对象安全（object safety）** | 一个 trait 能否作为 `dyn Trait` 使用 |
| **RPIT / RPITIT** | 返回位置 `impl Trait` / trait 方法中的返回位置 `impl Trait` |
| **NPO（niche optimization）** | 利用非法值位模式把 `Option<T>` 压成同尺寸 |
| **RAII** | 资源获取即初始化，靠 `Drop` 自动释放 |
| **cancel safety** | future 被 drop 时不留副作用的性质 |
| **背压（backpressure）** | 消费端限速生产端的机制（有界队列） |
| **MSRV** | 最低支持 Rust 版本 |
