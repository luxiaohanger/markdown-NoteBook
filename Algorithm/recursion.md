# 递归转迭代
***自己模拟函数调用的栈***

//todo tail recursion
```cpp
int func(int n){
    if(n==0||n==1)return 1;
    else return n*func(n-1);
}



```