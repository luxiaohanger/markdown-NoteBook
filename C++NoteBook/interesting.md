# 那些有趣的问题
## for_each 循环
```cpp
for(int x : check[0]) x = 1;//无法改变值，x是拷贝值
for (int &x : check[0]) x = 1;//正确写法
```
## if? while?
这两个怎么会写反呢