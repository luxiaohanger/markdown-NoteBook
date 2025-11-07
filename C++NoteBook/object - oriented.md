# C++ 面向对象
本节用于介绍相关知识补充

## 构造和赋值
1.多数情况下，`T a = b;` 和 `T a(b);` 是等价的，这是调用构造函数的不同写法

2.当定义拷贝赋值运算符 / 移动赋值运算符 后；可以用 `=` 直接对已有实例赋值  
 `a = b;` (拷贝赋值)  
 `a = move(b);` (移动赋值)
## 构造函数
类成员创建时调用构造函数

### 无构造函数
1.编译器添加默认构造函数，此时如果类成员变量没有默认值，则值是随机值

2.如果类满足聚合类型条件：无构造函数、无私有/受保护非静态数据成员、无基类或虚函数，则可以聚合初始化 ---> 用 `{ }`初始化

3.类成员变量有显式的默认值，则初始化为默认值

4.用空 `{ }`初始化，则赋值 0 / nullptr

```cpp
class MyClass {
public:
    int value;
    std::string name;
};

// 在堆上分配（默认初始化）
MyClass* obj1 = new MyClass; 
// value未初始化（随机值），name调用默认构造函数（空字符串）

// 在堆上分配（值初始化）
MyClass* obj2 = new MyClass(); 
MyClass* obj2 = new MyClass{}; 
// value被零初始化为0，name调用默认构造函数

// 在堆上分配（聚合初始化 - C++11起）
MyClass* obj3 = new MyClass{42, "Alice"};
// value=42, name="Alice"
```
### 含参数的构造函数
```cpp
//冒号开始初始化列表 
//逗号隔开
Myclass (int a,string s):value(a),name(s){

}

//构造函数可以有默认参数，优先使用传入值，当参数不够时使用默认值
//注意：默认参数后不能有非默认参数，也即默认参数要在右边
Myclass (int a = 0,string s = “aaa”):value(a),name(s){

}

// 头文件 widget.h
class Widget {
public:
    Widget(int w = 1280, int h = 720); // 声明处指定默认参数
};

// 源文件 widget.cpp
Widget::Widget(int w, int h) : width(w), height(h) {} 
// 定义中不能再重复默认参数
```
### 拷贝构造函数
```cpp
Myclass(const Myclass& a){
    //把新对象的成员值赋为和 a 一样
}
```
### 移动构造函数  
1.移动构造函数的参数是一个**右值引用** (`ClassName&&`)。这个参数类型就像一个信号灯，告诉编译器：“这个函数专门用来处理那些即将消亡的临时对象（右值）”。

2.当用一个**右值**（例如函数返回的临时对象、`std::move` 转换后的对象）来初始化一个新对象时，编译器会优先选择移动构造函数（如果存在），而不是拷贝构造函数。

3.在移动构造函数内部，因为你知道传入的 `other` 是一个即将消亡的临时对象（通过右值引用标识），所以你可以安全地“窃取”它的资源（如动态内存指针、文件句柄等），而不是进行深拷贝。

```cpp
class MyVector {
private:
    int* m_data;
    size_t m_size;

public:
    MyVector(MyVector&& other) noexcept // 右值引用参数
        : m_data(other.m_data), m_size(other.m_size) { 
        // 初始化列表直接接管指针和大小
        // 关键步骤：将源对象置于安全状态
        other.m_data = nullptr; // 使源对象可安全析构
        other.m_size = 0;
    }

    ~MyVector(){
        delete[] m_data;
        m_data = nullptr;
    }
}

int* data = new int[3]{1, 2, 3};  // 正确分配数组
MyVector v1{data, 3};
MyVector v2(move(v1));//栈分配
MyVector* v2 = new MyVector(move(v1));//堆分配
 ```
 ### 构造链调用
 派生类构造函数会调用基类构造函数，可以用初始化列表显式指定调用哪个构造函数，否则调用默认构造函数
 ```cpp
 Child_class(int a):base_class(a){
    //
 }
 ```

## 析构函数
类成员被销毁时，自动调用析构函数  

*但何时销毁是个需要理解的问题！*  

1.在栈空间分配的类对象，栈销毁时调用析构函数  

2.堆空间分配的类对象，程序运行期间不会被销毁，因此需要手动调用`delete`，进而调用析构函数

3.类作为其他类成员变量时遵循析构链执行  

4.类指针作为类成员变量时，销毁的是指针本身，而非其指向的类实例，需要在调用者析构函数中手动销毁  

5.`delete` 会调用析构函数，`free` 不会

### delete函数
1.必须和 `new` 函数配对使用  

2.注意指针重复（同一个对象被多个指针指向），此时不能多次 delete

3.delete操作​​不会​​改变指针变量本身的值。指针在 delete之后仍然指向原来的内存地址（那块内存现在已被释放，是无效的）。  
--> *在 delete之后，立即将指针设置为 nullptr！*  
--> 将指针置为 nullptr后，delete nullptr 是安全的,避免多次 delete 的问题

4.对于分配的数组 / STL 内存：
```cpp
//使用 delete[]当且仅当内存是通过 new[]分配的数组
//会依次调用数组中每个元素的析构函数（如果是类对象数组）
//按​​逆序​​（从最后一个元素到第一个）调用每个元素的析构函数
classT* pointer = new classT[n]; 
-->
delete[] pointer;

//对于此类STL指针数组，需要在析构函数中手动遍历管理内存
vector<classA*> arr;
-->
for(auto& p:arr){
    delete p;
    p = nullptr;
}
``` 
5.二维数组分配与释放
```cpp
const int ROWS = 3; 
const int COLUMNS = 4;
 
char **chArray2; 

// allocate the rows 
chArray2 = new char* [ ROWS ]; 

// allocate the (pointer) elements for each row 
for (int row = 0; row < ROWS; row++ ) 
	chArray2[ row ] = new char[ COLUMNS ]; 

//delete
for (int row = 0; row < ROWS; row++) 
{ 
	delete [ ] chArray2[ row ]; 
	chArray2[ row ] = NULL; 
} 

delete [ ] chArray2; 
chArray2 = NULL; 
```
## Constructor Chaining & Destructor Chaining
当一个对象被创建时，C++ 会按照一定的规则去调用构造函数，这个过程叫做 构造链。 

它遵循 ***“先父后子，先成员后自己”*** 的原则。  

1.*基类构造函数*

如果有继承关系，会先调用基类的构造函数。
如果有多个基类，按照 继承列表中的声明顺序 来构造。

2.*成员对象构造函数*

然后调用类中 成员对象（类类型成员） 的构造函数。
调用顺序是 成员变量在类中定义的顺序，而不是初始化列表的书写顺序。

3.*当前类自身的构造函数*

最后才调用派生类本身的构造函数体

**析构链与之相反**

--------
## this 指针
this是一个隐藏的指针，可以在类的成员函数中使用，它可以用来指向调用对象

每一个对象都能通过 this 指针来访问自己的地址

当一个对象的成员函数被调用时，编译器会隐式地传递该对象的地址作为 this 指针

静态成员函数、友元函数没有 this 指针
## 友元
使用 `friend` 关键字，在类内部声明，在类外部定义（否则可能出现访问问题），友元函数 / 友元类 可以访问声明类的所有成员

友元不是任何类的成员，在类外部定义后直接调用,同理友元也没有 `this` 指针

友元的访问权限开放只针对声明此友元的类，对于其基类、派生类访问权限不变
```cpp
class MyClass {
private:
    int secretData;

    // 声明一个友元函数 (在类内部)
    friend void friendFunction(MyClass& obj);

    // 声明一个友元类 (在类内部)
    friend class FriendClass;

public:
    // ... 其他成员 ...
};

// 在类外部定义友元函数
void friendFunction(MyClass& obj) {
    obj.secretData = 42; // 可以访问私有成员 secretData
}

// 在别处定义友元类
class FriendClass {
public:
    void modifySecret(MyClass& obj) {
        obj.secretData = 100; // FriendClass 的成员函数可以访问 MyClass 的私有成员
    }
};
```
## 内联函数
C++习惯上把成员声明放在类定义内，把成员函数定义放在类外，在类定义中的定义的函数都是内联函数，即使没有使用 `inline` 说明符

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

----------
##  `static` 成员

使用 `static` 修饰的类成员属于**类本身**而非类的实例



| **特性**             | **静态成员**                                  | **普通成员**                          |
|----------------------|---------------------------------------------|---------------------------------------|
| **存储位置**         | 全局数据区（独立于对象）                    | 对象内存中（每个实例独立存储）        |
| **生命周期**         | 程序启动时创建，程序结束时销毁              | 对象创建时生成，对象销毁时释放        |
| **内存占用**         | 类声明中不占用对象空间                      | 每个对象都包含成员副本                |
| **访问方式**         | 通过 `类名::成员` 或 `对象.成员`            | 必须通过对象实例访问                  |
| **初始化位置**       | 类外初始化（全局作用域）                    | 构造函数初始化列表或函数体内          |
| **成员函数访问权限** | 只能访问其他静态成员                        | 可访问所有成员（包括静态和非静态）    |
| **this 指针**        | 无 this 指针                                | 隐含 this 指针                        |


```cpp
class Counter {
public:
    static int count;  // 声明（不分配内存）
    
    Counter() { ++count; }
    ~Counter() { --count; }
};

// 类外初始化（必须）
int Counter::count = 0; 
```

**特点：**
- 所有对象**共享**同一份数据
- 初始化必须在**类外全局作用域**完成
- 可用 `constexpr` 在类内初始化（C++17）
  ```cpp
  class MathConstants {
      static constexpr double PI = 3.1415926; // C++17
  };
  ```
## final
`final` 是一个​​上下文关键字

1.修饰虚函数，阻止派生类​​重写 (override)​​ 该虚函数,表明该虚函数是继承链的最终实现

2.修饰类，阻止该类被其他类​​继承​​。放在类定义的​​类名​​之后，`{` 之前

3.修饰纯虚函数，表明这个函数必须被实现，并且一旦实现就绑定不变
```cpp
virtual void mustBeImplemented() const = 0 final; 
```
# Inheritance  继承
## 单继承
```cpp
class derived-class: access-specifier base-class{

};
//冒号继承，注意末尾有分号
```
-----
***类成员的访问限制修饰***  
|访问|public|protected|private|
|----|-----|-----|-----|
|类内|yes|yes|yes|
|派生类|yes|yes|no|
|类外|yes|no|no|

类默认 private ， 结构体默认 public

函数重写时，可以自定义访问控制权限，但是权限是静态绑定的

函数重载一般要有一样的访问权限

----

***继承访问限制修饰符​***

`子类成员访问限制权限高于继承权限的降级为继承权限`

•
​​public 继承​​：基类成员的原始权限在派生类中​​保持不变​​

（public → public, protected → protected, private → 不可见）

•
​​protected 继承​​：public 降级为 protected

（public → protected, protected → protected, private→不可见）

•
​​private 继承​​：全部​​降级为 private​​

（public → private, protected → private, private → 不可见）  

***注意，C++中访问限制修饰符是写在内部，针对成员的，不存在对类本身的修饰***

---

*一个派生类继承了所有的基类方法，但下列情况除外：*

基类的构造函数、析构函数和拷贝构造函数。
基类的重载运算符。
基类的友元函数。

## 多继承
```cpp
class <派生类名>:<继承方式1><基类名1>,<继承方式2><基类名2>,…
{
<派生类类体>
};

```
## 重载 Overloading
### 函数重载
`函数名一样，参数列表不同（返回值无所谓）`



| 特性         | 同一作用域内的函数重载                                     | 派生类中定义同名函数导致基类函数被隐藏                     |
| :----------- | :--------------------------------------------------------- | :--------------------------------------------------------- |
| **作用域**   | **同一个作用域内**（例如同一个类内部、全局作用域）。       | **不同作用域**（派生类作用域 vs 基类作用域）。             |
| **名称查找** | 编译器查找名称时，会**同时看到该作用域内的所有同名函数**。 | 编译器在派生类作用域**找到名称后即停止查找**，**不会**再去基类作用域查找同名名称。 |
| **可见性**   | 所有重载版本在名称查找阶段都是可见的。                     | 基类中的同名函数在派生类作用域内**不可见**（被隐藏），除非显式引入（如 `using`）。 |
| **重载解析** | 编译器在名称查找找到的所有同名函数中进行**重载解析**，选择参数最匹配的那个。 | **名称查找阶段**就决定了基类函数不会被考虑。**重载解析只在派生类找到的那个函数上进行**（如果参数不匹配，直接报错）。 |
| **错误类型** | 如果调用时没有参数匹配的重载版本，错误是 **“没有匹配的函数”**（重载解析失败）。 | 如果尝试调用被隐藏的基类函数（即使参数匹配），错误是 **“‘基类函数名’在‘派生类名’中不可访问”或类似**（名称查找失败）。 |
| **目的**     | 提供相同功能的不同接口（基于不同参数）。                   | **派生类引入新功能**或**有意覆盖基类行为**（即使参数不同）。 |
| **解决冲突** | 不需要特殊语法。                                           | 需要使用 `using Base::functionName;` 将基类函数引入派生类作用域，使其参与重载解析。 |



### 运算符重载
重载的运算符是带有特殊名称的函数，函数名是由关键字 `operator` 和其后要重载的运算符符号构成的

仿函数则是对函数调用运算符 `()` 进行重载
```cpp
class A{
    int val_a;
    int val_b;

    A operator+(const A& other) const { 
        return {
            val_a + other.val_a,
            val_b + other.val_b
        };
    }
};

//main
A a{1,2}; // 使用聚合初始化
A b{3,4};
A c = a + b; 
```

---


# Polymorphism 多态
## 虚函数
在基类中声明一个函数为虚函数，使用关键字 `virtual`。  

允许子类重写它（使用 `override` 关键字），从而在运行时通过基类指针或引用调用子类的重写版本，实现动态绑定。  

派生类可以选择性地重写虚函数，但不是必须。

重写函数自动成为虚函数,因此虚函数一旦声明就可以不断被派生类重写

### 核心机制总结
| **特性**       | **虚函数**                     | **非虚函数**                  |
|----------------|-------------------------------|------------------------------|
| **绑定方式**   | 动态绑定（运行时）             | 静态绑定（编译时）           |
| **决定因素**   | 对象的**动态类型**（实际类型） | 指针的**静态类型**（声明类型）|
| **调用规则**   | 调用本类虚函数表最终版本             | 调用基类版本                 |
| **实现机制**   | 通过虚函数表（vtable）查找     | 直接编译时确定地址           |

```cpp
class Base {
public:
    virtual void show() { 
        cout << "Base show" << endl; 
    }
};

class Derived : public Base {
public:
    void show() override {  // ✅ 正确重写
        cout << "Derived show" << endl;
    }
};
```
协变返回类型允许（派生类返回类型可以是基类返回类型的派生类）  

const 限定必须一致（同 const 或同非 const）

## 虚函数表
### 📌 虚函数表（vtable）的本质

1.  **每个类有自己的 vtable**
    *   每个包含虚函数（或继承虚函数）的类都有**自己独立的虚函数表**
    *   即使是派生类，也有自己的 vtable（继承自基类但可能被修改）
    *   **vtable 是按类创建的，不是按对象创建的**

2.  **vtable 的内容**
    *   vtable 是一个**函数指针数组**
    *   每个槽位（slot）对应类中的一个虚函数
    *   存放的是**该类提供的最新实现版本**的函数指针

### 📌 vtable 是如何构建的？

编译器在编译时为每个类生成 vtable，规则如下：

| 步骤 | 行为 |
|------|------|
| **1. 基类 vtable** | 按声明顺序包含所有虚函数的指针（指向基类自己的实现） |
| **2. 派生类 vtable** | 从基类复制 vtable 结构 |
| **3. 重写处理** | 如果派生类重写虚函数，**替换对应槽位的指针**为派生类版本 |
| **4. 新增虚函数** | 在 vtable **末尾添加**新槽位 |


### 📌 虚函数调用机制

当通过基类指针调用虚函数时：


1.  **获取 vptr**
    *   从对象头部获取虚表指针（vptr）→ 指向本类的 vtable

2.  **确定槽位位置**
    *   编译器知道 虚函数 在 vtable 中的位置
    

3.  **跳转到函数**
    *   访问 vtable[i] → 得到 `&X::func1`
    *   执行 `X::func1()`




## 纯虚函数和抽象类（接口）
纯虚函数是没有实现的虚函数，在基类中用 `= 0` 来声明。  

纯虚函数表示基类定义了一个接口，但具体实现由派生类负责。  

纯虚函数使得基类变为抽象类（abstract class），无法实例化。  

抽象类里面也可以有非抽象函数，可能为派生类共有的成员函数

纯虚函数参数规则与普通函数相同，派生类必须实现所有纯虚函数，实现时参数列表必须与声明严格匹配（除默认参数外，默认参数值在编译时根据​​静态类型​​确定，而非运行时动态类型）
```cpp
class Shape {
public:
    virtual int area() = 0;  // 纯虚函数，强制子类实现此方法
};
 
class Rectangle : public Shape {
private:
    int width, height;
public:
    Rectangle(int w, int h) : width(w), height(h) { }
    
    int area() override {  // 实现纯虚函数
        return width * height;
    }
};
```
## 基类指针指向派生类对象
和Java类似，基类指针可以指向派生类对象，但函数绑定有动态静态之分

**核心：静态绑定与动态绑定**  

函数是否动态绑定主要由虚函数关键字 `virtual` 决定：当函数是虚函数并且通过指针 / 引用调用时，才能动态绑定

*基类指针指向派生类对象时，对象的静态类型是基类，调用函数时：对于虚函数，会动态绑定，检查虚函数表；对于非虚函数，静态绑定只调用基类成员*


```cpp
Animal* animal = new Dog();
animal->speak();  
// 实际调用Dog::speak()而非Animal::speak()
// 静态类型：Animal*，变量声明的类型（编译时确定）
// 动态类型：Dog*,实际指向的对象类型（运行时确定）
```
### 虚析构函数
在基类指针指向派生类对象时，如果派生类含有指针和堆内存调用，基类析构函数必须是虚析构函数，避免内存泄漏

**delete函数检查对象是否有虚析构函数，通过虚函数表 vtable 查找实际虚构函数地址，调用后链式调用基类析构函数**  

```
定义一个函数为虚函数，不代表函数为不被实现的函数。

定义他为虚函数是为了允许用基类的指针来调用子类的这个函数。

定义一个函数为纯虚函数，才代表函数没有被实现。

定义纯虚函数是为了实现一个接口，起到一个规范的作用，规范继承这个类的程序员必须实现这个函数。
```
---------
为避免内存泄漏，在小规模（多态需求弱）场景下，直接使用派生类指针管理派生类对象更简单（相比基类指针+虚函数）

# OOP 的文件级应用
简单介绍 `.h`（头文件）和 `.cpp`（源文件）的经典配合方式

**核心思想：接口与实现分离**

1.  **`.h` 文件 (头文件)：**
    *   **职责：声明接口 (Interface Declaration)**
    *   **包含内容：**
        *   **类定义 (Class Definition)：** 这是头文件的核心。
            *   **成员变量 (Member Variables)：** 声明类的数据成员（通常是 `private` 或 `protected`）。
            *   **成员函数声明 (Member Function Declarations)：** 声明类的所有成员函数（`public`, `protected`, `private`），**但不包含函数体（定义）**。例如：
                ```cpp
                // MyClass.h
                class MyClass {
                public:
                    MyClass(int value); // 构造函数声明
                    int getValue() const; // 成员函数声明
                    void setValue(int newValue); // 成员函数声明
                    void doSomethingComplex(); // 成员函数声明 (假设实现复杂)

                private:
                    int secretData_; // 成员变量声明
                };
                ```
            *   **友元声明 (Friend Declarations)：** 声明友元函数或友元类。
        *   **内联函数定义 (Inline Function Definitions)：** 对于**非常小、简单**的函数（如 `getter`/`setter`），可以直接在类定义内部**声明并定义**（隐式内联）。例如：
            ```cpp
            // MyClass.h (续)
            class MyClass {
                // ... 其他声明 ...
            public:
                // 小函数直接在类内定义 (隐式 inline)
                int getValue() const { return secretData_; }
                void setValue(int newValue) { secretData_ = newValue; }
            };
            ```
        *   **必要的类型定义和前置声明：** `typedef`, `using`, `enum class`, 其他类的 `class SomeOtherClass;`（前置声明）等。
        *   **包含必要的其他头文件：** 如果类声明中使用了其他类型（如 `std::string`, 自定义类等），需要包含对应的头文件 (`#include <string>`, `#include "OtherClass.h"`)。**目标是只包含绝对必要的头文件，以最小化依赖。**
    

2.  **`.cpp` 文件 (源文件)：**
    *   **职责：实现细节 (Implementation Details)**
    *   **包含内容：**
        *   **包含对应的 `.h` 文件：** `#include "MyClass.h"`。这是必须的，因为实现需要知道类的完整定义（包括私有成员）。
        *   **成员函数定义 (Member Function Definitions)：** 使用作用域解析运算符 (`::`) 定义在 `.h` 文件中**声明但未在类内定义**的成员函数。例如：
            ```cpp
            // MyClass.cpp
            #include "MyClass.h"

            // 构造函数定义
            MyClass::MyClass(int value) : secretData_(value) {}

            // 复杂成员函数定义 (在 .h 中只有声明)
            void MyClass::doSomethingComplex() {
                // ... 复杂的实现逻辑 ...
                // 可以自由使用 secretData_ 和其他私有成员
            }

            // 注意：getValue 和 setValue 已经在 .h 的类内定义了，这里不需要再定义！
            ```
        *   **静态成员变量定义 (Static Member Variable Definitions)：** 如果类中有 `static` 成员变量，其声明在 `.h` 中，但定义（分配存储空间）通常在 `.cpp` 文件中（除非是 `constexpr` 或 `inline` 变量）。
        *   **友元函数定义 (Friend Function Definitions)：** 如果友元函数不是在类内定义的（即不是内联友元），通常也定义在 `.cpp` 文件中。
        *   **其他辅助函数或局部实现细节。**
    *   **目的：**
        *   隐藏具体的实现逻辑。
        *   可以自由包含实现所需的额外头文件（如 `<algorithm>`, `<fstream>` 等），而不会将这些依赖“传染”给所有包含 `.h` 文件的代码。
        *   修改 `.cpp` 文件中的实现（只要不改变 `.h` 文件中的接口）通常只需要重新编译该 `.cpp` 文件本身，大大缩短大型项目的编译时间（**编译防火墙**）。

**OOP 中的配合流程 (以 `MyClass` 为例)**

1.  **设计接口 (`MyClass.h`):**
    ```cpp
    // MyClass.h
    #pragma once // 或 #ifndef MYCLASS_H ... 防止重复包含

    class MyClass {
    public:
        // 构造函数声明
        MyClass(int value);
        // 小函数直接在类内定义 (接口的一部分)
        int getValue() const { return secretData_; }
        void setValue(int newValue) { secretData_ = newValue; }
        // 复杂函数只声明
        void doSomethingComplex();

    private:
        int secretData_; // 数据封装
    };
    ```

2.  **实现细节 (`MyClass.cpp`):**
    ```cpp
    // MyClass.cpp
    #include "MyClass.h" // 必须包含自己的头文件
    #include <iostream>  // 实现可能需要额外头文件 (不影响 MyClass.h 的使用者)

    // 构造函数定义
    MyClass::MyClass(int value) : secretData_(value) {}

    // 复杂成员函数实现
    void MyClass::doSomethingComplex() {
        std::cout << "Secret data is: " << secretData_ << std::endl;
        // ... 其他复杂操作 ...
    }
    ```

3.  **使用类 (`main.cpp` 或其他模块):**
    ```cpp
    // main.cpp
    #include "MyClass.h" // 只需要包含头文件，了解接口

    int main() {
        MyClass obj(42);       // 创建对象 (编译器知道构造函数签名)
        obj.setValue(100);     // 调用小函数 (定义在头文件里)
        int val = obj.getValue(); // 调用小函数
        obj.doSomethingComplex(); // 调用函数 (实现在 .cpp, 链接器负责连接)
        return 0;
    }
    ```

    ## 声明和定义
    

| **特征**                | **声明 (Declaration)**                          | **定义 (Definition)**                          | **必须一致** | **特殊规则**                                                                 |
|-------------------------|------------------------------------------------|------------------------------------------------|--------------|-----------------------------------------------------------------------------|
| **非静态成员变量**      | `int value;`                                   | 类内初始化：`int value = 10;` (C++11+)         | 是           | 1. 类型必须完全匹配<br>2. 类外不能重新定义                                  |
| **静态成员变量**        | `static int count;`                            | 类外定义：`int Class::count = 0;`              | 是           | 1. C++17支持`inline static`类内定义<br>2. 类外定义不带`static`关键字        |
| **普通成员函数**        | `void func(int param);`                        | `void Class::func(int param) { ... }`          | 签名必须一致 | 1. 定义需类名限定<br>2. `const`/`volatile`限定符必须一致                   |
| **虚函数**              | `virtual void vfunc() const;`                  | `void Class::vfunc() const { ... }`            | 签名必须一致 | 1. 派生类声明可用`override`<br>2. 定义处禁止`virtual`/`override`           |
| **纯虚函数**            | `virtual void pure() = 0;`                     | 可选类外定义：`void Class::pure() { ... }`     | 签名必须一致 | 1. 声明必须有`=0`<br>2. 定义不能有`=0`                                     |
| **构造函数**            | `ClassName(params);`                           | `ClassName::ClassName(params) : init{...} {...}` | 名称必须一致 | 1. 无返回类型<br>2. 初始化列表仅在定义处                                   |
| **析构函数**            | `~ClassName();`                                | `ClassName::~ClassName() { ... }`              | 名称必须一致 | 1. 无返回类型<br>2. 虚析构函数在派生类定义中不需`virtual`                  |
| **模板成员函数**        | `template<typename T> void tfunc(T param);`    | `template<typename T> void Class::tfunc(T param) {...}` | 是           | 1. 通常声明定义在一起<br>2. 类外定义需完整模板前缀                         |
| **静态成员函数**        | `static void sfunc();`                         | `void Class::sfunc() { ... }`                  | 签名必须一致 | 1. 定义不带`static`<br>2. 不能使用`const`/`volatile`                       |
| **默认参数**            | 声明处指定：`void f(int x = 0);`               | 定义处禁止指定默认参数                         | -            | 只能在**一处**指定（通常在声明处）                                         |
| **constexpr成员**       | `constexpr int size = 10;` (C++11+)            | 类内直接初始化                                 | 是           | 1. 必须是字面值类型<br>2. C++14起可在类外定义                              |
| **友元函数**            | `friend void friendFunc();`                    | 类外独立定义：`void friendFunc() { ... }`      | 不适用       | 1. 声明在类内<br>2. 定义在类外全局作用域                                   |

