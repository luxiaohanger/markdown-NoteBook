# inline
inline 关键字用来 建议编译器在调用函数时将其 函数体直接展开到调用处，从而减少函数调用的开销（如压栈、跳转、返回等操作）
```cpp
inline int add(int a, int b) {
    return a + b;
}

int main() {
    int x = add(3, 4); // 编译器可能直接替换成 int x = 3 + 4;
}
```
## 适用
简单的 Getter/Setter

小的数学计算函数

模板类中的成员函数