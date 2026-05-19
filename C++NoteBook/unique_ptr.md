# `std::unique_ptr` 学习笔记

## 1. 它解决什么问题

裸指针 + 手动 `delete` 的典型问题：

```cpp
Foo* p = new Foo();
// ... 中间 return、抛异常、早退 ...
delete p;  // 可能执行不到 → 泄漏；或执行两次 → 未定义行为
```

当对象**只有一个明确所有者**时，用 `std::unique_ptr` 表达「独占所有权」：

```text
unique_ptr 销毁或 reset → 对象自动释放
不能拷贝 → 不会出现两个 owner 各自 delete
可以移动 → 所有权可以安全转移给别的变量/函数/容器
```

一句话：**谁持有 `unique_ptr`，谁就负责释放；离开作用域自动释放。**

## 2. 头文件

```cpp
#include <memory>
```

`unique_ptr`、`make_unique` 与 `shared_ptr` 同在 `<memory>` 中。

## 3. 创建对象

推荐（C++14 起）：

```cpp
auto p = std::make_unique<Foo>();
```

带构造参数：

```cpp
auto p = std::make_unique<Foo>(port, addr);
```

也可以（不如 `make_unique` 简洁）：

```cpp
std::unique_ptr<Foo> p(new Foo());
```

`make_unique` 的好处：

- 写法短，类型只写一次。
- 异常安全更好（若构造参数求值抛异常，不会只 `new` 了一半却没人管）。

### 3.1 从裸指针构造（接管已有堆对象）

当对象**已经**由 `new` 创建，且你打算把「释放责任」交给 `unique_ptr` 时：

```cpp
Socket* raw = new Socket(fd, port, addr);

// 从这一刻起，由 p 负责 delete，不要再手动 delete raw
std::unique_ptr<Socket> p(raw);
```

也可以直接写进成员初始化列表或函数调用：

```cpp
class Session {
public:
    Session(Context* ctx, std::unique_ptr<Connection> conn)
        : ctx_(ctx), conn_(std::move(conn)) {}
};

// 调用方：裸指针转手，所有权在构造时交给 Session
Connection* raw = factory->accept();  // 假设返回 new 出来的指针
auto session = std::make_shared<Session>(
    ctx, std::unique_ptr<Connection>(raw));
// 此后不要 delete raw，也不要再使用 raw
```

**必须遵守的三条规则：**

```text
1. 一块堆内存只能被「一个」unique_ptr 接管一次
2. 接管后，原裸指针调用方不得再 delete，最好不再使用 raw
3. 不要对同一 raw 再写 unique_ptr<T>(raw)，否则 double free
```

**和 `make_unique` 的分工：**

```text
make_unique<T>()     → 我负责 new，也负责之后 delete
unique_ptr<T>(raw)   → 别人已经 new 了，我从现在开始负责 delete
```

**常见模式：上游仍传裸指针，下游构造时一次性接管**

适用：任务队列是 `std::function<void()>`、lambda 不便按值捕获 `unique_ptr`，但希望成员最终由 `unique_ptr` 管理。

```text
工厂 / accept：返回 Resource*（约定：只有一个 Owner 会接管）
异步任务里：std::unique_ptr<Resource>(raw) 交给 Owner 的构造函数
Owner 成员：std::unique_ptr<Resource>
```

```cpp
void Scheduler::attach(Resource* raw, ...) {
    loop_->post([this, raw, handler]() {
        auto owner = std::make_shared<Session>(
            ctx_, std::unique_ptr<Resource>(raw));
        // ...
    });
}
```

lambda 捕获裸指针往往是为了满足 `std::function` 的可拷贝要求；**真正接管应只发生一次**，在 Owner 构造里 `unique_ptr<T>(raw)`。

### 3.2 `make_unique` 别忘了括号

```cpp
// 对
vec.emplace_back(std::make_unique<Worker>());

// 错：少了 ()，编译器报一堆 unique_ptr 模板错误
vec.emplace_back(std::make_unique<Worker>);
```

传给按值接收 `unique_ptr` 的函数时，要用 `std::move`：

```cpp
// 对
createSession(ctx, std::move(conn));

// 错：试图拷贝 unique_ptr
createSession(ctx, conn);
```

## 4. 独占所有权（没有引用计数）

```cpp
auto p1 = std::make_unique<int>(42);
// auto p2 = p1;  // 编译错误：不能拷贝
auto p2 = std::move(p1);  // 可以：所有权从 p1 转移到 p2
```

转移后：

```text
p1 == nullptr（不拥有对象）
p2 拥有对象，负责释放
```

与 `shared_ptr` 对比：

```text
shared_ptr：拷贝 +1，最后一个离开才析构
unique_ptr：只能有一个 owner，移动即换主人
```

## 5. 访问对象

用法与裸指针、`shared_ptr` 类似：

```cpp
auto p = std::make_unique<Foo>();

p->run();
(*p).run();
```

判断是否为空：

```cpp
if (p) {
    p->run();
}
```

主动释放（提前析构对象，指针置空）：

```cpp
p.reset();           // 析构对象，p 变为 empty
p.reset(new Foo());  // 先释放旧对象，再接管新对象（少用，优先 make_unique + move）
```

## 6. 拷贝与移动

**禁止拷贝**（编译期报错）：

```cpp
std::unique_ptr<Foo> a = std::make_unique<Foo>();
std::unique_ptr<Foo> b = a;  // 错
```

**允许移动**：

```cpp
std::unique_ptr<Foo> b = std::move(a);
```

移动后不要再使用已空的 `a`（除非重新赋值）。

lambda 按值捕获 `unique_ptr` 会移动进去（C++14）：

```cpp
auto p = std::make_unique<Foo>();
postTask([p = std::move(p)]() {
    p->run();
});
```

## 7. 作为函数参数

### 7.1 只借用、不接管所有权

传引用或裸指针（调用方保证对象活着）：

```cpp
void use(Foo& foo);
void use(const Foo& foo);
void use(Foo* foo);  // 不 delete，不负责生命周期
```

### 7.2 接管所有权（推荐 `unique_ptr` 按值 + move）

```cpp
void takeOwnership(std::unique_ptr<Foo> p) {
    // 函数体结束时 p 析构，除非再 move 出去
}

auto obj = std::make_unique<Foo>();
takeOwnership(std::move(obj));  // 调用后 obj 为空
```

### 7.3 工厂函数返回 `unique_ptr`（常见模式）

```cpp
std::unique_ptr<Foo> createFoo() {
    return std::make_unique<Foo>();
}

auto p = createFoo();  // 移动语义，无额外拷贝开销
```

工厂直接返回 `unique_ptr` 的写法：

```cpp
std::unique_ptr<Connection> accept();
// ...
onNewConnection(std::move(client));
```

### 7.4 传 `const unique_ptr&`（较少用）

只观察、不拿走所有权时用；若函数要保存对象，应传值并 `move` 进成员。

简单经验：

```text
单 owner、要转移：按值传 unique_ptr + std::move
只临时用一下：传 Foo& / Foo*
工厂创建：返回 unique_ptr
```

## 8. 放进容器

`vector`、`map` 等可以持有 `unique_ptr`，元素本身不可拷贝，但可以移动：

```cpp
std::vector<std::unique_ptr<Worker>> workers;
workers.emplace_back(std::make_unique<Worker>());

std::unordered_map<int, std::unique_ptr<Listener>> listeners;
listeners[8080] = std::make_unique<Listener>(loop, 8080);
```

从容器移除元素 → 该元素的 `unique_ptr` 析构 → 对象释放：

```cpp
listeners.erase(8080);
workers.clear();
```

**不能**这样写（缺少移动）：

```cpp
std::vector<std::unique_ptr<Foo>> v;
Foo* raw = new Foo();
v.push_back(raw);  // 错
```

## 9. 不要手动 `delete`

错误：

```cpp
auto p = std::make_unique<Foo>();
delete p.get();  // 错：双重释放风险
```

错误：

```cpp
Foo* raw = new Foo();
std::unique_ptr<Foo> p1(raw);
std::unique_ptr<Foo> p2(raw);  // 错：两个 unique_ptr 管同一块内存
```

正确：一块堆内存只交给**一个** `unique_ptr`（或交给 `shared_ptr` 一条控制链）。

## 10. `get()`、`release()` 与 `reset()`

### `get()`

```cpp
Foo* raw = p.get();
```

- 不转移所有权，`p` 仍然拥有对象。
- 用于调用只接受裸指针的旧 API。
- **不要** `delete raw`；**不要**把 `raw` 长期存起来当 owner。

### `release()`

```cpp
Foo* raw = p.release();
```

- `p` 放弃所有权，变为 empty。
- **你**必须之后在合适的地方 `delete raw`（或再交给另一个智能指针）。
- 现代 C++ 中尽量少用；优先 `move` 给别的 `unique_ptr`。

### `reset()`

```cpp
p.reset();              // 释放当前对象
p = std::make_unique<Foo>();  // 更常见的替换写法
```

## 11. 与 `shared_ptr` / `weak_ptr` 的关系

`unique_ptr` **没有**配套的 `weak_ptr`。

| 需求 | 做法 |
|------|------|
| 单 owner | `unique_ptr` |
| 多 owner、异步延长生命 | `shared_ptr` |
| 观察但不拥有 | `weak_ptr`（观察的是 `shared_ptr` 管理的对象） |

可以从 `unique_ptr` **转移**出所有权，再构造 `shared_ptr`（对象只应被一条智能指针链管理）：

```cpp
std::unique_ptr<Foo> u = std::make_unique<Foo>();
std::shared_ptr<Foo> s = std::move(u);  // u 为空，s 接管
// 或：std::shared_ptr<Foo> s(std::move(u));
```

**不要**对同一裸指针同时存在活跃的 `unique_ptr` 和 `shared_ptr` 两套独立控制块。

## 12. 作为类成员（RAII）

典型：父对象拥有子对象，父析构时子自动释放。

```cpp
class Session {
    Loop* loop_;  // 非拥有：由外层保证活得比 Session 久
    std::unique_ptr<Connection> conn_;
    std::unique_ptr<IoChannel> channel_;
    std::unique_ptr<Buffer> read_buf_;
    std::unique_ptr<Buffer> write_buf_;
};
```

```cpp
class App {
    std::unique_ptr<ThreadPool> pool_;
    std::unique_ptr<MainLoop> main_loop_;
    std::vector<std::unique_ptr<Worker>> workers_;
};
```

好处：

- 类析构函数里通常**不必**再写 `delete channel_` 等。
- 成员析构顺序与声明顺序**相反**，设计时要注意依赖（见第 18 节）。

## 13. 自定义删除器

默认删除方式：`delete ptr`。

管理 `FILE*`、C API 句柄、或「必须在指定线程析构」时，可指定删除器：

```cpp
auto f = std::unique_ptr<FILE, decltype(&fclose)>(
    fopen("a.txt", "r"),
    &fclose
);
```

删除器类型是 `unique_ptr` 模板参数的一部分；`shared_ptr` 用类型擦除，自定义删除器写法更统一，但 `unique_ptr` 零开销更直接。

若需要「在指定线程析构对象」，可能用 **`shared_ptr` + 自定义 deleter**；单 owner 的子对象一般用默认 `delete` 即可。

## 14. 数组版本

```cpp
auto arr = std::make_unique<int[]>(100);
arr[0] = 1;
```

管理 `new T[n]` 时用 `std::unique_ptr<T[]>` 或 `make_unique<T[]>(n)`，会调用 `delete[]`。

## 15. 线程安全边界

- **移动、销毁 `unique_ptr` 本身**：若多个线程同时操作**同一个** `unique_ptr` 对象，需要同步（和普通变量一样）。
- **多个线程通过不同 `unique_ptr` 指向同一对象**：本来就不该发生（独占）。
- **对象内部数据**：`unique_ptr` 只保证释放时机，**不保证** `Foo` 内部成员并发安全。

```text
unique_ptr：保证「谁释放、何时释放」清晰
不保证：多线程同时调用 p->write() 安全
```

## 16. 什么时候用 `unique_ptr`

适合：

- 堆对象有**唯一**所有者（类成员、工厂返回值、构造时一次性接管裸指针）。
- 想用 RAII 替代析构函数里一排 `delete`。
- 所有权要在函数/容器间**转移**（`move`）。

不适合：

- 多个模块、异步任务都要延长同一对象生命 → 用 `shared_ptr`。
- 对象可以放在栈上且作用域清晰 → 直接用值类型，不必 `unique_ptr`。
- 只是观察、不拥有 → 裸指针或引用（如 `IoChannel*` 指向外部 `Loop`）。

## 17. 和 `shared_ptr` 的区别

```text
unique_ptr:
  独占所有权
  不能拷贝，只能移动
  无引用计数开销
  适合树状组合、成员子对象、工厂转移

shared_ptr:
  共享所有权
  可以拷贝
  有控制块与引用计数
  适合异步任务、连接表、回调保活

weak_ptr:
  配合 shared_ptr，不拥有对象
```

优先级（与 `shared_ptr` 笔记一致）：

```text
能用局部对象就不用指针
能用 unique_ptr 就不用 shared_ptr
确实要共享生命周期时再用 shared_ptr
```

## 18. 析构顺序（组合类必知）

成员按**声明顺序**构造，按**相反顺序**析构。

```cpp
class Worker {
    std::thread thread_;
    std::unique_ptr<Loop> loop_;
    // ...
};
```

若 `Loop` 必须先于其他成员析构，应把它声明在**更靠后**的位置（这样会**先**析构）。

应用退出时仍可能需要**显式** `stop()` / `join()`：

```text
unique_ptr 只保证 delete 顺序，不代替业务上的 shutdown 协议
```

例如：先停所有 worker 线程并 `join`，再销毁线程池，避免异步回调访问已销毁的 `Loop`。

## 19. 与其它智能指针如何分工（速查）

| 场景 | 建议 |
|------|------|
| 子对象唯一归属父对象（连接、通道、缓冲区等） | 父类里 `unique_ptr` 成员 |
| 应用级单例式组件（线程池、主循环、worker 列表） | `unique_ptr` 或 `vector<unique_ptr<>>` |
| 连接表、异步任务要共享同一会话 | `shared_ptr` + 必要时 `enable_shared_from_this` |
| 只借用外部循环/上下文 | 非拥有的裸指针或引用 |
| 工厂产出堆对象并交给唯一 owner | 返回 `unique_ptr`，或裸指针 + 构造时 `unique_ptr(raw)` 接管一次 |

## 20. 两种「移交堆对象」写法

### 写法 A：传裸指针，Owner 构造时接管（常与 `std::function` 队列搭配）

```cpp
class Owner {
public:
    Owner(Context* ctx, std::unique_ptr<Resource> res);
};

void Scheduler::attach(Resource* raw, ...) {
    loop_->post([this, raw]() {
        auto obj = std::make_shared<Owner>(
            ctx_, std::unique_ptr<Resource>(raw));
    });
}
```

### 写法 B：全程 `unique_ptr` + `move`（语义最清晰）

```cpp
void Scheduler::attach(std::unique_ptr<Resource> res, ...) {
    loop_->post([this, res = std::move(res)]() mutable {
        auto obj = std::make_shared<Owner>(ctx_, std::move(res));
    });
}
```

写法 B 若队列类型是 `std::function<void()>` 且未改成只可移动包装，会编译失败（见第 24 节）。

共同点：成员 `std::unique_ptr<Resource>` 负责释放；若还要跨线程共享 `Owner` 本身，对外仍用 `shared_ptr`。

## 21. 几个容易混淆的细节

### 1. `std::move` 并不「移动对象本身」

`move` 只是把左值转成「可被移动」状态；真正转移的是**所有权**（指针里的地址）。

### 2. 移动后的源 `unique_ptr` 仍要存在，只是为空

```cpp
auto a = std::make_unique<Foo>();
auto b = std::move(a);
// a 合法但为空，不要解引用 a
```

### 3. 函数返回 `unique_ptr` 不要写 `std::move`

```cpp
std::unique_ptr<Foo> factory() {
    return std::make_unique<Foo>();  // 自动 move，无需 std::move
}
```

### 4. `make_unique` 与数组

```cpp
auto p = std::make_unique<int[]>(n);
```

### 5. 不要用 `unique_ptr` 代替「观察者」

长期保存 `Loop*` 表示「我不拥有，只使用」是合理设计；若写成 `unique_ptr<Loop>` 却多处 `get()` 传递，所有权语义反而模糊。

### 6. 与 `enable_shared_from_this` 无关

`enable_shared_from_this` 只服务于已由 `shared_ptr` 管理的对象。`unique_ptr` 单独管理的类**不需要**继承它。

### 7. 构造里形参与成员同名（运行期 segfault）

```cpp
class Owner {
    std::unique_ptr<Resource> res_;
public:
    Owner(std::unique_ptr<Resource> res)
        : res_(std::move(res)) {
        // 错：这里的 res 是形参，已被 move 空
        channel_ = std::make_unique<Channel>(res->fd());
        // 对：用成员
        channel_ = std::make_unique<Channel>(res_->fd());
    }
};
```

或把形参改名，避免遮蔽：

```cpp
Owner(std::unique_ptr<Resource> incoming)
    : res_(std::move(incoming)) {
    channel_ = std::make_unique<Channel>(res_->fd());
}
```

检查成员是否为空应写 `res_.get()` / `this->res_`，不要只写与形参同名的标识符。

## 22. 常见错误清单

- 拷贝 `unique_ptr` 导致编译失败却强行 `.get()` 绕开所有权。
- 对 `get()` 返回的指针 `delete`。
- 两个 `unique_ptr` 从同一裸指针构造。
- 接管裸指针后，调用方又 `delete raw`（double free）。
- 移动后继续使用源指针解引用。
- 把本该 `shared_ptr` 的异步共享场景硬改成 `unique_ptr`，导致悬垂引用。
- 认为有了 `unique_ptr` 就不用管 `stop()` / `join()` 顺序。
- 在容器里存裸指针，却指望 `unique_ptr` 成员自动管理同一块内存。
- 头文件只有前向声明，却在 `.cpp` 里析构 `unique_ptr` 成员却没 `#include` 完整类型。
- `make_unique<T>` 写成没有 `()` 的形式。
- 用 `std::function` 保存捕获了 `unique_ptr` 的 lambda。
- 构造里形参与成员同名，`move` 后仍用形参名解引用（见 21.7）。

## 23. 最小记忆版

```text
unique_ptr：独占拥有，自动释放，只能移动不能拷贝
make_unique：新建堆对象并交给 unique_ptr
unique_ptr<T>(raw)：接管已有堆对象，接管后别再 delete raw
std::move：在 unique_ptr 之间转移所有权
get()：借裸指针，不转移、不 delete
单 owner 用 unique；多 owner / 异步保活用 shared
组合类成员优先 unique_ptr；观察者用裸指针或引用
传递可裸指针时，接管只在 Owner 构造做一次；形参勿与成员同名
```

## 24. 常见编译与运行期踩坑

### 编译期

**不完整类型**：`unique_ptr<T>` 析构需要 `T` 的完整定义。

```cpp
// 在会析构 unique_ptr<T> 的 .cpp 里
#include "widget.h"
```

头文件里若有 `unique_ptr` 成员，析构函数宜 **声明在 .h、定义在 .cpp**。

**`make_unique<T>` 少写 `()`**：见 3.2 节。

**`std::function` 与只可移动的 lambda**：捕获 `unique_ptr` 的 lambda 不可拷贝进 `std::function<void()>`；用写法 A（构造时接管裸指针）或改队列类型。

**拷贝 `unique_ptr`**：按值传参处写 `std::move`。

**接管后又 `delete raw`**：double free。

### 运行期

**形参遮蔽成员（见 21.7）**：`move` 进成员后，函数体内仍写与形参同名的 `ptr->...`，实际访问已空的形参 → `getFd` / 解引用 `this==nullptr` 类崩溃。`bt` 常落在构造函数里，而非业务读路径。

**异步任务未执行导致裸指针泄漏**：`stop` 后队列里仍挂着未接管的 `raw`，属于 shutdown 设计，不是 `unique_ptr` 语法问题。

### 心智模型

```text
编译器报错 ≈ 所有权没写清楚
成员用 unique_ptr；跨线程共享用 shared_ptr
std::function 队列 ↔ 构造时一次性 unique_ptr(raw)
同名形参 move 后不要再解引用形参名
```

## 25. 与 `shared_ptr` 笔记的分工

| 文档 | 侧重 |
|------|------|
| `shared_ptr.md` | 共享生命周期、异步保活、`weak_ptr` |
| 本文 | 独占所有权、`move`、裸指针接管、`make_unique` |

---

*关联：`cursor_docs/shared_ptr.md`*
