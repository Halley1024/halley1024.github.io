---
layout: post
title: A Detailed Explanation of Rust Lifetimes
date: 2026-09-26 20:49 +0800
categories: [Rust,Homepage]
tags: [Rust]     # TAG names should always be lowercase
math: true
mermaid: true
---

# Rust生命周期详解

> [!CAUTION]
>
> 本篇文档仅以自己的角度去理解Rust生命周期，文档可能存在一些用词不严谨和理解偏差的问题，仅供参考，欢迎指正。

------

rust 生命周期机制是与所有权机制同等重要的资源管理机制。要想理解rust语言的生命周期，最关键就是理解下面这几句话：

> [!IMPORTANT]
>
> 1. **生命周期机制是让码农按照规范来写代码，而非是一套rust语言的自我纠错机制。因此不要试图「通过加标注把代码修好」。** 
>
>    生命周期标注是描述性语言，不是控制语句。当编译器拒绝你的代码时，它在告诉你一句很有价值的话——「**你设想的这段数据，在这里可能已经不存在了**」。听懂这句话，比背下所有规则都重要。
>
> 2. **资源所有权寿命 $\ge$ 引用生命周期。**
>
>    你所写代码必须符合的要这个要求才能保证不会出现引用悬垂问题。
>
> 3. **Rust尽可能通过所有权机制、借用检查、生命周期来保证内存安全，你写不出悬垂、越界、数据竞争的代码。除非你显性使用`unsafe`代码**

## 001 · 悬垂引用

什么叫做悬垂引用？我们知道对于引用类型数据而言，其变量本生不存放数据，它只保存指向堆里的内存空间。因此一旦原本它所指向的内存空间中的数据失效了，那么其本保存的指向信息就会指到其他内存空间中去，因此就导致了悬垂引用的问题。举个例子：

```rust
fn main() {
    let r;                // 先声明，稍后赋值
    {
        let x = 5;
        r = &x;           // x 绑定在内存里
    }                     // ← x 在这里被 drop
    println!("{}", r);    // ← r 还活着，但指向的东西没了
}
```

编译器返回的结果：

```bash
error[E0597]: `x` does not live long enough
 --> src/main.rs:5:13
  |
4 |         let x = 5;
  |             - binding `x` declared here
5 |         r = &x;
  |             ^^ borrowed value does not live long enough
6 |     }
  |     - `x` dropped here while still borrowed
7 |     println!("{}", r);
  |                    - borrow later used here
```

注意最后三行：`dropped here`、`borrow later used here`。编译器实际上把 `x` 的存活区间和 `r` 的使用点放在同一条时间轴上做了比较。这就是「生命周期」这个词的字面含义。

编译器报错原因是： `ra`重新指向的内容空间里由指向`a = 100`变为指向`a = 20`的内容空间，但是`a = 20`的内容空间失效了，理应`ra`也该消失，但是`ra`没有消失，所以导致`ra`指向的为空，出现**悬垂引用**。

> [!IMPORTANT]
>
> **生命周期是「区间」，不是「时长」。**编译器关心的是「引用被使用的所有时刻」是否全部落在「被引用值存活的区间」之内，而不是某个具体的、需要你去计算的毫秒数。`'a` 永远代表一个*区间*（可能跨越多个作用域），而不是一个点。

------

## 002 ·生命周期注释

生命周期是「描述」而非「指令」。因为生命周期描述`'a`值告诉我编译器变量与引用的生命周期关系，编译完成后，所有生命周期标注被完全擦除。为什么需要加入这些描述呢？举个简单的例子：

```rust
// 假如不使用生命周期注释 'a		// 程序不给过
fn longest(x: &str, y: &a str) -> &str {
    if x.len() > y.len() { x } else { y }
}
// 加入生命周期注释 'a
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() { x } else { y }
}
fn main(){
    let x:String = String::from("Hello");
    let y:String = String::from("Hi");
    let res = longest(&x, &y);
    assert_eq!(res, &x);
}
```

如果不使用生命周期，那么`longest`返回的是一个引用类型，那么返回值会有悬垂引用风险。并且由于返回值是多值条件返回，编译器在编译前无法明确知道怎么返回哪一个值，所以借用检查器无法检测出引用的生命周期，就无法保证程序不出现悬垂引用。因此必须借助生命周期注释`'a`。

既然如此，我们该怎么解读这个生命注释呢？拿加入生命周期注释的函数来说

```rust
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() { x } else { y }
}
```

使用`'a`实际就是告诉编译器：**我给了一个生命周期区间，这个生命周期区间长度取决于`x`和`y`引用的生命周期最短的一个**。为什么这样说就能保证编译器能够保证返回值的引用不会悬垂呢？其原因如下：

1. rust中不允许返回函数内创建的变量的引用。这样必出现悬垂引用。
2. 既然返回的不是函数内变量的引用，那么返回的引用类型必然是传入的参数`x`或者`y`。但是编译器在运行前都不知道返回的具体是哪一个，因此就必须保证返回的引用存在的生命周期要同时小于`x`和`y`的生命周期。**简而言之，就是不准`x`和`y`在返回的引用调用前失效。**

```rust
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() { x } else { y }
}
fn main() {
    let res;
    let x: String = String::from("Hello");
    {
        let y: String = String::from("Hi");
        res = longest(&x, &y);	
        // assert_eq!(res, &x);	<- 调用res需要与y变量保持一样的生命周期
    }
    assert_eq!(res, &x);	// 这个编译器认为返回值可能出现返回y的引用，y的生命周期小于res，会出现悬垂引用
}
```

**简而言之，加入`'a`是让编译器能看明白函数签名，确保返回的引用能和生命周期最短的引用一起消失**。这样编译器就认为不会有悬垂引用导致的内存泄漏问题。

还有一些辅助语法：

| 语法      | 位置                    | 含义                                           |
| :-------- | :---------------------- | :--------------------------------------------- |
| `'a`      | 泛型参数列表 / 类型位置 | 具名生命周期参数                               |
| `'_`      | 类型位置                | 占位符，让编译器自行推断（`fn f(x: &'_ str)`） |
| `'static` | 类型位置 / 约束         | 贯穿整个程序的生命周期，见第 07 节             |
| `'a: 'b`  | where 子句              | `'a` 至少和 `'b` 一样长                        |
| `T: 'a`   | where 子句              | 类型 `T` 中不含有比 `'a` 更短的引用            |
| `for<'a>` | trait 约束              | 高阶生命周期，见第 10 节                       |

------

## 003 ·生命周期注释省略规则

早期 Rust 要求给每个引用都写标注，噪音极大。后来引入了三条省略规则，编译器按顺序应用，能推出来就推。这三条规则值得背下来——它们是理解「为什么这个签名能编译、那个不能」的钥匙。省略规则如下：

1. 如果输入的多个引用参数，那么输入的每个引用参数均有一个独立的生命周期。

   ```rust
   fn f(x: &i32, y: &i32) -> ...	
   // 等价于
   fn f<'a, 'b>(x: &'a str, y: &'b str) -> ...
   ```

   **注意：多个输入引用参数输入，要返回引用时，至少要注明一个；如果返回其引用的值，那么就不需要生命周期注释**

   ```rust
   fn f(x: &str, y: &str) -> &str { ... }   // ❌ 必须标注
   fn f(x: &str, y: &str) -> String { ... }   // ✅ 必须标注
   ```

2. 如果输入的只有一个引用参数，它被赋给所有输出生命周期

   ```rust
   fn f(x: &str) -> &str
   // 等价于
   fn f<'a>(x: &'a str) -> &'a str
   
   fn f(x: &str, n: i32) -> &str { ... }    // ✅ 只有 x 是引用
   ```

3. 如果输入参数存在`&self`或 `&mut self`，`self` 的生命周期赋给所有输出

   ```rust
   struct Foo {
       x: String
   }
   
   impl Foo {
       fn get(&self, x: &str) -> &str
       // 等价于
       fn get<'a, 'b>(&'a self, x: &'b str) -> &'a str
   }
   ```

   `&self`引用：在结构体实现结构体方法中，是指借用结构体本身；在trait实现类中，是指实现`trait`的类型本身（实现主体不一定是结构体、枚举、元组、原型都可以实现`trait`）。

   这里为什么让所有输出的生命周期与`&self`相同就能保证不出现引用悬垂问题呢？因为返回的引用`&self.x`的生命周期要不短于`&self`所以无担心。（**结构体一般保存的是值类型，一旦含有引用，整个结构体就必须带生命周期参数**）

> [!NOTE]
>
> **当你在纠结要不要加生命周期标注时，通常说明你的函数设计里混进了「借用」和「所有权」的模糊地带。**很多情况下，直接把参数改成 `&str → String`（或返回 `String` 而不是 `&str`）能让一大片标注和约束凭空消失。先考虑改设计，再考虑加标注。

------

## 004 · 结构体持有引用

一般来说，结构体里存放的都是一些值类型数据，不需要对结构体进行标注。但是当结构体里出现引用时就需要对结构体进行标注。

```rust
struct Excerpt<'a> {	// 表示part借用来的变量的生命周期比须必Excerpt实例长
    part: &'a str,
}

struct Pair<'a, 'b> {	// 结构体不同的引用也允许有独立的生命周期，基本逻辑也是一样
    a: &'a str,
    b: &'b str,     // a、b 生命周期独立
}

impl<'a> Excerpt<'a> {
    // 省略规则 3：输出绑定到 &self
    fn announce(&self, announcement: &str) -> &str {
        println!("Attention please: {}", announcement);
        self.part
    }
}
```

语义上，`struct Excerpt<'a>` 表达的是：一个 `Excerpt<'a>` 实例，**不可能比 `part` 所指向的那段数据活得更久**。编译器会在每一个构造和使用它的地方验证这个不变量。结构体一旦持有引用，它的类型签名就会渗透到你 API 的每一个角落，因此结构体使用引用的代价很高，**所以对于实体类，建议不使用引用类型的成员变量**。

------

## 005 · `'static`生命周期注释

`'static`两种写法拥有两种不一样的解释，这也是最容易混淆的概念：

- `&'static T`：引用指向的数据**存活于整个程序期间（进程）**。

  - 典型的例子有：字面量、`Box::leak()`、`static x: i32 = 1`常量

- `T: 'static`：类型 `T` 中**不含有任何非 'static 的借用**（即不借用短命数据）

  - 典型的有：`String`、`Vec<i32>`、`i32`、`Box<T>` 
  - 常见的启动线程函数`thread::spawn`，里面接受一个闭包参数，闭包参数里面不允许使用借用不是`'static'`的变量。除非闭包拥有内容，如使用`move`关键字。

  ```rust
  use std::sync::{Arc, Mutex};
  use std::thread;
  
  fn main() {
      let counter = Arc::new(Mutex::new(0));
      let mut workers = Vec::new();
      for _ in 0..4 {
          let counter = Arc::clone(&counter);		// 可以不需要clone，但是第一次循环之后，后续线程拿不到counter就会报错
          workers.push(thread::spawn(move || {	// move指令是必须的，因为counter随时会关闭，无法使用借用，只能拥有。
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

  

一个拥有自己数据的类型，可以活到程序结束，也可以随时销毁——它不受任何外部区间的约束。例如：`String`，`Vec`，`Box`等

> [!CAUTION]
>
> 看到借用检查器报 `borrowed value does not live long enough` 时，**不要第一反应就去加 `'static` 或 `Box::leak`**。绝大多数情况下真正的问题是所有权设计错了：该 move 的地方传了引用，该返回拥有所有权的值却返回了借用。把 `'static` 当成万能解药会迅速演变成内存泄漏。



------

## 006 · 生命周期约束与子类型

虽然 Rust 没有传统意义上的类继承子类型，但**生命周期之间存在子类型关系**：如果 `'long` 是比 `'short` 更长的区间，那么 `&'long T` 可以被当作 `&'short T` 使用。区间更长，能用的地方自然更多。

生命周期约束可以归为rust约束语法一类。约束语法主要有三种：`trait`约束（`T:trait`）、生命周期约束（`'a: 'b`）、关联类型约束`I::Item = X`。这里我只总结生命周期约束语法。

首先，我们需要了解生命周期语法（`'a: 'b`）的语义，读作：`'a`引用至少活的比`'b`要久。

```rust
fn f<'a: 'b, 'b>(x: &'a str, y: &'b str) -> &'b str {
    y
}
// 等价于
fn f<'a, 'b>(x: &'a str, y: &'b str) -> &'b str
where
    'a: 'b,
{
    y
}
// 这里其实约束其实可以省略，因为返回的'b只与y有关，rust允许不同的引用参数拥有独立的生命周期
```

不仅两个生命周期标注的引用可以建立约束，泛型和其他生命周期引用也能建立约束`T:'a`，读作：`T`中所有的引用都至少要比`'a`要长。

```rust
fn f<'a, 'b, T: 'b>(x: &'a T, y: &'b T) -> &'a T{
    x
}
// 等价于
fn f<'a, 'b, T>(x: &'a T, y: &'b T) ->  &'a T
where 
	T:'b
{
    x
}
```

**这里需要注意的是，生命周期标注和泛型标注属于平级，使用同一个<>包裹。**

------

## 007 · 型变

**型变（Variance）** 是 Rust 里一个偏底层的概念。当类型构造器「套」在生命周期外面时，里面的生命周期能不能被替换成长度不同的另一个？这个规则就是**型变**。换而言之，型变就是引用的生命周期转换。因为Rust严格的编译检测机制，传入的引用类型的生命周期要与内部接受的引用类型一致，但是这样话增加编程的难度，因此rust运行对一些能够确保不出现内存泄露的生命周期转化操作进行自动化处理。型变分为了三类：

1. 协变：通常是指长生命周期的**只读引用**转为了短生命周期的**只读引用**。这个在Rust语言中是**天然允许**的。因为长生命周期的不可变引用返回还是原来的引用，生命周期没有改变。

   ```rust
   // &str
   fn print_str(s: &str) {
       println!("{}", s);
   }
   fn main() {
       // 'static 的长引用
       let long: &'static str = "hello";	// 字面量的生命周期是'static
       
       // 函数要求 &'short str，传入长引用 ✅
       print_str(long);
   
       // 更明显：把 &'static str 存进一个要求短生命周期的变量
       let short: &str = long;   // 实际是 &'static str 被当成 &'short str
       println!("{}", short);
   }
   // Box<T>，Vec<T>
   fn take_strs(v: Vec<&str>) {		// 按值传递，所属权发生变化
       for s in v {
           println!("{}", s);
       }
   }
   
   fn main() {
       let a: &'static str = "static";
       let b: &'static str = "also static";
   
       let v: Vec<&'static str> = vec![a, b];		// 
   
       // Vec<&'static str> 当作 Vec<&'short str> 用 ✅
       take_strs(v);
   }
   
   // 同级协变
   fn wants_short<'a>(x: &mut &'a str, y: &'a str) {
       *x = y;
   }
   
   fn main() {
       let s: String = String::from("original");
       let mut s_ref: &str = &s;
       let temp: &'static str = "temp";
       wants_short(&mut s_ref, &temp);		// 这里temp的'static生命周期写变成'a
       println!("{}", s_ref);
   }
   ```

2. 不变：一般来说函数传入的是`&mut`可变引用的话，那么编译器是不允许将`'static->'a`操作的，因为返回的类型可能是短命引用。

   ```rust
   fn wants_short<'a>(x: &mut &'a str, y: &'a str) {
       *x = y;		// 这条错误
   }
   
   fn main() {
       let mut s: &'static str = "original";		// 这里生命了一个全局有效的字面量
       let temp = String::from("temp");
       wants_short(&mut s, &temp);   // 借用检查器会优先判断 s 的生命周期是'static，所以传入'a默认为'static，那么就不允许传入短命引用，
   
       println!("{}", s);
   }
   ```

   上面的代码是编译不通过的，**因为`x`与`y`绑定相同的生命周期**，然而`wants_short(&mut s, &temp);`传入的参数一个是`'static`、一个是`'a`，编译器为了避免`'static`的被修改成`'a`而导致原本`'a`引用的资源释放了，但是`'static`引用还是存续导致悬垂引用。

   上面有三种修改方法，

   1. 将传入引用的生命周期对齐，也就是`let temp = String::from("temp"); -> let temp = "temp";`；或`let mut s: &'static str = "original"; -> let mut s: &str = "original";`
   2. 还有一种就是函数内部生命`x`和`y`两个引用的生命周期不一样，也就是`fn wants_short<'a>(x: &mut &'a str, y: &'a str) -> fn wants_short<'a, 'b>(x: &mut &'a str, y: &'b str)`。

   > [!NOTE]
   >
   > **一言以概之，传入有`&mut`可变引用，那么就必须保证函数内部引用操作时，生命周期严格对齐，不允许将`'long -> 'short`**。
   
3. 逆变：是型变的一种，描述：当子类型关系传到外层时，**方向反转**。典型的就是当函数作为引用参数传递时，如下：

   ```rust
   fn call_with_static(f: fn(&'static str)) {	// 接受一个全局有效的字面量引用
       f("hello");
   }
   
   fn takes_any<'a>(s: &'a str) {	// 
       println!("{}", s);
   }
   
   fn main() {
       call_with_static(takes_any);   // ✅ 通过
   }
   ```

   > [!NOTE]
   >
   > **其实只需要记住，函数从外往里面拨开，引用变量的生命周期是逐层缩短的，符合协变定义。**

常见引用类型允许型变的范围如下：


| 类型                     | 对生命周期 `'a` | 对内部类型 `T` | 原因                       |
| :----------------------- | :-------------- | :------------- | :------------------------- |
| `&'a T`                  | 协变            | 协变           | 只读，缩短区间永远安全     |
| `&'a mut T`              | 协变            | 不变           | 可写！见下方反例           |
| `Box<T>` / `Vec<T>`      | —               | 协变           | 拥有所有权，只读语义       |
| `Cell<T>` / `RefCell<T>` | —               | 不变           | 内部可变性，可在别处改写   |
| `*const T`               | —               | 协变           | 裸指针只读视角             |
| `*mut T`                 | —               | 不变           | 可写                       |
| `fn(T) -> U`             | —               | 逆变 / 协变    | 参数位置逆变，返回位置协变 |

------

## 008 · 高阶生命周期

有些约束需要表达「对**任意**生命周期都成立」，而不仅仅是对某个具体的 `'a`。这就是 `for<'a>` 语法。

```rust
// 含义：接受一个「对任意输入生命周期都能工作」的函数
// 注意输出与输入共享同一个 'a，而不是各自独立
fn apply<F>(s: &str, f: F) -> &str
where
    F: for<'a> Fn(&'a str) -> &'a str,
{
    f(s)
}

fn main() {
    let text = String::from("hello world");
    // 这个闭包满足 for<'a>：输入输出都绑定到同一个 'a
    let result = apply(&text, |s| &s[..5]);
    println!("{}", result);
}
```

好消息是：**你几乎不需要手写 `for<'a>`**。当你写 `Fn(&str) -> &str` 时，生命周期省略规则会自动把它展开成 `for<'a> Fn(&'a str) -> &'a str`。只有当省略规则推出你不想要的结果时，才需要显式写。

> [!NOTE]
>
> HRTB 与闭包的交互有个已知难点：如果一个闭包的返回引用**来自它自己的捕获环境**而不是来自参数，它就无法满足 `for<'a> Fn(&'a str) -> &'a str`。这时通常会撞上 `expected a closure that implements the Fn trait, but this closure only implements FnOnce` 之类的报错。解决方向是把捕获的值改成 move 进去、或者干脆用具名函数替代闭包。

------

## 009 ·生命周期实战报错

生命周期错误看起来吓人，但 `rustc` 的错误信息其实是同类工具里最好的。学会按类型分类，排查就会快很多。

| 错误码  | 典型信息                                                     | 根因与方向                                                   |
| :------ | :----------------------------------------------------------- | :----------------------------------------------------------- |
| `E0106` | missing lifetime specifier                                   | 多输入引用 + 无 `&self`。显式标注，或让返回值拥有所有权。    |
| `E0597` | borrowed value does not live long enough                     | 被借用的值太早 drop。把数据的声明上提到更外层作用域。        |
| `E0499` | cannot borrow as mutable more than once                      | 同一时间存在两个可变借用。拆成两段、用 `split_at_mut`、或引入内部作用域。 |
| `E0502` | cannot borrow as immutable because it is also borrowed as mutable | 读写借用重叠。缩短读借用的存活区间（通常提前取好值）。       |
| `E0515` | cannot return reference to local variable                    | 返回了局部变量的引用。**改设计**——返回拥有所有权的类型。     |
| `E0621` | explicit lifetime required                                   | 缺少 `'a: 'b` 或 `T: 'a` 约束，补上 where 子句。             |
| `E0521` | borrowed data escapes outside of function/closure            | 把短命引用塞进了长命容器。检查所有权归属。                   |

