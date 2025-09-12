# 更快的输入输出
## 1.解除绑定
```C++
// 必须包含此头文件
#include <iostream> 
int main() {
// 关闭同步
std::ios::sync_with_stdio(false); 
// 解除 cin 与 cout 的绑定
std::cin.tie(nullptr);            
// 使用 cin/cout 进行高效输入输出
int x;
std::cin >> x;
std::cout << "Value: " << x << "\n";
return 0;
}
```  
## 2. 把endl换成 “\n”
## 3. 参数传递尽量传递引用，无需复制
如果参数是 大容器，写 const & 更安全，性能在大多数情况下更好。

如果参数是 小对象，用值传递反而可能更快。因为可能会做RVO优化，而“访问加了一层间接寻址”反而稍微慢一点