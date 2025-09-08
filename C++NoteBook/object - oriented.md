# Inheritance  继承
## 基础的单继承
```cpp
class derived-class: access-specifier base-class{

}
//冒号继承，注意末尾有分号
```
-----
类本身的访问限制  
|访问|public|protected|private|
|----|-----|-----|-----|
|类内|yes|yes|yes|
|派生类|yes|yes|no|
|类外|yes|no|no|
----
*注意！类继承时的继承访问限制和本身的相互独立*  

***继承时的修饰符（public/protected/private）​​会限制派生类对外暴露基类成员的最高权限​​：***

•
​​public 继承​​：

基类成员的原始权限在派生类中​​保持不变​​

（public→public, protected→protected, private→不可见）

•
​​protected 继承​​：

基类的 public和 protected成员在派生类中​​降级为 protected​​

（public→protected, protected→protected, private→不可见）

•
​​private 继承​​：

基类的 public和 protected成员在派生类中​​降级为 private​​

（public→private, protected→private, private→不可见）  

***注意，C++中访问限制修饰符是写在内部针对成员的，不存在对类本身的修饰，继承修饰除外***

---

***一个派生类继承了所有的基类方法，但下列情况除外：***

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
函数名一样，参数列表不同（返回值无所谓）

重载要求所有版本在​​同一作用域​​内。派生类会​​隐藏基类同名函数​​（即使参数不同），除非使用 using声明引入基类函数：
```cpp
class Base {
public:
    void func(int);
};

class Derived : public Base {
public:
    void func(double);      // 隐藏 Base::func(int)
    using Base::func;       // ✅ 引入基类重载版本
};

Derived d;
d.func(10);   // 调用 Base::func(int)
d.func(3.14); // 调用 Derived::func(double)
```

### 运算符重载


---


# Polymorphism 多态
## 虚函数
在基类中声明一个函数为虚函数，使用关键字virtual。  
允许子类重写它（override），从而在运行时通过基类指针或引用调用子类的重写版本，实现动态绑定。  
派生类可以选择性地重写虚函数，但不是必须。
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
```cpp
class Base {
public:
    virtual void print() const;
     // const 常量成员函数，函数内部不会修改对象的成员变量
     //常量对象只能调用 const 成员函数
};

class Derived : public Base {
public:
    void print() const override; // ✅ 正确：const 一致
};
```

## 纯虚函数和抽象类（接口）
纯虚函数是没有实现的虚函数，在基类中用 = 0 来声明。  
纯虚函数表示基类定义了一个接口，但具体实现由派生类负责。  
纯虚函数使得基类变为抽象类（abstract class），无法实例化。
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
## 基类指针和派生类指针
和Java类似，基类指针可以指向派生类对象，但访问权限有限，只能访问基类已有的成员  

**运行时多态**  指针调用成员（虚函数）时，实际调用的是指针指向实际类型重写的函数
```cpp
Animal* animal = new Dog();
animal->speak();  
// 实际调用Dog::speak()而非Animal::speak()
// 静态类型：Animal*，变量声明的类型（编译时确定）
// 动态类型：Dog*,实际指向的对象类型（运行时确定）
```
**虚析构函数**
在基类指针指向派生类对象时，如果派生类含有指针和堆内存调用，必须有基类虚析构函数，避免内存泄漏
```cpp
class Base {
public:
    Base() { cout << "Base构造\n"; }
    ~Base() { cout << "Base析构\n"; } // 非虚析构函数
};

class Derived : public Base {
public:
    Derived() { 
        cout << "Derived构造\n";
        data = new int[100]; // 分配资源
    }
    ~Derived() {
        cout << "Derived析构\n";
        delete[] data; // 释放资源
    }
private:
    int* data;
};

int main() {
    Base* obj = new Derived(); // 基类指针指向派生类对象
    delete obj; // 仅调用Base的析构函数！内存泄漏
    return 0;
}

//如果声明析构函数为虚函数，输出结果：
//构造链
Base构造
Derived构造
//析构链
Derived析构  // 先调用派生类析构，确保内存不泄露
Base析构    // 再调用基类析构
```
***核心原理：静态绑定与动态绑定***  
obj的静态类型是Base*，delete操作符作用于Base*类型，查找Base类的析构函数（非虚），生成直接调用Base::~Base()的代码  

**delete函数检查对象是否有虚析构函数，通过虚函数表vtable查找实际虚构函数地址，调用后链式调用基类析构函数**  

***虚函数是实现动态绑定的关键！***  
编译阶段：为每个包含虚函数的类创建虚函数表，表中存储该类所有虚函数的实际地址，类实例包含隐藏指针vptr指向其vtable  

运行阶段：通过基类指针调用虚函数时，通过对象的vptr找到正确的vtable，从vtable中获取实际函数地址，执行派生类的函数实现 

---------
为避免内存泄漏，在小规模（多态需求弱）场景下，直接使用派生类指针管理派生类对象更简单（相比基类指针+虚函数）
## 构造链Constructor Chaining和析构链Destructor Chaining