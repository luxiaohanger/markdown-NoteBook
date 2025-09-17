# 那些有趣的问题（杂碎的问题）
## for_each 循环
```cpp
for(int x : check[0]) x = 1;//无法改变值，x是拷贝值
for (int &x : check[0]) x = 1;//正确写法
```
## 迭代器初始化
Cpp 容器允许使用迭代器进行初始化，只要元素类型兼容，可以跨容器初始化
```cpp
Container2 new_container(old_container.begin(), old_container.end());
```
## 容易写错的
1.写反`if` `while` ; `i` `j` ; `` ``

2.`if` 语句忘加大括号（尤其是 `break`）导致代码逻辑错误