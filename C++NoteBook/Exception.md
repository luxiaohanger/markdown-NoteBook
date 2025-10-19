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

## 常见运行错误



### **1. 访问冲突（Access Violation）**
- **错误代码**：`0xC0000005` (Windows) / `SIGSEGV` (Linux/macOS)
- **触发场景**：
  ```c
  int *ptr = nullptr;
  *ptr = 42;  // 解引用空指针

  int arr[5] = {0};
  arr[10] = 1;  // 数组越界

  int *p = (int*)malloc(sizeof(int));
  free(p);
  *p = 10;      // 使用已释放内存
  ```
- **本质原因**：访问了进程无权操作的内存地址（空指针、已释放内存、只读内存等）
---

### **2. 堆损坏（Heap Corruption）**
- **错误代码**：`0xC0000374` (STATUS_HEAP_CORRUPTION)
- **触发场景**：
  ```c
  char *buffer = (char*)malloc(10);
  strcpy(buffer, "This string is too long!");  // 缓冲区溢出
  free(buffer);  // 触发堆损坏

  int *p = new int[10];
  delete p;      // 错误：应使用 delete[]
  ```
- **根本原因**：
  - 缓冲区溢出覆盖堆管理结构
  - 错误的内存释放方式（`new[]`/`delete` 不匹配）
  - 双重释放（Double Free）


---

### **3. 整数除零（Integer Divide by Zero）**
- **错误代码**：`0xC0000094` (STATUS_INTEGER_DIVIDE_BY_ZERO)
- **示例**：
  ```c
  int a = 10, b = 0;
  int c = a / b;  // 触发异常
  ```
- **特殊注意**：浮点数除零不会崩溃，而是产生 `INF` 或 `NaN`

---

### **4. 未处理异常（Unhandled Exception）**
- **错误代码**：`0xC0000005` (通用异常代码)
- **常见类型**：
  - **C++ 异常**：未被捕获的 `std::exception`
  ```cpp
  throw std::runtime_error("Critical failure");
  ```
  - **系统异常**：如非法指令（`SIGILL`）、浮点异常（`SIGFPE`）

---

### **5. 断言失败（Assertion Failure）**
- **错误代码**：非系统级错误（程序主动终止）
- **示例**：
  ```c
  #include <cassert>
  void critical_function(int x) {
      assert(x > 0 && "x must be positive");  // 断言
      // ...
  }
  ```


---

### **6. 资源耗尽错误**
| 错误类型          | 错误代码/表现               | 触发原因                  |
|-------------------|---------------------------|--------------------------|
| **内存不足**      | `bad_alloc` (C++)         | `new` 分配失败            |
| **句柄泄漏**      | `ERROR_TOO_MANY_HANDLES`   | 未关闭文件/内核对象       |
| **线程创建失败**  | `STATUS_WORKING_SET_QUOTA` | 线程栈空间或数量超限      |

---

### **7. 多线程相关错误**
- **数据竞争（Data Race）**：
  ```cpp
  // 未加锁的共享变量访问
  int counter = 0;
  void increment() { counter++; }  // 多线程调用导致未定义行为
  ```
- **死锁（Deadlock）**：
  ```cpp
  std::mutex m1, m2;
  // 线程1：锁定 m1 后尝试锁 m2
  // 线程2：锁定 m2 后尝试锁 m1
  ```

### **8. 栈溢出 0xC00000FD**
