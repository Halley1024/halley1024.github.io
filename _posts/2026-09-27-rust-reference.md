---
layout: post
title: Rust智能指针详解
date: 2026-09-26 20:49 +0800
categories: [Blogs,Rust]
tags: [Blogs]     # TAG names should always be lowercase
math: true
mermaid: true
---

# Rust 智能指针详解

智能指针可以管理值的所有权、位置或生命周期。这里也一并介绍常与智能指针配合的内部可变性和同步容器：`Cell<T>`、`RefCell<T>`、`Mutex<T>`、`RwLock<T>`；它们本身并非严格意义上的指针。下列代码块各自独立，可分别保存为 `main.rs` 运行。

| 类型 | 主要用途 | 典型组合 |
| --- | --- | --- |
| `Box<T>` | 堆分配、递归类型、特征对象 | `Box<dyn Trait>` |
| `Rc<T>` | 单线程共享所有权 | `Rc<RefCell<T>>` |
| `Arc<T>` | 多线程共享所有权 | `Arc<Mutex<T>>` |
| `RefCell<T>` | 单线程运行时借用检查 | `Rc<RefCell<T>>` |
| `Mutex<T>` | 互斥读写 | `Arc<Mutex<T>>` |
| `RwLock<T>` | 多读者或单写者 | `Arc<RwLock<T>>` |
| `Weak<T>` | 不延长生命周期的回指 | `Rc::downgrade` / `Arc::downgrade` |
| `Cell<T>` | 单线程复制或替换内部值 | `Cell<u32>` |

## 001 · `Box<T>`：在堆上拥有值

**用途：** `Box<T>` 拥有堆上的 `T`。常用于让递归类型具有确定大小、持有 `dyn Trait` 特征对象，或转移较大值的所有权而不搬动值本身。

**典型使用场景：**

- **递归数据结构：** 链表的下一个节点、语法树的子表达式、树节点的子节点等需要间接层，否则类型大小无法确定。例如 `enum Expr { Add(Box<Expr>, Box<Expr>), Number(i32) }`。
- **统一存放不同的具体类型：** 一组实现同一特征的对象需要放入同一个集合，且每个对象由集合独占时，可使用 `Vec<Box<dyn Trait>>`。
- **为较大值提供固定大小的所有权句柄：** 需要把值放进不同的容器或在函数之间转移所有权，又希望移动的是指针而非值本身时，可以考虑装箱；是否划算要看分配成本和实际值大小。

如果只是临时读取现有值，优先借用 `&T`；如果需要多个所有者，考虑 `Rc<T>` 或 `Arc<T>`。

**用法：** 使用 `Box::new(value)` 创建，使用 `*` 解引用。递归枚举中的 `Box` 为下一层提供固定大小的间接层。

```rust
enum List {
    Cons(i32, Box<List>),
    Nil,
}

fn sum(list: &List) -> i32 {
    match list {
        List::Cons(value, next) => value + sum(next),
        List::Nil => 0,
    }
}

fn main() {
    let list = List::Cons(1, Box::new(List::Cons(2, Box::new(List::Nil))));
    assert_eq!(sum(&list), 3);
}
```

**常见操作：**

| 调用 | 作用与用法 |
| --- | --- |
| `Box::new(value)` | 把 `value` 放到堆上，返回拥有它的 `Box<T>`。 |
| `*boxed`、`&*boxed` | 分别访问内部值、借用内部值；`boxed` 可变时还可通过 `*boxed` 修改。 |
| `Box::pin(value)` | 创建 `Pin<Box<T>>`，用于需要固定值位置的场景，例如某些异步任务或自引用类型。 |
| `Box::leak(boxed)` | 消耗 `Box` 并返回长期有效的 `&mut T`；此后不会自动释放该分配，适合确实要把值保留到程序结束的少数场景。 |
| `Box::into_raw(boxed)` | 消耗 `Box` 并交出原始指针，主要用于 FFI；要避免泄漏，须在确保所有权唯一且指针有效的前提下用 `unsafe { Box::from_raw(ptr) }` 接回。 |

**注意事项：** `Box<T>` 只有一个所有者；移动 `Box` 会转移所有权，不会深拷贝 `T`。它不提供共享所有权或内部可变性。堆分配有成本，普通小值通常无需装箱。能否跨线程传递，仍取决于 `T` 的 `Send` / `Sync` 性质。

## 002 · `Rc<T>`：单线程共享所有权

**用途：** 让多个位置共同拥有同一个值，例如单线程的图节点。`Rc` 使用非原子的强引用计数；最后一个强引用销毁时，内部值被销毁。

**典型使用场景：**

- **单线程共享只读配置：** 多个界面组件或处理器长期持有同一份配置，且它们的生命周期不方便由一个借用统一管理。
- **共享的数据结构节点：** 多条路径指向同一个图节点，或持久化数据结构的新旧版本复用未改变的节点。
- **多个对象共同持有资源：** 例如事件回调与界面状态共同持有一个对象，并希望最后一个持有者退出时自动释放。

仅仅需要短暂访问时，普通引用更简单；需要跨线程共享时改用 `Arc<T>`。如果共享节点需要修改内部状态，可组合 `Rc<RefCell<T>>`，并留意引用环。

**用法：** `Rc::clone(&value)` 创建指向同一分配的新所有者。

```rust
use std::rc::Rc;

fn main() {
    let shared = Rc::new(String::from("Rust"));
    let first = Rc::clone(&shared);
    let second = Rc::clone(&shared);
    assert_eq!(first.as_str(), "Rust");
    assert_eq!(second.as_str(), "Rust");
    assert_eq!(Rc::strong_count(&shared), 3);
}
```

**常见操作：**

| 调用 | 作用与用法 |
| --- | --- |
| `Rc::new(value)` | 创建第一个强所有者。 |
| `Rc::clone(&shared)` | 增加强引用计数，返回指向同一值的另一个 `Rc`，不会深拷贝 `T`。 |
| `Rc::strong_count(&shared)` | 查看当前强引用数量，主要用于理解或诊断所有权；不要用它代替所有权设计。 |
| `Rc::downgrade(&shared)` | 创建不延长内部值生命周期的 `Weak<T>`。 |
| `Rc::ptr_eq(&a, &b)` | 判断两个 `Rc` 是否指向同一份分配，而不是比较内部值是否相等。 |
| `Rc::get_mut(&mut shared)` / `Rc::make_mut(&mut shared)` | 前者只在没有其他强、弱引用时返回 `Some(&mut T)`；后者在有其他强引用时克隆内部值（`T: Clone`），仅有弱引用时则与弱引用分离，提供写时复制。 |

**注意事项：** `Rc::clone` 只增加计数，不深拷贝 `T`。`Rc<T>` 不能跨线程共享，也不能仅凭共享的 `Rc` 取得 `&mut T`；单线程共享修改通常使用 `Rc<RefCell<T>>`。相互持有强 `Rc` 会形成引用环并泄漏内部值，回指应使用 `Weak`。

## 003 · `Arc<T>`：多线程共享所有权

**用途：** 与 `Rc<T>` 类似，但引用计数使用原子操作，适合跨线程共享所有权。

**典型使用场景：**

- **多个工作线程共享只读数据：** 例如路由表、词典、模型参数或其他创建后基本不变的数据；每个线程持有一个 `Arc`。
- **后台任务共同持有服务状态：** 多个任务的生命周期各不相同，但都需要保证同一份状态在任务结束前有效。
- **跨线程共享可变状态：** 将 `Arc` 与 `Mutex`、`RwLock` 或原子类型组合，分别负责所有权与并发访问控制。

如果只在线程内共享，`Rc<T>` 通常更合适；如果每个线程只需一份独立数据，直接移动或克隆数据可能比共享状态更简单。

**用法：** 每个线程持有一个 `Arc::clone`。只读共享时不需要锁。

```rust
use std::sync::Arc;
use std::thread;

fn main() {
    let message = Arc::new(String::from("hello"));
    let worker_message = Arc::clone(&message);
    let worker = thread::spawn(move || {
        assert_eq!(worker_message.as_str(), "hello");
    });
    worker.join().unwrap();
    assert_eq!(message.as_str(), "hello");
}
```

**常见操作：**

| 调用 | 作用与用法 |
| --- | --- |
| `Arc::new(value)` | 创建第一个可供线程共享的强所有者。 |
| `Arc::clone(&shared)` | 增加强引用计数，常在把句柄移入新线程前调用。 |
| `Arc::downgrade(&shared)` | 创建 `std::sync::Weak<T>`，用于跨线程场景中的非拥有关系。 |
| `Arc::strong_count(&shared)` | 读取某一时刻的强引用数量；其他线程可能立即改变它，不能据此做同步决策。 |
| `Arc::ptr_eq(&a, &b)` | 判断是否共享同一份分配。 |
| `Arc::get_mut(&mut shared)` / `Arc::make_mut(&mut shared)` | 前者仅在没有其他强、弱引用时取得 `&mut T`；后者在有其他强引用时克隆内部值（`T: Clone`），仅有弱引用时则与弱引用分离。 |

**注意事项：** `Arc` 只保证引用计数的线程安全，不会自动让内部 `T` 线程安全；跨线程共享通常要求 `T: Send + Sync`。共享修改可使用 `Arc<Mutex<T>>`、`Arc<RwLock<T>>` 或原子类型。单线程优先考虑开销较低的 `Rc<T>`。强引用环也可能泄漏，可用 `std::sync::Weak` 打破。

## 004 · `RefCell<T>`：单线程运行时借用检查

**用途：** 当编译器无法从代码结构证明内部修改安全，而程序逻辑能保证借用规则时，允许通过共享引用修改内部值。常与 `Rc<T>` 组合。

**典型使用场景：**

- **单线程图或树的可变节点：** 多个 `Rc` 指向同一节点，需要在运行时更新其子节点、属性或缓存。
- **对象接口只有 `&self`，但需要更新内部状态：** 例如记录调用次数、维护惰性计算结果或测试替身记录收到的调用；内部数据需要借用并原地修改。
- **借用关系取决于运行时流程：** 程序能保证读写不会重叠，但编译器无法仅靠静态分析证明。

若修改的是简单 `Copy` 值，先考虑 `Cell<T>`；若可以通过 `&mut self` 完成修改，普通可变字段更直接。`RefCell<T>` 不替代多线程锁。

**用法：** `borrow()` 返回只读守卫 `Ref<T>`，`borrow_mut()` 返回可变守卫 `RefMut<T>`；守卫离开作用域后，借用结束。

```rust
use std::cell::RefCell;
use std::rc::Rc;

fn main() {
    let shared = Rc::new(RefCell::new(vec![1]));
    let other_owner = Rc::clone(&shared);
    other_owner.borrow_mut().push(2);
    assert_eq!(*shared.borrow(), vec![1, 2]);
}
```

**常见操作：**

| 调用 | 作用与用法 |
| --- | --- |
| `RefCell::new(value)` | 创建在运行时检查借用规则的容器。 |
| `cell.borrow()` / `cell.borrow_mut()` | 分别取得只读、可变守卫；借用冲突时会 panic。 |
| `cell.try_borrow()` / `cell.try_borrow_mut()` | 尝试借用，冲突时返回 `Err`，适合需要自行处理失败的代码。 |
| `cell.replace(new_value)` | 用新值替换并返回旧值；当前有活跃借用时会 panic。 |
| `cell.get_mut()` | 已拥有 `&mut RefCell<T>` 时直接取得 `&mut T`，无需运行时借用检查。 |
| `cell.into_inner()` | 消耗容器，取回内部的 `T`。 |

**注意事项：** 借用规则仍然存在，只是改在运行时检查：同一时刻只能有一个可变借用，或多个只读借用。冲突时 `borrow()` / `borrow_mut()` 会 panic；需要处理失败时使用 `try_borrow()` / `try_borrow_mut()`。尽量缩小守卫的作用域。`RefCell<T>` 不能用于跨线程共享修改。

## 005 · `Mutex<T>`：互斥访问

**用途：** 同一时刻只允许一个线程访问受保护的数据。跨线程共享时常用 `Arc<Mutex<T>>`：`Arc` 管所有权，`Mutex` 管同步。

**典型使用场景：**

- **多个线程更新同一份复合状态：** 例如共享队列、统计汇总或若干必须一起更新的字段；整个更新过程需要保持一致。
- **读写都较频繁，或临界区很短：** 不需要区分读锁和写锁时，`Mutex` 通常是清晰的起点。
- **一个线程拥有数据，但多个线程需要受控访问：** 使用 `Arc<Mutex<T>>` 为每个线程提供所有权句柄，并通过锁串行访问。

如果只是单个整数计数，可以考虑原子类型；如果读取远多于写入且读操作值得并行化，再考虑 `RwLock<T>`。若能通过消息传递让单个线程独占数据，也可避免共享锁。

**用法：** `lock()` 返回 `LockResult<MutexGuard<T>>`；守卫可解引用以读写数据，销毁时自动解锁。

```rust
use std::sync::{Arc, Mutex};
use std::thread;

fn main() {
    let counter = Arc::new(Mutex::new(0));
    let mut workers = Vec::new();
    for _ in 0..4 {
        let counter = Arc::clone(&counter);
        workers.push(thread::spawn(move || {
            let mut value = counter.lock().unwrap();
            *value += 1;
        }));
    }
    for worker in workers {
        worker.join().unwrap();
    }
    assert_eq!(*counter.lock().unwrap(), 4);
}
```

**常见操作：**

| 调用 | 作用与用法 |
| --- | --- |
| `Mutex::new(value)` | 创建保护 `value` 的互斥锁。 |
| `mutex.lock()` | 阻塞直到取得锁，返回 `LockResult<MutexGuard<T>>`；守卫离开作用域后自动解锁。 |
| `mutex.try_lock()` | 不等待锁，立即返回守卫或 `WouldBlock` / `Poisoned` 错误。 |
| `mutex.is_poisoned()` | 查看锁目前是否中毒；并发线程可能随后改变状态，不能据此保证数据一定一致。 |
| `mutex.get_mut()` | 已拥有 `&mut Mutex<T>` 时直接访问内部值，返回 `LockResult<&mut T>`，无需锁定。 |
| `mutex.into_inner()` | 消耗锁并取出 `T`，仍需处理可能的中毒错误。 |

**注意事项：** `lock()` 可能阻塞；`try_lock()` 可非阻塞尝试。持锁线程 panic 时锁通常会中毒，`lock()` 返回 `Err(PoisonError)`；应根据数据是否仍一致决定恢复或传播错误。缩短持锁时间，避免重复锁定同一个锁或以不一致顺序取得多个锁，以免死锁。普通 `std::sync::Mutex` 的守卫不应跨异步 `.await` 持有。

## 006 · `RwLock<T>`：多读者、单写者

**用途：** 允许多个读者同时读取，或一个写者独占访问。适合读多写少的共享数据；是否比 `Mutex` 快，应以实际负载测量。

**典型使用场景：**

- **运行时可更新的配置或缓存：** 大量线程读取当前快照，少数操作偶尔更新内容。
- **读操作相对耗时：** 同时允许多个读者可减少相互等待，但写者仍须等所有读者退出。
- **需要把读取和写入权限清楚地区分：** 读取者取得读守卫，修改者取得写守卫，类型接口直接表达访问方式。

如果读操作很短、写入并不少，锁管理开销可能抵消并行读取的收益，此时先使用 `Mutex<T>` 并测量。完全不变的数据只需要 `Arc<T>`，不需要读写锁。

**用法：** `read()` 取得只读守卫，`write()` 取得可变守卫；常与 `Arc` 组合。

```rust
use std::sync::{Arc, RwLock};
use std::thread;

fn main() {
    let settings = Arc::new(RwLock::new(String::from("v1")));
    let reader_settings = Arc::clone(&settings);
    let reader = thread::spawn(move || {
        let value = reader_settings.read().unwrap();
        assert_eq!(value.as_str(), "v1");
    });
    reader.join().unwrap();
    *settings.write().unwrap() = String::from("v2");
    assert_eq!(settings.read().unwrap().as_str(), "v2");
}
```

**常见操作：**

| 调用 | 作用与用法 |
| --- | --- |
| `RwLock::new(value)` | 创建读写锁。 |
| `lock.read()` | 阻塞直到取得读守卫；可有多个读者同时持有。 |
| `lock.write()` | 阻塞直到取得写守卫；写者独占访问。 |
| `lock.try_read()` / `lock.try_write()` | 非阻塞尝试获取相应守卫；失败时处理占用或中毒错误。 |
| `lock.get_mut()` | 已拥有 `&mut RwLock<T>` 时直接取得 `LockResult<&mut T>`，不必获取锁。 |
| `lock.into_inner()` | 消耗锁并取出内部值；若已中毒，返回错误以便处理。 |

**注意事项：** 持有读锁时再请求写锁可能死锁，应先释放读守卫。`read()` / `write()` 可能阻塞，也有 `try_read()` / `try_write()`。写者持锁 panic 通常会使锁中毒；读者持锁 panic 通常不会。公平性和等待顺序依赖平台，不能假设写者一定优先。

## 007 · `Weak<T>`：不拥有值的弱引用

**用途：** 表示“可以找到对象，但不负责延长其生命周期”的关系。树中常让父节点强拥有子节点，子节点弱引用父节点，以免强引用环。`Rc` 对应 `std::rc::Weak`，`Arc` 对应 `std::sync::Weak`。

**典型使用场景：**

- **父子节点回指：** 父节点强拥有子节点，子节点保存指向父节点的 `Weak`，便于向上查找而不形成强引用环。
- **观察者或缓存索引：** 索引希望在对象仍存活时找到它，但不应仅因为索引存在就阻止对象释放；访问时尝试 `upgrade()`。
- **后台任务回看所属对象：** 任务可以在对象仍存在时继续工作，对象已销毁时则通过 `None` 结束相关操作。

当关系本身必须保证对象存活时应保留强 `Rc` / `Arc`；只有“允许对象先销毁”的关系才适合 `Weak`。

**用法：** 通过 `Rc::downgrade(&strong)` 或 `Arc::downgrade(&strong)` 创建；访问前调用 `upgrade()`，其结果为 `Option<Rc<T>>` 或 `Option<Arc<T>>`。

```rust
use std::cell::RefCell;
use std::rc::{Rc, Weak};

struct Node {
    name: String,
    parent: RefCell<Weak<Node>>,
}

fn main() {
    let parent = Rc::new(Node {
        name: String::from("parent"),
        parent: RefCell::new(Weak::new()),
    });
    let child = Rc::new(Node {
        name: String::from("child"),
        parent: RefCell::new(Rc::downgrade(&parent)),
    });
    assert_eq!(child.parent.borrow().upgrade().unwrap().name, "parent");
    assert_eq!(child.name, "child");
    drop(parent);
    assert!(child.parent.borrow().upgrade().is_none());
}
```

**常见操作：** 以下以 `std::rc::Weak<T>` 为例；`std::sync::Weak<T>` 有对应的创建、升级和计数方法。

| 调用 | 作用与用法 |
| --- | --- |
| `Rc::downgrade(&strong)` | 从强 `Rc` 创建弱引用；`Arc` 对应 `Arc::downgrade(&strong)`。 |
| `Weak::new()` | 创建不指向任何分配的空弱引用，适合作为尚无父节点等状态的初始值。 |
| `weak.upgrade()` | 尝试取得强 `Rc<T>`，返回 `Option<Rc<T>>`；若内部值已销毁则为 `None`。 |
| `weak.strong_count()` | 查看当前仍有多少强引用；只能作观察，不能代替 `upgrade()` 判断可否访问。 |
| `weak.ptr_eq(&other)` | 判断两个弱引用是否指向同一份分配；两个空的 `Weak::new()` 也被视为相等。 |

**注意事项：** 弱引用不增加强引用计数，不能保证内部值仍存在，每次 `upgrade()` 都应处理 `None`。内部值销毁后，只要还有 `Weak`，保存引用计数的控制块分配仍可能保留。设计图结构时，应明确哪些边拥有节点，哪些边只负责回指。

## 008 · `Cell<T>`：复制或替换内部值

**用途：** 在单线程中通过共享引用修改内部值，适合计数器、标志位等小型 `Copy` 值。它不像 `RefCell` 那样提供对内部值的普通借用。

**典型使用场景：**

- **带有 `&self` 方法的轻量状态：** 例如访问次数、是否已初始化、最近一次选中的编号；调用者不必持有整个对象的可变引用。
- **在回调或界面对象中切换状态：** 读取旧值、计算新值并整体替换，且整个过程在单线程内进行。
- **非 `Copy` 值的整体交换：** 不需要借用内部字段，只需要用 `replace()` 取出旧值并放入新值时也可使用。

需要长期借用内部值或修改其某个字段时，改用 `RefCell<T>`；跨线程的简单标志位或计数器可考虑对应的原子类型。

**用法：** `get()` 在 `T: Copy` 时复制出值；`set()` 写入新值；`replace()` 替换并取出旧值。非 `Copy` 值也能使用 `replace()`、`take()`（要求 `Default`）或消耗自身的 `into_inner()`。

```rust
use std::cell::Cell;

struct Stats {
    visits: Cell<u32>,
}

impl Stats {
    fn visit(&self) {
        self.visits.set(self.visits.get() + 1);
    }
}

fn main() {
    let stats = Stats { visits: Cell::new(0) };
    stats.visit();
    stats.visit();
    assert_eq!(stats.visits.get(), 2);
}
```

**常见操作：**

| 调用 | 作用与用法 |
| --- | --- |
| `Cell::new(value)` | 创建内部值可被整体替换的容器。 |
| `cell.get()` | 复制并返回当前值，要求 `T: Copy`。 |
| `cell.set(new_value)` | 通过 `&Cell<T>` 写入新值，不要求 `T: Copy`。 |
| `cell.replace(new_value)` | 放入新值并返回旧值，适合非 `Copy` 类型的整体交换。 |
| `cell.take()` | 取出旧值，并留下 `T::default()`；要求 `T: Default`。 |
| `cell.into_inner()` | 消耗 `Cell<T>` 并取出内部值。 |

**注意事项：** `Cell<T>` 没有 `borrow_mut()`，不能直接取得内部值的 `&T` 或 `&mut T`；需要借用复杂结构并原地修改时使用 `RefCell<T>`。`get()` 要求 `T: Copy`，`set()` 不要求。`Cell<T>` 不能用于跨线程共享；简单的跨线程计数可考虑原子类型。

## 与智能指针有关的常用 Trait

Trait 描述类型能做什么。智能指针最常见的是 `Deref`、`DerefMut` 和 `Drop`；共享所有权和跨线程使用还涉及 `Clone`、`Send`、`Sync`。要注意**包装类型**与**访问时返回的守卫类型**不是同一个类型。

| 类型 | 与本文用法最相关的 Trait 或守卫 |
| --- | --- |
| `Box<T>` | 实现 `Deref<Target = T>`、`DerefMut`；若 `T: Clone`，克隆 `Box` 会克隆内部值 |
| `Rc<T>` / `Arc<T>` | 实现 `Deref<Target = T>` 和 `Clone`；克隆会增加强引用计数，不克隆内部值 |
| `Weak<T>` | 实现 `Clone`，但不实现指向 `T` 的 `Deref`；必须先 `upgrade()` |
| `RefCell<T>` | 容器本身不直接解引用到 `T`；`borrow()` 返回的 `Ref<T>` 实现 `Deref`，`borrow_mut()` 返回的 `RefMut<T>` 还实现 `DerefMut` |
| `Mutex<T>` | 容器本身不直接解引用到 `T`；`lock()` 返回的 `MutexGuard<T>` 实现 `Deref` 和 `DerefMut` |
| `RwLock<T>` | `read()` 返回的 `RwLockReadGuard<T>` 实现 `Deref`；`write()` 返回的 `RwLockWriteGuard<T>` 还实现 `DerefMut` |
| `Cell<T>` | 不通过 `Deref` 暴露内部引用；使用 `get()`、`set()`、`replace()` 等方法操作值 |

### `Deref`：像引用一样读取

`Deref` 的核心方法是 `deref(&self) -> &Target`。因此可以用 `*pointer` 读取内部值，也可以在需要 `&Target` 的地方利用**解引用强制转换**。`Box<T>`、`Rc<T>` 和 `Arc<T>` 都有此能力；`Weak<T>` 没有，因为目标值可能已经销毁。

```rust
fn print_length(text: &str) -> usize {
    text.len()
}

fn main() {
    let boxed = Box::new(String::from("Rust"));
    assert_eq!(print_length(&boxed), 4); // &Box<String> → &String → &str
    assert_eq!(boxed.len(), 4);          // 自动解引用后调用 String 的方法
}
```

这里的自动转换只发生在引用等合适的位置；它不会把一个 `Box<String>` 的所有权自动转换成 `String`。编写自己的智能指针时，只有希望它自然表现得像目标类型、且解引用不会意外失败时，才实现 `Deref`。

### `DerefMut`：通过可变守卫修改

`DerefMut` 在 `Deref` 基础上提供 `deref_mut(&mut self) -> &mut Target`。`Box<T>` 可直接使用；`RefCell<T>`、`Mutex<T>` 和 `RwLock<T>` 则需要先取得相应守卫，访问规则由容器负责检查。

```rust
use std::cell::RefCell;
use std::sync::Mutex;

fn main() {
    let mut boxed = Box::new(1);
    *boxed += 1;

    let cell = RefCell::new(2);
    *cell.borrow_mut() += 1; // RefMut<i32> 的 DerefMut

    let mutex = Mutex::new(3);
    *mutex.lock().unwrap() += 1; // MutexGuard<i32> 的 DerefMut

    assert_eq!((*boxed, *cell.borrow(), *mutex.lock().unwrap()), (2, 3, 4));
}
```

`Rc<T>` 和 `Arc<T>` 不提供通常意义上的 `DerefMut`：共享所有权时不能无条件交出唯一的 `&mut T`。需要修改时使用前文的内部可变性或同步容器；如果确知没有其他强所有者，也可了解 `Rc::get_mut` / `Arc::get_mut`。

### `Drop`：离开作用域时清理

`Drop` 定义值销毁时要执行的清理逻辑。`Box` 释放其拥有的堆分配；`Rc`、`Arc` 销毁一个句柄时减少强引用计数，最后一个强引用消失才销毁内部值；借用和锁的守卫离开作用域时解除借用或释放锁。通常无需手动为这些标准库类型实现 `Drop`。

```rust
use std::sync::Mutex;

fn main() {
    let data = Mutex::new(vec![1]);
    {
        let mut guard = data.lock().unwrap();
        guard.push(2);
    } // guard 在此处被销毁，锁随即释放

    let guard = data.lock().unwrap();
    assert_eq!(*guard, vec![1, 2]);
    drop(guard); // 需要提前释放时，调用 std::mem::drop
}
```

不能直接调用 `Drop::drop(&mut value)` 提前销毁某个值，应调用 `drop(value)` 转移其所有权。守卫的释放时机由其实际作用域决定；需要及时解锁时，可使用小作用域或显式 `drop(guard)`。

### `Clone`：复制值还是增加所有者

`Clone` 的具体行为取决于类型，不能把所有 `.clone()` 都理解为深拷贝。`Box<T>::clone()` 在 `T: Clone` 时创建独立的内部值；`Rc<T>` 和 `Arc<T>` 的 `clone()` 增加强引用计数；`Weak<T>::clone()` 增加弱引用计数。共享指针建议写成 `Rc::clone(&value)` 或 `Arc::clone(&value)`，使“增加所有者”的意图更明显。

```rust
use std::rc::Rc;

fn main() {
    let boxed = Box::new(String::from("A"));
    let boxed_copy = boxed.clone();
    assert_ne!(boxed.as_ptr(), boxed_copy.as_ptr()); // 两份独立的字符串数据

    let shared = Rc::new(String::from("B"));
    let other_owner = Rc::clone(&shared);
    assert!(Rc::ptr_eq(&shared, &other_owner)); // 指向同一份分配
    assert_eq!(Rc::strong_count(&shared), 2);
}
```

### `Send` 与 `Sync`：决定能否跨线程

这两个是标记 Trait：`Send` 表示值可以安全地转移到另一个线程；`Sync` 表示其共享引用 `&T` 可以安全地在线程间传递。它们没有要主动调用的方法，编译器会在创建线程等位置检查约束。

- `Rc<T>` 既不是 `Send` 也不是 `Sync`；`Arc<T>` 在 `T: Send + Sync` 时才能跨线程共享。
- `Cell<T>`、`RefCell<T>` 不是 `Sync`，因此 `Arc<RefCell<T>>` 不能变成可跨线程共享的可变容器。
- `Mutex<T>` 在 `T: Send` 时可用于跨线程同步访问；`RwLock<T>` 通常要求 `T: Send + Sync`。这也是 `Arc<Mutex<T>>` 和 `Arc<RwLock<T>>` 常见的原因。
- 不要为了绕过编译错误而随意手写 `unsafe impl Send` 或 `unsafe impl Sync`；那会把线程安全保证交给实现者自己。

## 选型顺序

1. 只有一个所有者时，优先使用普通值或引用；需要堆上间接层时选 `Box<T>`。
2. 需要共享所有权时，单线程选 `Rc<T>`；跨线程且 `T` 满足线程安全要求时选 `Arc<T>`。
3. 需要从共享引用修改数据时，单线程的小型值选 `Cell<T>`，动态借用选 `RefCell<T>`；跨线程按访问模式选 `Mutex<T>`、`RwLock<T>` 或原子类型。
4. 需要回指但不延长生命周期时，选对应 `Rc` 或 `Arc` 的 `Weak<T>`，并检查 `upgrade()` 的结果。

## 参考资料

- [《The Rust Programming Language》：Smart Pointers](https://doc.rust-lang.org/book/ch15-00-smart-pointers.html)
- [`Box<T>`](https://doc.rust-lang.org/std/boxed/struct.Box.html)、[`Rc<T>`](https://doc.rust-lang.org/std/rc/struct.Rc.html)、[`Arc<T>`](https://doc.rust-lang.org/std/sync/struct.Arc.html)
- [`RefCell<T>`](https://doc.rust-lang.org/std/cell/struct.RefCell.html)、[`Cell<T>`](https://doc.rust-lang.org/std/cell/struct.Cell.html)
- [`Mutex<T>`](https://doc.rust-lang.org/std/sync/struct.Mutex.html)、[`RwLock<T>`](https://doc.rust-lang.org/std/sync/struct.RwLock.html)
- [`std::rc::Weak<T>`](https://doc.rust-lang.org/std/rc/struct.Weak.html)、[`std::sync::Weak<T>`](https://doc.rust-lang.org/std/sync/struct.Weak.html)
- [`Deref`](https://doc.rust-lang.org/std/ops/trait.Deref.html)、[`DerefMut`](https://doc.rust-lang.org/std/ops/trait.DerefMut.html)、[`Drop`](https://doc.rust-lang.org/std/ops/trait.Drop.html)、[`Clone`](https://doc.rust-lang.org/std/clone/trait.Clone.html)
- [`Send`](https://doc.rust-lang.org/std/marker/trait.Send.html)、[`Sync`](https://doc.rust-lang.org/std/marker/trait.Sync.html)
