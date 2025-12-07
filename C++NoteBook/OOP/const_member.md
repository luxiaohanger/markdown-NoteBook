##  `const` 成员函数

它允许你声明一个成员函数不会修改对象的状态。通过在成员函数声明末尾添加 `const` 关键字来定义。

### 核心特性

1.**不可修改对象状态**：
   - `const` 成员函数不能修改类的任何非静态成员变量（除非成员被声明为 `mutable`）
   - 不能调用非 `const` 成员函数（因为这些函数可能修改对象状态）

2.**常量对象支持**：
   - `const` 对象（通过 `const` 引用或指针访问的对象）只能调用 `const` 成员函数
   - 非 `const` 对象可以调用 `const` 和非 `const` 成员函数

```cpp
class MyClass {
public:
    // 非 const 成员函数
    void nonConstFunc() {
        // 可以修改成员变量
    }
    
    // const 成员函数
    void constFunc() const {
        // 不能修改成员变量
    }
};
```

### `mutable` 成员

`mutable` 是 C++ 中一个特殊的存储类说明符，用于修饰类的数据成员。它的核心作用是：​​允许在 const 成员函数中修改被声明为 mutable 的成员变量​​。

不能用于静态数据成员​​,不能用于 const 成员​

```cpp
class Logger {
    mutable int logCount; // mutable 成员可以在 const 函数中修改
    std::string message;
public:
    void log() const {
        logCount++; // 允许修改 mutable 成员
        // message = "new"; // 错误！不能修改非 mutable 成员
    }
};
```

### 函数重载
```cpp
class Display {
public:
    // 非 const 版本
    void show() {
        std::cout << "Non-const show\n";
    }
    
    // const 版本（重载）
    void show() const {
        std::cout << "Const show\n";
    }
};

int main() {
    Display d1;
    const Display d2; //const 对象
    
    d1.show(); // 调用非 const 版本
    d2.show(); // 调用 const 版本
}
```

### 返回类型
```cpp
class Data {
    std::vector<int> values;
public:
    // 返回 const 引用，防止外部修改
    const std::vector<int>& getValues() const {
        return values;
    }
};
```

1.**构造函数和析构函数**不能是 `const` 成员函数  
2.**静态成员函数**不能是 `const`（因为它们不操作特定对象实例）  
3.**成员指针**在 `const` 函数中表现为 `T* const`（指针常量），而不是 `const T*`
