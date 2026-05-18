# `std::shared_ptr` 通用学习笔记

## 1. 它解决什么问题

裸指针只保存地址，不表达“谁负责释放对象”。

```cpp
Foo* p = new Foo();
delete p;
```

当一个对象会被多个地方使用时，手动 `delete` 很容易出错：

- 删早了：其他代码继续访问，产生 use-after-free。
- 删晚了：对象泄漏。
- 删多了：double free。

`std::shared_ptr` 用引用计数管理对象生命周期：

```text
还有 shared_ptr 指向对象：对象继续活着
最后一个 shared_ptr 销毁：对象自动析构
```

## 2. 头文件

```cpp
#include <memory>
```

`shared_ptr`、`weak_ptr`、`make_shared`、`enable_shared_from_this` 都在这个头文件里。

## 3. 创建对象

推荐：

```cpp
auto p = std::make_shared<Foo>();
```

带构造参数：

```cpp
auto p = std::make_shared<Foo>(1, "hello");
```

不优先推荐：

```cpp
std::shared_ptr<Foo> p(new Foo());
```

`make_shared` 更简洁，通常也更高效，因为对象和引用计数控制块可以一起分配。

## 4. 引用计数

```cpp
auto p1 = std::make_shared<int>(42);
auto p2 = p1;
auto p3 = p2;
```

此时三个 `shared_ptr` 指向同一个 `int`，引用计数是 3。

```cpp
p3.reset();
```

引用计数变成 2，对象还活着。

当 `p1`、`p2` 也销毁后，引用计数变成 0，对象自动析构。

可以用 `use_count()` 观察引用计数，但不要在业务逻辑里依赖它：

```cpp
std::cout << p1.use_count() << '\n';
```

## 5. 访问对象

`shared_ptr` 像普通指针一样使用：

```cpp
auto p = std::make_shared<Foo>();

p->run();
(*p).run();
```

判断是否为空：

```cpp
if (p) {
    p->run();
}
```

置空：

```cpp
p.reset();
```

## 6. 拷贝与移动

拷贝会增加引用计数：

```cpp
std::shared_ptr<Foo> p2 = p1;
```

移动会转移这个 `shared_ptr` 本身，不增加引用计数：

```cpp
std::shared_ptr<Foo> p2 = std::move(p1);
```

移动后 `p1` 为空，`p2` 持有对象。

## 7. 作为函数参数

如果函数需要共享对象生命周期，可以传值：

```cpp
void use(std::shared_ptr<Foo> p);
```

调用时会增加引用计数，函数执行期间对象一定活着。

如果函数只临时使用对象，不需要延长生命周期，优先传引用：

```cpp
void use(Foo& foo);
void use(const Foo& foo);
```

如果函数只想观察这个智能指针本身，可以传 `const std::shared_ptr<Foo>&`：

```cpp
void maybeUse(const std::shared_ptr<Foo>& p);
```

简单经验：

```text
需要共同拥有对象：传 shared_ptr
只使用对象内容：传 Foo& / const Foo&
需要保存到成员变量：传 shared_ptr 并 move / copy
```

## 8. 放进容器

```cpp
std::vector<std::shared_ptr<Foo>> vec;
vec.push_back(std::make_shared<Foo>());
```

map 中也常见：

```cpp
std::unordered_map<int, std::shared_ptr<Foo>> table;
table[id] = std::make_shared<Foo>();
```

删除：

```cpp
table.erase(id);
```

`erase` 只是让容器不再持有这份 `shared_ptr`。如果没有其他 `shared_ptr` 指向对象，对象才会析构。

## 9. 不要手动 delete

错误：

```cpp
auto p = std::make_shared<Foo>();
delete p.get(); // 错
```

`shared_ptr` 管理的对象由 `shared_ptr` 自动释放，不要手动 `delete`。

也不要这样：

```cpp
Foo* raw = new Foo();

std::shared_ptr<Foo> p1(raw);
std::shared_ptr<Foo> p2(raw); // 错
```

这会创建两个独立控制块，最终可能 delete 两次。

## 10. `get()` 的作用

```cpp
Foo* raw = p.get();
```

`get()` 只拿到底层裸指针，不转移所有权。

可以用于调用只接受裸指针的旧接口：

```cpp
legacy_api(p.get());
```

不要把 `get()` 得到的指针保存到长期结构里，也不要 `delete` 它。

## 11. `weak_ptr`

`weak_ptr` 是弱引用，不增加引用计数。

它用来表达：

```text
我不拥有这个对象
我只想在它还活着时使用它
```

创建：

```cpp
std::shared_ptr<Foo> p = std::make_shared<Foo>();
std::weak_ptr<Foo> w = p;
```

使用：

```cpp
if (auto locked = w.lock()) {
    locked->run();
} else {
    // 对象已经释放
}
```

`lock()` 成功会返回一个临时 `shared_ptr`，保证当前作用域内对象活着。

## 12. 为什么需要 `weak_ptr`

### 避免循环引用

如果两个对象互相用 `shared_ptr` 指向对方：

```cpp
struct A {
    std::shared_ptr<B> b;
};

struct B {
    std::shared_ptr<A> a;
};
```

即使外部引用都释放了，`A` 和 `B` 仍然互相持有，引用计数不会归零，造成泄漏。

解决办法：其中一边用 `weak_ptr`。

```cpp
struct B {
    std::weak_ptr<A> a;
};
```

### 异步回调中观察对象

回调不一定要延长对象生命周期，只想“对象还活着就执行”，可以捕获 `weak_ptr`。

```cpp
std::weak_ptr<Foo> weak = p;

postTask([weak]() {
    if (auto p = weak.lock()) {
        p->run();
    }
});
```

## 13. `enable_shared_from_this`

有时对象的成员函数里需要得到“管理自己的 `shared_ptr`”。

这时让类继承：

```cpp
class Foo : public std::enable_shared_from_this<Foo> {
public:
    void start();
};
```

成员函数中：

```cpp
void Foo::start() {
    auto self = shared_from_this();
}
```

`self` 是一个 `std::shared_ptr<Foo>`，和外部持有的 `shared_ptr` 共享同一个控制块。

## 14. `shared_from_this()` 的前提

对象必须已经由 `shared_ptr` 管理。

正确：

```cpp
auto p = std::make_shared<Foo>();
p->start();
```

错误：

```cpp
Foo foo;
foo.start(); // 如果 start 内部调用 shared_from_this，会错
```

错误：

```cpp
Foo::Foo() {
    auto self = shared_from_this(); // 错
}
```

构造函数执行时，`shared_ptr` 还没有完全接管对象。

## 15. 不要用 `shared_ptr<T>(this)`

错误：

```cpp
class Foo {
public:
    std::shared_ptr<Foo> self() {
        return std::shared_ptr<Foo>(this); // 错
    }
};
```

这会创建新的控制块，容易重复释放。

正确做法是继承 `enable_shared_from_this`：

```cpp
class Foo : public std::enable_shared_from_this<Foo> {
public:
    std::shared_ptr<Foo> self() {
        return shared_from_this();
    }
};
```

## 16. 自定义删除器

默认情况下，`shared_ptr` 引用计数归零时执行：

```cpp
delete ptr;
```

有时释放资源需要特殊逻辑，可以提供自定义删除器：

```cpp
std::shared_ptr<FILE> file(
    fopen("a.txt", "r"),
    [](FILE* f) {
        if (f) {
            fclose(f);
        }
    }
);
```

网络库中也可能用自定义删除器，把对象真正析构动作投递回指定线程。

## 17. 线程安全边界

`shared_ptr` 的引用计数操作是线程安全的。

这表示：

```text
多个线程可以拷贝、销毁不同的 shared_ptr 实例
引用计数不会乱
```

但它不表示对象内部是线程安全的。

```cpp
auto p = std::make_shared<Foo>();

// 两个线程同时调用
p->write();
p->write();
```

如果 `Foo::write()` 修改内部状态，仍然需要锁、线程归属或其他同步手段。

一句话：

```text
shared_ptr 保证对象活着
不保证对象内部并发访问安全
```

## 18. 什么时候用 `shared_ptr`

适合：

- 一个对象需要被多个模块共同持有。
- 异步任务需要保证对象在任务执行期间活着。
- 对象生命周期很难用单一 owner 表达。
- 容器、回调、任务队列都可能保存同一个对象。

不适合：

- 明确只有一个 owner。此时优先 `std::unique_ptr`。
- 只是临时借用对象。此时传引用或裸指针即可。
- 为了省事把所有对象都改成 `shared_ptr`。这会让所有权关系变模糊。

## 19. 和 `unique_ptr` 的区别

```text
unique_ptr:
  独占所有权
  不能拷贝，只能移动
  适合明确单 owner

shared_ptr:
  共享所有权
  可以拷贝
  适合多个地方共同持有

weak_ptr:
  不拥有对象
  用于观察 shared_ptr 管理的对象
```

优先级经验：

```text
能用局部对象就不用指针
能用 unique_ptr 就不用 shared_ptr
确实共享生命周期时再用 shared_ptr
需要观察但不拥有时用 weak_ptr
```

## 20. 一个通用异步场景例子

假设有一个对象启动异步任务，希望任务执行期间对象不被释放：

```cpp
class TaskOwner : public std::enable_shared_from_this<TaskOwner> {
public:
    void start() {
        auto self = shared_from_this();

        postTask([self]() {
            self->doWork();
        });
    }

private:
    void doWork() {
        // ...
    }
};
```

创建时必须用 `make_shared`：

```cpp
auto owner = std::make_shared<TaskOwner>();
owner->start();
```

如果不希望任务强行延长对象生命周期，可以捕获 `weak_ptr`：

```cpp
class TaskOwner : public std::enable_shared_from_this<TaskOwner> {
public:
    void start() {
        std::weak_ptr<TaskOwner> weak = shared_from_this();

        postTask([weak]() {
            if (auto self = weak.lock()) {
                self->doWork();
            }
        });
    }

private:
    void doWork() {
        // ...
    }
};
```

## 21. 几个容易混淆的细节

### 1. `shared_from_this()` 不是创建新对象

`std::make_shared<T>()` 用来创建新对象：

```cpp
auto p = std::make_shared<Foo>();
```

`shared_from_this()` 用来在已有对象的成员函数中，拿到“管理自己的 `shared_ptr`”：

```cpp
class Foo : public std::enable_shared_from_this<Foo> {
public:
    void run() {
        auto self = shared_from_this();
    }
};
```

所以分工是：

```text
外部创建对象：make_shared
对象内部获取自己：shared_from_this
```

不要在成员函数里用 `make_shared<Foo>()` 来表示自己，那会创建另一个全新的对象。

### 2. `shared_from_this()` 的使用时机

`shared_from_this()` 必须在对象已经被 `shared_ptr` 管理之后使用。

正确：

```cpp
auto p = std::make_shared<Foo>();
p->run();  // run 内部可以 shared_from_this()
```

错误：

```cpp
Foo* p = new Foo();
p->run();  // run 内部 shared_from_this() 会出错
```

错误：

```cpp
Foo::Foo() {
    auto self = shared_from_this(); // 构造函数里不要这样做
}
```

也不要在析构函数里依赖 `shared_from_this()`。它应该在对象正常存活期间、投递异步任务或保存回调之前使用。

### 3. 成员函数里是否必须写 `self->`

在普通成员函数的同步代码里，如果已经拿到了 `self`：

```cpp
void Foo::run() {
    auto self = shared_from_this();
    doWork();  // 可以，等价于 this->doWork()
}
```

这里 `self` 保证当前作用域内对象不会被释放，成员访问可以直接写。

但在异步 lambda 中，建议显式捕获并使用 `self->`：

```cpp
void Foo::run() {
    auto self = shared_from_this();

    postTask([self]() {
        self->doWork();
    });
}
```

不要为了在 lambda 里直接写成员函数而捕获裸 `this`：

```cpp
postTask([self, this]() {
    doWork(); // 能编译，但又引入了裸 this
});
```

### 4. 引用计数归零会立即析构

`shared_ptr` 不是垃圾回收。引用计数变成 0 的那一刻，对象会立即析构。

```cpp
{
    auto p = std::make_shared<Foo>();
} // p 离开作用域，Foo::~Foo() 立即执行
```

析构发生在哪个线程，取决于最后一个 `shared_ptr` 在哪个线程释放。

```cpp
std::shared_ptr<Foo> p = std::make_shared<Foo>();

std::thread t([p = std::move(p)]() mutable {
    p.reset(); // 如果这是最后一份 shared_ptr，析构就在这个线程发生
});
```

这点在多线程程序里很重要：`shared_ptr` 保证对象活着，但不保证对象一定在你期望的线程析构。

### 5. `shared_ptr` 管生命周期，状态变量管对象是否可用

`shared_ptr` 只能说明对象内存还在。

但很多对象还有自己的业务状态，例如：

```text
对象还活着，但已经关闭
对象还活着，但已经停止
对象还活着，但不允许继续执行某个操作
```

因此有时仍然需要状态变量：

```cpp
enum class State {
    Running,
    Closing,
    Closed
};
```

不要把“对象没析构”和“对象仍可用”混为一谈。

```text
shared_ptr：解决对象生命周期
状态变量：解决对象当前是否允许操作
```

### 6. 删除回调的语义会改变

使用裸指针时，某些回调可能表示：

```text
请 delete 这个对象
```

改成 `shared_ptr` 后，通常不再手动 `delete`，而是从管理容器中移除：

```cpp
table.erase(id);
```

更准确的语义变成：

```text
当前 owner 不再持有这个对象
如果没有其他 shared_ptr，对象自然析构
```

所以回调命名也可以从 `deleteXxx` 改成 `removeXxx`、`closeXxx`、`onXxxClosed`，避免误导。

## 22. 常见错误清单

- 用 `shared_ptr<T>(this)` 创建自我指针。
- 对 `shared_ptr.get()` 返回的裸指针执行 `delete`。
- 从同一个裸指针创建多个 `shared_ptr`。
- 在构造函数中调用 `shared_from_this()`。
- 认为 `shared_ptr` 能自动解决对象内部线程安全。
- 不加思考地把所有对象都改成 `shared_ptr`。
- 让两个对象用 `shared_ptr` 互相持有，形成循环引用。

## 23. 最小记忆版

```text
shared_ptr：共享拥有对象，最后一个离开时释放
weak_ptr：不拥有对象，lock 成功才能用
make_shared：创建 shared_ptr 的推荐方式
enable_shared_from_this：成员函数里安全拿到自己的 shared_ptr
shared_ptr 不等于线程安全，只保证对象生命周期
```
