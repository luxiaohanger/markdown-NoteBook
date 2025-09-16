# 项目结构
```cpp
//a.cpp
#include "const.h"
int count=0;
int num=0;
void func1(int a){

}
 
static void process(){
   //声明为static的函数拥有文件作用域
   //它的 链接性 是内部链接（internal linkage），也就是说其他源文件无法通过 extern 或者包含头文件访问它。
   //这与默认的全局函数（非 static）不同，默认全局函数具有 外部链接（external linkage），可以被其他源文件调用。
}
```

```cpp
//a.h -->存放a.cpp中对外展示的声明
extern int num;
extern void func(int a);
```
```cpp
//const.h
const double pi = 3.14;
```
```cpp
//b.cpp
#include "a.h"
#include "const.h"
double salary;
void process(){

}
```