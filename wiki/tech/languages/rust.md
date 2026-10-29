---
title: "Rust 语言与语法详解（结合 codex 代码库实例）"
domain: tech/languages
type: deep-dive
tags: [rust, ownership, async, serde, codex-rs]
source: "codex-rs release-0.151.0 源码实例"
updated: 2026-10-16
---

# Rust 语言与语法详解（结合 codex 代码库实例）

> 目标读者：能读懂 C/C++/Go/Java，想系统掌握 Rust 语法与工程惯用法。
> 本文所有"codex 实例"均来自 codex-rs（release-0.151.0）。标注了 `文件:行号`，方便对照源码。
> 约定：`⚠️` 标记易错点；`💡` 标记工程惯用法。

---

## 目录

- [1. 工程与模块系统](#1-工程与模块系统)
  - [1.1 Cargo workspace 与 crate](#11-cargo-workspace-与-crate)
  - [1.2 module 与可见性](#12-module-与可见性)
  - [1.3 `use` 与路径](#13-use-与路径)
  - [1.4 `main`、crate 入口与文件布局](#14-maincrate-入口与文件布局)
  - [1.5 常量与静态变量](#15-常量与静态变量)
- [2. 基础语法](#2-基础语法)
  - [2.1 变量与绑定](#21-变量与绑定)
  - [2.2 标量与复合类型](#22-标量与复合类型)
  - [2.3 函数与表达式](#23-函数与表达式)
  - [2.4 控制流](#24-控制流)
  - [2.5 字符串：`String` vs `&str`](#25-字符串string-vs-str)
  - [2.6 `Option` 与 `Result` 基础](#26-option-与-result-基础)
- [3. 所有权、借用与生命周期](#3-所有权借用与生命周期)
  - [3.1 三条规则](#31-三条规则)
  - [3.2 借用规则](#32-借用规则)
  - [3.3 生命周期标注](#33-生命周期标注)
  - [3.4 内部可变性（Interior Mutability）](#34-内部可变性interior-mutability)
- [4. 结构体、枚举与模式匹配](#4-结构体枚举与模式匹配)
  - [4.1 struct 三种形态](#41-struct-三种形态)
  - [4.2 enum：带数据的和类型](#42-enum带数据的和类型)
  - [4.3 `match`：穷尽 + 解构 + guard + 绑定](#43-match穷尽--解构--guard--绑定)
  - [4.4 便捷模式语法](#44-便捷模式语法)
- [5. Trait 与泛型](#5-trait-与泛型)
  - [5.1 trait 定义与默认方法](#51-trait-定义与默认方法)
  - [5.2 泛型函数 / 结构体 / 方法](#52-泛型函数--结构体--方法)
  - [5.3 `impl Trait` 与 `dyn Trait`](#53-impl-trait-与-dyn-trait)
  - [5.4 RPITIT：trait 方法返回 `impl Future`](#54-rpitittrait-方法返回-impl-future)
  - [5.5 关联类型、关联常量与 type alias](#55-关联类型关联常量与-type-alias)
  - [5.6 derive：自动实现常见 trait](#56-derive自动实现常见-trait)
  - [5.7 孤儿规则、newtype 与 From/Into](#57-孤儿规则newtype-与-frominto)
  - [5.8 `Self` 与 self 接收者速查](#58-self-与-self-接收者速查)
  - [5.9 运算符重载与 trait 边界](#59-运算符重载与-trait-边界)
- [6. 错误处理](#6-错误处理)
  - [6.1 panic vs Result](#61-panic-vs-result)
  - [6.2 `?` 与自动 `From` 转换](#62--与自动-from-转换)
  - [6.3 thiserror：声明式错误类型](#63-thiserror声明式错误类型)
  - [6.4 `Result` 类型别名与自定义 Result](#64-result-类型别名与自定义-result)
  - [6.5 错误传播的工程习惯](#65-错误传播的工程习惯)
- [7. 智能指针与所有权模型延伸](#7-智能指针与所有权模型延伸)
  - [7.1 `Arc`：跨任务共享的事实标准](#71-arc跨任务共享的事实标准)
  - [7.2 `Weak`：打破环](#72-weak打破环)
  - [7.3 `Pin` 与 `BoxFuture`](#73-pin-与-boxfuture)
  - [7.4 `Cow`：写时克隆](#74-cow写时克隆)
  - [7.5 `PhantomData`：在结构体里"声明"生命周期/类型参数](#75-phantomdata在结构体里声明生命周期类型参数)
- [8. 并发与异步](#8-并发与异步)
  - [8.1 概念对照](#81-概念对照)
  - [8.2 `spawn` 与 `'static`](#82-spawn-与-static)
  - [8.3 常用同步原语](#83-常用同步原语)
  - [8.4 channel 选型](#84-channel-选型)
  - [8.5 `select!`：多路等待](#85-select多路等待)
  - [8.6 取消模型：`CancellationToken`](#86-取消模型cancellationtoken)
  - [8.7 RAII 卫士](#87-raii-卫士)
  - [8.8 手动实现 `Future`（进阶）](#88-手动实现-future进阶)
- [9. 宏与元编程](#9-宏与元编程)
  - [9.1 `macro_rules!`（声明宏）](#91-macro_rules声明宏)
  - [9.2 常用宏速查](#92-常用宏速查)
  - [9.3 derive 宏与属性宏](#93-derive-宏与属性宏)
- [10. 属性（Attributes）与条件编译](#10-属性attributes与条件编译)
- [11. serde 序列化惯用法](#11-serde-序列化惯用法)
  - [11.1 基本形态](#111-基本形态)
  - [11.2 标签联合（内部/相邻/无标签）](#112-标签联合内部相邻无标签)
  - [11.3 常见字段属性](#113-常见字段属性)
  - [11.4 自定义序列化模块（进阶）](#114-自定义序列化模块进阶)
  - [11.5 ts-rs / schemars](#115-ts-rs--schemars)
- [12. codex 代码风格约定（AGENTS.md 摘要）](#12-codex-代码风格约定agentsmd-摘要)
- [13. 综合示例：一个 mini agent loop](#13-综合示例一个-mini-agent-loop)
- [14. 常见编译错误速查](#14-常见编译错误速查)
- [附：延伸阅读（codex 内可直接当教材读的文件）](#附延伸阅读codex-内可直接当教材读的文件)

---

## 1. 工程与模块系统

### 1.1 Cargo workspace 与 crate

- **package**：一个 `Cargo.toml` 及其源码；**crate**：一次编译单元（lib 或 bin）。
- **workspace**：多个 crate 共享一个 `Cargo.lock` 和 `target/`：

```toml
# codex-rs/Cargo.toml（节选）
[workspace]
members = ["core", "protocol", "codex-api", "tools", "tui", "app-server", ...]
resolver = "2"
```

- crate 名以 `-` 连接，代码里引用时自动变成 `_`：`codex-core` → `use codex_core::...`。
- codex 约定：核心 crate 统一前缀 `codex-`（`core/` 目录的 crate 叫 `codex-core`）。

### 1.2 module 与可见性

```rust
// lib.rs 声明模块；文件树即模块树
mod agent;                 // 私有模块
pub mod protocol;          // 公开模块
pub(crate) mod session;    // 仅当前 crate 可见

// 单个 item 的可见性
pub struct Session {}          // 公开
pub(crate) struct TurnInputQueue { items: Vec<TurnInput> }  // 字段默认私有
pub(super) fn helper() {}      // 仅父模块可见（codex 中大量使用）
pub(in crate::tools) fn f() {} // 限定路径可见
```

- 默认一切都是**私有**的；"选择性公开"是 Rust API 设计的核心手段。
- codex 刻意压缩 API 面：`pub(crate)` 随处可见，例如 `SessionTask`、`RegularTask`（`core/src/tasks/mod.rs:61-65`）。

### 1.3 `use` 与路径

```rust
use std::sync::Arc;                       // 绝对路径（从 crate root）
use crate::session::session::Session;     // 从当前 crate 根
use super::SessionTask;                   // 从父模块
use self::inner::Foo;                     // 从当前模块
use codex_protocol::models::ResponseItem; // 跨 crate

// 一次导入多个
use codex_protocol::protocol::{EventMsg, TurnAbortReason, TurnAbortedEvent};

// 别名
use codex_protocol::error::Result as CodexResult;
```

💡 codex 里每个 `use` 独占一行、按字母序排列（rustfmt 默认 group 规则）。

### 1.4 `main`、crate 入口与文件布局

```rust
// bin crate 的入口
fn main() { ... }

// lib crate 的入口是 lib.rs，通常只做模块声明与 re-export
pub use crate::error::CodexErr;
```

codex 的测试文件布局约定（AGENTS.md）：

```rust
// core/src/session/parser.rs
#[cfg(test)]
#[path = "parser_tests.rs"]   // 测试放独立文件，避免实现文件膨胀
mod tests;
```

### 1.5 常量与静态变量

```rust
const GRACEFULL_INTERRUPTION_TIMEOUT_MS: u64 = 100;             // 编译期常量，内联
pub(crate) const TASK_COMPACT_METRIC: &str = "codex.task.compact";

static ACTIVE_TURNS: Gauge = Gauge::new("core.turns.active");   // 全局静态，需 Sync
static PENDING_MAILBOX_MESSAGES: Gauge = Gauge::new("core.mailbox.pending");
```

⚠️ `static mut` 基本被淘汰：需要全局可变状态时用 `OnceLock`/`LazyLock`/`AtomicXxx`/`Mutex`。

```rust
use std::sync::OnceLock;

static CONFIG: OnceLock<Config> = OnceLock::new();

fn config() -> &'static Config {
    CONFIG.get_or_init(|| load_config())
}
```

---

## 2. 基础语法

### 2.1 变量与绑定

```rust
let x = 42;              // 不可变绑定（默认！）
let mut count = 0;       // 可变绑定
count += 1;

let y: u64 = 100;        // 显式类型标注
let (a, b) = (1, "two"); // 解构绑定

// shadowing：用同名 let 覆盖上一个绑定（可改变类型）
let spaces = "   ";
let spaces = spaces.len();   // &str -> usize

// 未使用变量用 _ 前缀消音
let _guard = lock();
```

💡 codex 中大量 `let ... = ... .await;` 链式风格，注意 `mut` 只加在必须处。

### 2.2 标量与复合类型

| 类别 | 类型 | 说明 |
|---|---|---|
| 整数 | `i8..i128`、`u8..u128`、`isize`、`usize` | 默认 `i32`；下标/长度用 `usize` |
| 浮点 | `f32`、`f64` | 默认 `f64` |
| 布尔 | `bool` | |
| 字符 | `char` | 4 字节 Unicode 标量 |
| 元组 | `(T1, T2)` | 定长、可异构 |
| 数组 | `[T; N]` | 定长、同构，栈上 |
| 切片 | `&[T]` | 数组/Vec 的视图 |
| 字符串 | `String` / `&str` | 拥有/借用 |

```rust
let tuple: (i32, &str) = (1, "a");
let arr: [u8; 3] = [1, 2, 3];
let slice: &[u8] = &arr[..];
let boxed: Vec<u8> = arr.to_vec();
```

### 2.3 函数与表达式

Rust 是**表达式语言**：`if`、`match`、块 `{}` 都有值；函数体最后一个表达式即返回值（无分号！）。

```rust
fn add(a: i32, b: i32) -> i32 {
    a + b          // 无分号 = 返回
}

fn early(v: Vec<i32>) -> Option<i32> {
    if v.is_empty() {
        return None;      // 提前返回也可以
    }
    Some(v[0])
}
```

### 2.4 控制流

```rust
if x > 0 { "pos" } else { "neg" };          // if 是表达式

let mut i = 0;
while i < 10 { i += 1; }

loop {                                        // 无限循环
    if done() { break 42; }                   // break 可带值
    else { continue; }
}

for item in &items { ... }                    // 迭代器 for（最多用）

'outer: for x in 0..3 {                       // 标签跳转
    for y in 0..3 {
        if x + y == 3 { break 'outer; }
    }
}
```

### 2.5 字符串：`String` vs `&str`

```rust
let owned: String = String::from("hello");     // 堆分配、可增长
let borrowed: &str = &owned;                    // 借用视图（切片）
let s2 = owned.clone();                         // 深拷贝
let is_empty = borrowed.is_empty();

// 拼接
let msg = format!("{name}: {value}");           // 内联格式串
let joined = vec!["a", "b"].join(",");          // "a,b"
```

⚠️ `&String` 会自动解引用为 `&str`，写函数签名时优先接收 `&str`。

```rust
fn log(msg: &str) { ... }   // 而非 &String
```

### 2.6 `Option` 与 `Result` 基础

```rust
let a: Option<i32> = Some(1);
let b: Option<i32> = None;

// 常用组合子（全部是表达式，无 panic）
let c = a.map(|v| v + 1).unwrap_or_default();
let d = a.and_then(|v| (v > 0).then_some(v));
let e = a.is_some_and(|v| v == 1);          // codex 常用

// Result
fn parse(s: &str) -> Result<i32, std::num::ParseIntError> {
    let n: i32 = s.parse()?;                // ? 自动把 Err 往上抛
    Ok(n * 2)
}
```

---

## 3. 所有权、借用与生命周期

### 3.1 三条规则

1. 每个值有唯一 owner；
2. 同一时刻只有**一个** owner；
3. owner 离开作用域，值被 drop。

```rust
let s1 = String::from("hi");
let s2 = s1;              // 移动（move），s1 失效
// println!("{s1}");      // ❌ E0382: borrow of moved value

let s3 = s2.clone();      // 显式深拷贝，两者都可用
```

实现了 `Copy` 的类型（整数、bool、char、引用、元组/数组 of Copy）赋值时是拷贝：

```rust
let x = 5;
let y = x;                // x 仍可用
```

### 3.2 借用规则

- 任意多个不可变借用 `&T`，**或**恰好一个可变借用 `&mut T`，二者不可同时存在。

```rust
fn len(s: &str) -> usize { s.len() }          // 借用读
fn push(s: &mut String, c: char) { s.push(c); } // 借用写

let mut s = String::from("a");
let r = &s;                 // 不可变借用开始
// push(&mut s, 'b');      // ❌ 不能同时可变借用
println!("{r}");            // 借用到此结束（NLL 非词法生命周期）
push(&mut s, 'b');          // ✅ 可以了
```

⚠️ NLL（non-lexical lifetimes）：借用结束于最后一次使用，而不是作用域末尾。

### 3.3 生命周期标注

生命周期回答"引用能活多久"。大多数时候编译器自动推导（省略规则）；结构体里存引用、或签名有多个引用且返回引用时需要显式标注。

```rust
// 返回的 &str 与输入 x 活得一样久
fn first_word<'a>(x: &'a str) -> &'a str { ... }
// 省略写法（输入只有一个引用时）
fn first_word(x: &str) -> &str { ... }

// 结构体持有引用
struct Token<'a> { text: &'a str }
```

`'static`：活到程序结束；字符串字面量就是 `&'static str`，也被用作"必须拥有/可跨任务"的约束：

```rust
pub struct RunningTask {
    kind: TaskKind,
    ...
}

// 约束：可以安全地 move 进 tokio::spawn 的 future
async fn spawn_worker<T: Send + 'static>(t: T) { tokio::spawn(async move { use_it(t).await }); }
```

### 3.4 内部可变性（Interior Mutability）

| 类型 | 线程 | 用途 |
|---|---|---|
| `Cell<T>` | 单线程 | 整体替换的小值 |
| `RefCell<T>` | 单线程 | 运行期检查借用规则 |
| `OnceLock<T>` | 多线程 | 一次性初始化 |
| `LazyLock<T>` | 多线程 | 首次访问时初始化 |
| `Mutex<T>` | 多线程 | 互斥访问（`std` 阻塞 / `tokio::sync` 异步） |
| `RwLock<T>` | 多线程 | 读多写少 |

```rust
use std::sync::Arc;
use tokio::sync::Mutex;   // ⚠️ 异步代码里用 tokio 的锁，lock 会 .await

struct Session {
    active_turn: Mutex<Option<ActiveTurn>>,
    services: Arc<SessionServices>,
}

impl Session {
    async fn take_active_turn(&self) -> Option<ActiveTurn> {
        self.active_turn.lock().await.take()
    }
}
```

⚠️ **不要跨 `.await` 持有 `std::sync::MutexGuard`**（会阻塞 executor / 触发 `Send` 约束错误）；tokio 的 `Mutex` 才允许持锁跨 await。

`RwLock<()>` 当"多读者-单写者闩锁"用（codex 工具并发控制）：

```rust
// codex-rs/core/src/tools/parallel.rs:41（字段在 :46）
pub(crate) struct ToolCallRuntime {
    ...
    parallel_execution: Arc<RwLock<()>>,   // 读锁=允许并发，写锁=独占
}
```

---

## 4. 结构体、枚举与模式匹配

### 4.1 struct 三种形态

```rust
struct User { name: String, age: u8 }        // 命名字段
struct Meters(f64);                          // newtype/tuple struct
struct Marker;                               // unit struct

let u = User { name: "a".into(), age: 1 };
let User { name, age } = &u;                 // 解构
let name = u.name;                           // 字段访问（注意 move/borrow）
```

方法用 `impl` 块；`self` 的形式决定 move/borrow：

```rust
impl User {
    fn new(name: String) -> Self { Self { name, age: 0 } }
    fn name(&self) -> &str { &self.name }        // 只读借用
    fn rename(&mut self, n: String) { self.name = n; }  // 可变借用
    fn into_name(self) -> String { self.name }   // 消费 self
}
```

💡 codex 的 `AgentPath` 是经典 newtype（`protocol/src/agent_path.rs:15`）：

```rust
pub struct AgentPath(String);        // 内部字段私有，防止非法路径

impl AgentPath {
    pub const ROOT: &str = "/root";
    pub fn root() -> Self { Self(Self::ROOT.to_string()) }
    pub fn from_string(path: String) -> Result<Self, String> { /* 校验后构造 */ }
    pub fn as_str(&self) -> &str { &self.0 }
}
```

### 4.2 enum：带数据的和类型

```rust
#[derive(Clone, Debug, PartialEq, Serialize, Deserialize)]
pub enum TurnInput {                                  // codex input_queue.rs:22
    UserInput { content: Vec<UserInput>, client_id: Option<String> },
    FunctionCallOutput(ResponseItem),
    ResponseItem(ResponseItemEnvelope),
    InterAgentCommunication(InterAgentCommunication),
}
```

枚举每个变体可以有不同的载荷（unit / tuple / struct 形式）。内存布局是 tag + 最大载荷。

### 4.3 `match`：穷尽 + 解构 + guard + 绑定

```rust
match event {
    ResponseEvent::OutputItemDone(item) => handle(item),
    ResponseEvent::OutputTextDelta(text) => stream(text),
    ResponseEvent::Completed { response_id, token_usage } => finish(response_id, token_usage),
    other => log(format!("{other:?}")),    // 或用具体变体列全，不用 `_`
}

// guard（条件分支）
match code {
    n if n < 0 => "negative",
    0 => "zero",
    _ => "positive",
}

// @ 绑定：既解构又保留整体
match age {
    n @ 0..=17 => println!("minor {n}"),
    n => println!("adult {n}"),
}

// 嵌套解构 + 借用
match &item {
    ResponseItem::Message { role, phase, .. } => {
        role == "assistant" && matches!(phase, Some(MessagePhase::Commentary))
    }
    ResponseItem::Reasoning { .. } => true,
    ...
}
```

💡 codex 风格（AGENTS.md）：**match 尽量穷尽、避免 `_` 通配**，这样新增变体时编译器会报错提醒你处理。

### 4.4 便捷模式语法

```rust
// if let
if let Some(x) = opt { use_it(x); }

// while let（消费迭代器）
while let Some(msg) = rx.recv().await { handle(msg); }

// let-else：不匹配就提前退出（适合 early return）
let Some(active_turn) = active_turn else {
    return false;
};

// let-chains（edition 2024）：多个条件串起来（codex turn.rs:2341）
if let Some((_, mut consumer)) = active_tool_argument_diff_consumer.take()
    && let Ok(Some(event)) = consumer.finish()
{
    sess.send_event(&turn_context, event).await;
}

// matches!：只判断形状
let aborted = matches!(reason, TurnAbortReason::Interrupted | TurnAbortReason::BudgetLimited);
```

⚠️ codex 源码要求：collapsible if —— 能 `a && b` 合并的嵌套 if 必须合并（见 §12）。

---

## 5. Trait 与泛型

### 5.1 trait 定义与默认方法

```rust
pub trait ToolExecutor<Invocation>: Send + Sync {
    fn tool_name(&self) -> ToolName;                       // 必须实现
    fn exposure(&self) -> ToolExposure { ToolExposure::Direct }  // 默认实现
    fn handle<'a>(&'a self, invocation: Invocation) -> ToolExecutorFuture<'a>
    where
        Invocation: 'a;
}
```

- `trait` 类似 interface，但可以带默认实现、关联类型、关联常量。
- `: Send + Sync` 是 supertrait 约束（实现者必须满足）。

### 5.2 泛型函数 / 结构体 / 方法

```rust
fn first<T: Clone>(v: &[T]) -> Option<T> {          // 泛型参数
    v.first().cloned()
}

struct Cache<K, V> { map: HashMap<K, V> }           // 泛型结构体

impl<K: Eq + Hash, V> Cache<K, V> {                 // impl 上带约束
    fn get(&self, k: &K) -> Option<&V> { self.map.get(k) }
}
```

`where` 子句用于复杂约束：

```rust
impl<T> AnySessionTask for T
where
    T: SessionTask,                                 // core/src/tasks/mod.rs:235
{
    ...
}
```

### 5.3 `impl Trait` 与 `dyn Trait`

```rust
// 参数位置：等价于泛型（静态分发 / monomorphization）
fn notify_all(handlers: impl Iterator<Item = Handler>) { ... }

// 返回位置：承诺返回某个实现该 trait 的具体类型（静态分发）
fn tasks(&self) -> impl Iterator<Item = &Task> { self.tasks.iter() }

// trait object：动态分发、类型擦除、可异构容器
let executors: Vec<Box<dyn ToolExecutor<ToolInvocation>>> = vec![...];
let guard: Arc<dyn AnySessionTask> = Arc::new(RegularTask::new());
```

| | `impl Trait` 返回 | `dyn Trait` |
|---|---|---|
| 分发 | 编译期静态 | 运行期 vtable |
| 能否异构集合 | 否 | 是 |
| 对象安全要求 | 无 | 方法不能带泛型、不能返回 `Self` 等 |

### 5.4 RPITIT：trait 方法返回 `impl Future`

Rust 1.75+ 支持 trait 里写 `-> impl Future`。codex 的 `SessionTask` 就是标准示例（`core/src/tasks/mod.rs:177-217`）：

```rust
pub(crate) trait SessionTask: Send + Sync + 'static {
    fn kind(&self) -> TaskKind;
    fn span_name(&self) -> &'static str;

    // 返回位置 impl Trait（RPITIT），比 #[async_trait] 零开销、更清晰
    fn run(
        self: Arc<Self>,
        session: Arc<Session>,
        ctx: Arc<TurnContext>,
        input: Vec<TurnInput>,
        cancellation_token: CancellationToken,
    ) -> impl std::future::Future<Output = SessionTaskResult> + Send;

    // 带默认实现的 RPITIT
    fn abort(&self, session: Arc<Session>, ctx: Arc<TurnContext>)
        -> impl std::future::Future<Output = ()> + Send
    {
        async move { let _ = (session, ctx); }   // 默认空实现
    }
}
```

实现方直接写 `async fn` 即可（自动满足契约）：

```rust
impl SessionTask for RegularTask {
    async fn run(self: Arc<Self>, sess: Arc<Session>, ctx: Arc<TurnContext>, ...)
        -> SessionTaskResult
    {
        ...
    }
}
```

⚠️ RPITIT 的 trait **不是 dyn-safe**。codex 用"手写擦除层"解决（`core/src/tasks/mod.rs:219-266`）：

```rust
pub(crate) trait AnySessionTask: Send + Sync + 'static {
    fn kind(&self) -> TaskKind;
    fn run(self: Arc<Self>, ...) -> BoxFuture<'static, SessionTaskResult>;  // 擦除
    fn abort<'a>(&'a self, ...) -> BoxFuture<'a, ()>;
}

// blanket impl：任何 SessionTask 自动获得 AnySessionTask
impl<T> AnySessionTask for T
where
    T: SessionTask,
{
    fn run(self: Arc<Self>, ...) -> BoxFuture<'static, SessionTaskResult> {
        Box::pin(SessionTask::run(self, ...))       // 把 RPITIT future 装箱
    }
}

// 使用：Arc<dyn AnySessionTask> 可以异构存进 RunningTask
let task: Arc<dyn AnySessionTask> = Arc::new(task);
```

💡 这就是 AGENTS.md 里"优先 RPITIT、避免 `#[async_trait]`"的落地方式：**接口用 RPITIT + 需要擦除时加一层手写 dyn 适配**。

### 5.5 关联类型、关联常量与 type alias

```rust
pub trait OrCancelExt: Sized {
    type Output;                                    // 关联类型
    fn or_cancel(self, token: &CancellationToken)
        -> impl Future<Output = Result<Self::Output, CancelErr>> + Send;
}

impl<F> OrCancelExt for F
where
    F: Future + Send,
    F::Output: Send,
{
    type Output = F::Output;                        // 投影输入 future 的输出
    async fn or_cancel(self, token: &CancellationToken) -> Result<Self::Output, CancelErr> {
        tokio::select! {
            _ = token.cancelled() => Err(CancelErr::Cancelled),
            res = self => Ok(res),
        }
    }
}
//（codex-rs/async-utils/src/lib.rs:9-31）
```

类型别名：

```rust
pub(crate) type SessionTaskResult = CodexResult<Option<String>>;
pub type ToolExecutorFuture<'a> =
    Pin<Box<dyn Future<Output = Result<Box<dyn ToolOutput>, FunctionCallError>> + Send + 'a>>;
```

### 5.6 derive：自动实现常见 trait

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash, Default, Serialize, Deserialize)]
pub struct AgentExecutionGuard { ... }
```

- `Debug`：`{:?}` 打印；**不要**给含密钥的类型 derive Debug。
- `Clone`：`.clone()`；`Copy`：赋值即拷贝（仅 POD）。
- `PartialEq/Eq/Hash`：比较与作为 HashMap key。
- `Default`：`T::default()`；`#[derive(Default)]` 用字段默认值，或用 `#[default]` 标记枚举变体：

```rust
#[derive(Default)]
pub enum AgentStatus {
    #[default]
    PendingInit,
    Running,
    ...
}
```

- 生态 derive：`Serialize/Deserialize`（serde）、`JsonSchema`（schemars）、`TS`（ts-rs 生成 TypeScript）、`Error`（thiserror）、`EnumDiscriminants`（strum）。

### 5.7 孤儿规则、newtype 与 From/Into

- 你只能为"自己 crate 的类型"或"覆盖到本地类型的 trait"实现外部 trait。
- newtype 是把外部类型包一层以获得实现权（也用于类型安全）：

```rust
pub struct Meters(f64);
impl From<f64> for Meters { fn from(v: f64) -> Self { Self(v) } }
// 自动获得 Into<Meters>
let m: Meters = 3.5.into();
```

- 实现 `FromIterator` 支持 `.collect()`：

```rust
impl FromIterator<ToolExposureSurface> for ToolExposures {   // tools/src/tool_executor.rs:41
    fn from_iter<T: IntoIterator<Item = ToolExposureSurface>>(surfaces: T) -> Self {
        surfaces.into_iter().fold(Self::NONE, |exposures, surface| exposures | surface.into())
    }
}
```

### 5.8 `Self` 与 self 接收者速查

| 写法 | 语义 |
|---|---|
| `self` | 按值消费 |
| `&self` | 不可变借用 |
| `&mut self` | 可变借用 |
| `self: Arc<Self>` | 任意 self 类型也可作接收者（codex `SessionTask::run`） |
| `Self`（返回） | 类型名简写 |
| `Self::new()` | 关联函数调用 |

### 5.9 运算符重载与 trait 边界

```rust
// 为类型实现标准 trait 即获得语法糖
impl From<ToolExposureSurface> for ToolExposures { ... }    // `.into()`
impl Iterator for MyIter { type Item = u8; fn next(&mut self) -> Option<u8> { ... } }  // for 循环
impl Display for MyErr { ... }                              // `{}`
```

---

## 6. 错误处理

### 6.1 panic vs Result

- `panic!` / `unwrap` / `expect`：不可恢复 bug（索引越界、断言失败）。
- `Result<T, E>`：可预期失败（IO、网络、解析）。

```rust
let n: i32 = s.parse().expect("input must be int");   // 测试/原型 OK
let n: i32 = s.parse()?;                              // 库代码用 ?
```

### 6.2 `?` 与自动 `From` 转换

`?` 在 Err 时提前返回，并自动调用 `From::from` 做错误类型转换：

```rust
fn load() -> Result<Config, CodexErr> {
    let text = std::fs::read_to_string(path)?;        // io::Error -> CodexErr（有 From impl）
    let cfg = toml::from_str(&text)?;                 // toml::de::Error -> CodexErr
    Ok(cfg)
}
```

### 6.3 thiserror：声明式错误类型

```rust
use thiserror::Error;

#[derive(Error, Debug, EnumDiscriminants)]     // protocol/src/error.rs:77
pub enum CodexErr {
    #[error("turn aborted")]
    TurnAborted,
    #[error("stream disconnected: {0}")]
    Stream(String),
    #[error("io error: {0}")]
    Io(#[from] std::io::Error),                // #[from] 自动生成 From<io::Error>
    ...
}
// EnumDiscriminants 生成 CodexErrorDetails：不带载荷的穷尽判别枚举，
// 便于 `matches!(err.details(), CodexErrorDetails::TurnAborted)`
```

匹配错误时 codex 用 `details()` 而不是直接匹配带载荷的变体：

```rust
Err(err) if matches!(err.details(), CodexErrorDetails::TurnAborted) => { ... }
```

### 6.4 `Result` 类型别名与自定义 Result

```rust
pub type Result<T> = std::result::Result<T, CodexErr>;
// 之后函数签名写 -> Result<Foo> 即含自定义错误
```

### 6.5 错误传播的工程习惯

- 库层：定义精确错误枚举（thiserror），暴露 `details()` 判别。
- 应用层：`anyhow` + `.context("...")` 补充上下文，打印时 `{e:#}` 展开链（codex 里 `info!("Turn error: {e:#}")`）。
- 对外协议：`to_codex_protocol_error()` 把内部错误映射成客户端可见的 `CodexErrorInfo`。

---

## 7. 智能指针与所有权模型延伸

| 类型 | 所有权 | 线程安全 | 典型用途 |
|---|---|---|---|
| `Box<T>` | 独占、堆 | `Send` 取决于 T | 递归类型、trait object、减小栈占用 |
| `Rc<T>` | 共享、计数 | ❌ | 单线程共享 |
| `Arc<T>` | 共享、原子计数 | ✅ | 多线程/async 共享 |
| `Weak<T>` | 不增加强计数 | ✅（配 Arc） | 打破循环引用 |
| `Cow<'a, T>` | 借用或拥有 | | 必要时才克隆 |
| `Pin<P>` | 固定地址 | | 自引用 / `dyn Future` |

### 7.1 `Arc`：跨任务共享的事实标准

```rust
let session = Arc::new(session);
let s2 = Arc::clone(&session);              // 克隆的是指针，不是数据
tokio::spawn(async move { s2.run().await });
use_it(&session);                           // 原 Arc 仍可用
```

⚠️ 只要共享就别 `clone()` 数据本身。注意用 `Arc::clone(&x)` 而非 `x.clone()`（意图更明确，避免无意深拷贝）。

### 7.2 `Weak`：打破环

`AgentControl` 持有 `Weak<ThreadManagerState>`，因为 `ThreadManagerState -> CodexThread -> Session -> SessionServices -> AgentControl` 是环（`core/src/agent/control.rs:117-131`）：

```rust
pub(crate) struct AgentControl {
    manager: Weak<ThreadManagerState>,     // 不是 Arc！
    ...
}
// 使用时升级
let Some(manager) = self.manager.upgrade() else { return; };
```

### 7.3 `Pin` 与 `BoxFuture`

异步块编译成自引用状态机，移动会破坏内部指针 ⇒ 必须 `Pin`。类型别名简化签名：

```rust
use futures::future::BoxFuture;
fn run(...) -> BoxFuture<'static, SessionTaskResult> {
    Box::pin(async move { ... })          // 装箱 + pin，获得 dyn 兼容的类型
}
```

### 7.4 `Cow`：写时克隆

```rust
use std::borrow::Cow;
fn normalize(s: &str) -> Cow<'_, str> {
    if s.contains(' ') { Cow::Owned(s.replace(' ', "_")) } else { Cow::Borrowed(s) }
}
```

我全文里没找到 codex 直接用 Cow 的例子，但 serde 反序列化时大量内部使用。这里作为通用知识保留。

### 7.5 `PhantomData`：在结构体里"声明"生命周期/类型参数

```rust
// codex-rs/tools/src/tool_call.rs:75
pub struct ToolCall<'call> {
    ...
    pub _lifetime: PhantomData<&'call ()>,   // 结构体实际不存引用，但语义上借用 'call
}
```

用途：让编译器把生命周期参数视为"正在使用"，从而正确检查借用而不需要真实字段。

---

## 8. 并发与异步

### 8.1 概念对照

| 概念 | 同步线程 | 异步任务 |
|---|---|---|
| 启动 | `std::thread::spawn` | `tokio::spawn` |
| 单位 | OS 线程 | `Future` 状态机，executor 调度 |
| 等待 | `join()` | `.await` |
| 阻塞调用 | OK | 禁止（用 `spawn_blocking`） |
| 共享 | `Arc<Mutex<T>>` | `Arc<tokio::sync::Mutex<T>>` 或 `RwLock` |

codex 几乎全是 async：tokio runtime + `tokio::spawn` + `CancellationToken`。

### 8.2 `spawn` 与 `'static`

```rust
tokio::spawn(async move {          // async move 捕获所有权
    let r = task.run(...).await;
    // ...
});
```

⚠️ spawn 的 future 必须 `Send + 'static`：捕获的引用必须换成 `Arc`/owned 数据。这就是 codex 里 `Arc<Session>` 满天飞的原因。

### 8.3 常用同步原语

```rust
// 异步互斥锁
let guard = self.active_turn.lock().await;

// Semaphore：限制并发
let permit = semaphore.clone().acquire_owned().await?;

// Notify：事件通知（无值）
done.notify_waiters();          // 唤醒所有等待者
done.notified().await;          // 等待

// oneshot：单值一次性回执
let (tx, rx) = tokio::sync::oneshot::channel();
```

### 8.4 channel 选型

| 类型 | 语义 | codex 用例 |
|---|---|---|
| `mpsc` | 多生产者单消费者队列 | Op 提交、事件流 |
| `oneshot` | 一次性回执 | 请求-响应配对 |
| `watch` | 最新值广播、可 `changed().await` | `InputQueueActivity` 变更信号 |
| `broadcast` | 多消费者广播 | 事件总线（上游） |

```rust
// watch 示例（core/src/session/input_queue.rs:92,135）
let (activity_tx, _) = watch::channel(InputQueueActivity::Mailbox);
activity_tx.send_replace(InputQueueActivity::Mailbox);   // 更新值并通知

let (mut rx, pending) = input_queue.subscribe_activity(Some(turn_state)).await;
rx.changed().await?;                                     // 等下一次变化
```

### 8.5 `select!`：多路等待

```rust
tokio::select! {
    _ = token.cancelled() => Err(CancelErr::Cancelled),   // 取消优先
    res = self => Ok(res),                                // 正常完成
}
```

```rust
// timeout 组合
let result = tokio::time::timeout(Duration::from_secs(5), rx.recv()).await;
```

⚠️ `select!` 默认随机轮询分支；有公平性/优先级需求时用 `biased;` 或如上例把取消放前面。

### 8.6 取消模型：`CancellationToken`

tokio-util 提供的层级化取消信号：

```rust
let token = CancellationToken::new();
let child = token.child_token();      // 子 token：父取消时子也取消
token.cancel();                       // 通知所有克隆/子 token
token.is_cancelled();
token.cancelled().await;              // 异步等待
```

codex 把 `CancellationToken` 贯穿整个 turn/tool 栈（abort 时 cancel，任务内 `or_cancel` 快速退出）。

### 8.7 RAII 卫士

- `AbortOnDropHandle`：drop 即 abort 任务，防止"孤儿任务"泄漏（`tasks/mod.rs:403`）。
- `GaugeGuard` / `_diagnostics_guard`：drop 时自动更新指标。
- `Permit`：drop 归还配额。

```rust
let running_task = RunningTask {
    handle: AbortOnDropHandle::new(handle),
    cancellation_token,
    _agent_execution_guard: agent_execution_guard,   // drop 释放执行配额
    _diagnostics_guard: ACTIVE_TURNS.track(),
    ...
};
```

💡 这是 Rust 的核心工程模式：**把资源生命周期编码进类型系统**，不用手动 try/finally。

### 8.8 手动实现 `Future`（进阶）

一般不用手写；理解以下即可：

```rust
pub struct ResponseStream { ... }
impl futures::Stream for ResponseStream {
    type Item = Result<ResponseEvent, CodexErr>;
    // poll_next 由 SSE 字节流解析驱动
}
```

codex 的模型流是 `Stream<Item = Result<ResponseEvent, CodexErr>>`，在 `stream_response` 里逐条 `next().await`。

---

## 9. 宏与元编程

### 9.1 `macro_rules!`（声明宏）

```rust
// core/src/util.rs:28 —— 把键值对统一打进 tracing 的 feedback_tags target
#[macro_export]
macro_rules! feedback_tags {
    ($( $key:ident = $value:expr ),+ $(,)?) => {   // $(...),+ 重复展开
        ::tracing::info!(
            target: "feedback_tags",
            $( $key = ::tracing::field::debug(&$value) ),+
        );
    };
}
```

调用：

```rust
codex_core::feedback_tags!(model = "gpt-5", cached = true);
codex_core::feedback_tags!(provider = provider_id, request_id = request_id);
```

### 9.2 常用宏速查

| 宏 | 作用 |
|---|---|
| `vec![a, b]` | 构造 `Vec` |
| `format!("{name}")` | 构造 `String` |
| `println!/eprintln!` | 标准输出 |
| `matches!(x, P)` | 模式判断为 bool |
| `assert!/assert_eq!` | 断言 |
| `unreachable!(...)` | 标记不可达分支（panic） |
| `debug_assert!` | 仅 debug 构建的断言 |
| `include_str!("file")` | 编译期把文件内容变成 `&'static str` |
| `todo!/unimplemented!` | 占位 |

```rust
let names: &str = include_str!("agent_names.txt");   // 编译期嵌入
```

### 9.3 derive 宏与属性宏

- `#[derive(...)]`：编译期生成 impl（Clone/Debug/Serialize...）。
- 属性宏：`#[tokio::test]`、`#[tracing::instrument]`、`#[serde(...)]`。

```rust
#[tokio::test]                       // 生成 tokio runtime + main
async fn spawns_and_cancels() { ... }

#[instrument(level = "trace", skip_all)]   // 自动 span + 参数记录
pub(crate) fn handle_tool_call(...) -> ... { ... }
```

---

## 10. 属性（Attributes）与条件编译

```rust
#[derive(Debug)]
#[serde(rename_all = "snake_case")]
#[ts(rename_all = "snake_case")]
pub enum NonSteerableTurnKind { Review, Compact }

#[cfg(test)]                          // 仅测试构建
mod tests;

#[cfg(feature = "experimental")]      // cargo feature 开关
fn preview_api() {}

#[cfg_attr(windows, path = "win.rs")] // 条件属性
#[cfg_attr(not(windows), path = "unix.rs")]
mod os;

#[allow(dead_code)]                   // 豁免 lint（新代码更推荐 #[expect(...)])
#[expect(unused_variables, reason = "placeholder")]
fn stub(_x: i32) {}
```

常用内置属性：`#[inline]`、`#[must_use]`、`#[non_exhaustive]`、`#[deprecated]`、`#[allow/deny/warn/expect(lint)]`。

`#[non_exhaustive]`：外部 crate 不能穷尽匹配该 enum/struct，便于向后兼容地扩展：

```rust
#[non_exhaustive]
pub enum ToolExposure { Direct, Deferred, /* 未来可加 */ }
```

---

## 11. serde 序列化惯用法

### 11.1 基本形态

```rust
use serde::{Deserialize, Serialize};

#[derive(Serialize, Deserialize)]
#[serde(rename_all = "camelCase")]          // 线上字段名风格
pub struct SpawnParams {
    pub agent_name: String,                 // -> "agentName"
    pub fork_turns: Option<ForkTurns>,      // 默认 None 语义
}
```

### 11.2 标签联合（内部/相邻/无标签）

```rust
// 内部标签：{"type": "message", "message": "..."}
#[derive(Serialize, Deserialize)]
#[serde(tag = "type", rename_all = "snake_case")]
pub enum ClientOp { Message { text: String }, Interrupt }

// 相邻标签：{"t":"Message","c":{...}}
#[serde(tag = "t", content = "c")]

// 无标签：按字段形状尝试解析
#[serde(untagged)]
```

### 11.3 常见字段属性

```rust
#[serde(default)]                                  // 缺省时用 Default
pub other_recipients: Vec<AgentPath>,

#[serde(skip_serializing_if = "Option::is_none")]  // None 不输出
pub id: Option<ResponseItemId>,

#[serde(rename = "traceId")]
pub trace_id: String,

#[serde(flatten)]
pub extra: HashMap<String, Value>,
```

### 11.4 自定义序列化模块（进阶）

`TurnInput` 想在内存里携带 envelope 元数据，但线上保持旧格式，于是用 `serde(with = ...)`（`core/src/session/input_queue.rs:30-64`）：

```rust
pub enum TurnInput {
    ...
    ResponseItem(#[serde(with = "turn_input_response_item")] ResponseItemEnvelope),
}

mod turn_input_response_item {
    pub(super) fn serialize<S: Serializer>(item: &ResponseItemEnvelope, s: S) -> Result<S::Ok, S::Error> {
        if item.metadata.is_some() {
            return Err(serde::ser::Error::custom("annotated response items cannot cross this boundary"));
        }
        item.item.serialize(s)                    // 只序列化内部 item，保持旧格式
    }
    pub(super) fn deserialize<'de, D: Deserializer<'de>>(d: D) -> Result<ResponseItemEnvelope, D::Error> {
        ResponseItem::deserialize(d).map(ResponseItemEnvelope::new)
    }
}
```

核心要点：`Serialize`/`Deserializer` 的 trait 约束、返回 `Result<_, S::Error>`、`serde::ser::Error::custom`。

### 11.5 ts-rs / schemars

```rust
#[derive(Serialize, Deserialize, JsonSchema, TS)]
#[ts(export_to = "v2/")]        // 生成 TypeScript 到指定目录
pub struct ThreadReadParams { ... }
```

---

## 12. codex 代码风格约定（AGENTS.md 摘要）

来自仓库 `AGENTS.md`，写 Rust 时逐条对齐：

1. **格式**：`just fmt`；crate 名 `codex-` 前缀。
2. **内联 format 变量**：`format!("{name}")` 而不是 `format!("{}", name)`。
3. **合并嵌套 if**（clippy `collapsible_if`）：`if cond1 && cond2 {}`。
4. **方法引用优于闭包**（`redundant_closure_for_method_calls`）：`.map(str::to_string)` 而非 `.map(|s| s.to_string())`。
5. **避免 bool/裸 Option 位置参数**：`foo(false)` 难读，用枚举/具名方法；必要时加 `/*param_name*/` 注释：
   ```rust
   sess.send_event(/* prefix */ None, event)   // 少见但存在
   inject(/*before_user_prompt*/ true)
   ```
6. **match 穷尽**，避免 `_` 通配。
7. **trait 设计**：优先 RPITIT（`-> impl Future + Send`），不用 `#[async_trait]`、不用 `#[allow(async_fn_in_trait)]`；需要 dyn 时手写擦除层。
8. **不要单引用 helper 方法**（只在调用一次的小方法直接内联逻辑）。
9. **模块大小**：目标 <500 LoC，>800 LoC 必须重建模块；禁止把新功能堆进 `tui/src/app.rs` / `chatwidget.rs` 这类高冲突文件。
10. **tracing**：给函数/方法加 `#[tracing::instrument]`，不要在调用点 `.instrument(...)`。
11. **测试**：`#[cfg(test)] #[path = "..._tests.rs"] mod tests;`；`pretty_assertions::assert_eq`；整体对象深比较；集成测试优先（core/suite 的 `test_codex`）。
12. **config schema**：改 `ConfigToml` 后运行 `just write-config-schema`。
13. **依赖变更**：改 Cargo.toml/lock 后运行 `just bazel-lock-update`。

---

## 13. 综合示例：一个 mini agent loop

把本文语法点串起来 —— 一个简化版的 agent turn 循环（结构对照 `core/src/session/turn.rs`）：

```rust
use std::sync::Arc;
use tokio_util::sync::CancellationToken;
use tokio::sync::Mutex;

// 1) trait + RPITIT：任务契约
trait Task: Send + Sync + 'static {
    fn kind(&self) -> TaskKind;
    fn run(
        self: Arc<Self>,
        session: Arc<Session>,
        cancellation_token: CancellationToken,
    ) -> impl std::future::Future<Output = Result<Option<String>, TaskError>> + Send;
}

// 2) 带数据的枚举：统一的输入/事件
enum TurnInput {
    UserInput { text: String, client_id: Option<String> },
    ToolOutput { call_id: String, output: String },
}

enum ResponseEvent {
    TextDelta(String),
    ToolCall { call_id: String, name: String, args: String },
    Completed { response_id: String },
}

// 3) 共享状态：Arc + Mutex + 取消
struct Session {
    history: Mutex<Vec<String>>,
    pending_input: Mutex<Vec<TurnInput>>,
}

impl Session {
    async fn run_turn(
        self: &Arc<Self>,
        input: Vec<TurnInput>,
        token: &CancellationToken,
    ) -> Result<Option<String>, TaskError> {
        let mut next_input = input;
        let mut last_message = None;

        // 4) 中层循环：一个 turn 内可能因 tool call / steer 多次采样
        loop {
            for item in next_input.drain(..) {
                self.history.lock().await.push(render(item));
            }

            // 5) 单次采样：流式事件循环
            let mut follow_up = false;
            let mut stream = call_model(self).await?;
            while let Some(event) = stream.next_event(token).await? {
                match event {
                    ResponseEvent::TextDelta(text) => {
                        println!("{text}");              // 流式输出
                        last_message = Some(text);
                    }
                    ResponseEvent::ToolCall { call_id, name, args } => {
                        let output = execute_tool(&name, &args).await;   // 假设并发执行
                        next_input.push(TurnInput::ToolOutput { call_id, output });
                        follow_up = true;                // 6) 有工具调用 → 需要下一轮
                    }
                    ResponseEvent::Completed { .. } => break,
                }
            }

            // 7) steer 汇合点：采样结束才有机会消费 pending input
            let pending = self.pending_input.lock().await.drain(..).collect::<Vec<_>>();
            let has_pending = !pending.is_empty();

            if follow_up {
                next_input.extend(pending);              // 与 tool 输出一起进下一轮
                continue;
            }
            if has_pending {
                next_input = pending;                    // 用户插入消息 → 再跑一轮
                continue;
            }
            return Ok(last_message);                     // 8) 真正结束
        }
    }
}

// 9) 泛型 + let-else + `?` 传播
async fn spawn_task<T: Task>(task: T, session: Arc<Session>, token: CancellationToken) {
    let Ok(()) = check_capacity().await else { return };
    tokio::spawn(async move {
        let _ = task.run(session, token).await;      // 忽略结果仅为演示
    });
}
```

对照真实 codex 的差异：
- 真实循环三层：`RegularTask::run`（pending input 重新起 turn）→ `run_turn`（采样边界）→ `try_run_sampling_request`（SSE 事件 + 工具 Future）。
- 真实工具执行是 `FuturesOrdered` 并发 + `RwLock<()>` 并发闩锁，非串行。
- 真实误差处理全部走 `CodexErr` + `CodexErrorDetails`。

---

## 14. 常见编译错误速查

| 错误 | 含义 | 解法 |
|---|---|---|
| E0382 use of moved value | 值已被移动 | `.clone()`、传引用、调整所有权顺序 |
| E0502 cannot borrow as mutable | 已有不可变借用 | 缩短借用作用域（NLL）、先收集结果再改 |
| E0499 two mutable borrows | 同时两个可变借用 | 用索引/`split_at_mut`/重构 |
| E0106 missing lifetime | 返回引用但无法推导 | 加 `<'a>` 标注 |
| E0277 trait bound not satisfied | 缺 trait 实现 | 加 derive/手写 impl/检查 `Send` 约束 |
| E0599 no method found | 方法不存在/未导入 trait | 导入 trait、检查解引用层级 |
| E0433/0412 unresolved path | 模块/名称不对 | 检查 `use`、可见性、crate 名连字符 |
| future is not `Send` | 跨 await 持有了非 Send 值 | 换 tokio 锁、缩短 guard 生命周期 |
| 循环引用导致内存泄漏 | `Arc` 成环 | 一边换 `Weak`（见 §7.2） |

---

## 附：延伸阅读（codex 内可直接当教材读的文件）

| 文件 | 可学内容 |
|---|---|
| `codex-rs/async-utils/src/lib.rs` | trait 扩展、关联类型、select!、测试写法 |
| `codex-rs/tools/src/tool_executor.rs` | bitflags、trait 对象、RPITIT 前身、FromIterator |
| `codex-rs/core/src/tasks/mod.rs` | RPITIT + dyn 擦除层、RAII 卫士、spawn 生命周期 |
| `codex-rs/core/src/session/turn.rs` | 真实 agent loop、let-chains、错误分支穷尽 |
| `codex-rs/protocol/src/error.rs` | thiserror + EnumDiscriminants |
| `codex-rs/protocol/src/agent_path.rs` | newtype、常量、校验构造 |
| `codex-rs/core/src/session/input_queue.rs` | watch、VecDeque、serde 自定义模块、迭代器链 |
| `codex-rs/core/src/agent/control.rs` | Weak 破环、Clone 共享控制面 |
