# 更快的输入输出
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
