# Op_&
## 引用声明 
```cpp
int& b=a;
//b是a的一个别名，实际在内存中是同一块地址(不能绑定其他变量，声明必须初始化)
```
## 取地址符
```cpp
int* p= &a;
```
## 按引用传递参数
```cpp
void swap(int &a, int &b) {
    int temp = a;
    a = b;
    b = temp;
}
```
## 按位与
```cpp
int a = 5;     // 二进制 0101
int b = 3;     // 二进制 0011
int c = a & b; // 结果 0001（十进制 1）
```
## 函数返回引用
返回对象的别名（而非拷贝），可直接修改原始对象  
有利于链式调用、作为左值操作、避免大数据拷贝
```cpp
class StringBuilder {
    std::string data;
public:
    StringBuilder& append(const std::string& str) {
        data += str;
        return *this;  // 返回当前对象的引用
    }
};
// 链式调用
StringBuilder().append("Hello").append(" World");
```
**注意不要返回局部变量引用**

即使是实例，使用引用会导致绕过RVO！不涉及拷贝  
*一般参数传入引用，返回此引用，全程操作原变量*
## 成员函数&
用于限定成员函数的调用类型
```cpp
struct Foo {
    void print() & {
        cout << "左值对象调用\n";
    }
    void print() && {
        cout << "右值对象调用\n";
    }
};

int main() {
    Foo f;
    f.print();       // 左值对象 -> 调用 print() &
    Foo().print();   // 右值对象 -> 调用 print() &&
}
```