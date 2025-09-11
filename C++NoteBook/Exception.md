# 异常处理
throw 抛出异常  
try 可能异常的代码块  
catch 捕获异常并处理
```cpp
double division(int a, int b)
{
   if( b == 0 )
   {
      throw "Division by zero condition!";//throw
   }
   return (a/b);
}
 
int main ()
{
   int x = 50;
   int y = 0;
   double z = 0;
 
   try {
    //可能异常
     z = division(x, y);
     cout << z << endl;
   }catch (const char* msg) {//异常类型为 const char*
   //处理异常
     cerr << msg << endl;
   }
 
   return 0;
}
```