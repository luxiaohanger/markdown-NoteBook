# 二分查找
问题必须具有某种单调性质，即随着某个参数的增大，条件从满足变为不满足（或反之），对于给定的候选解，必须能够快速判断它是否满足条件。将许多看似复杂的问题转化为简单的决策问题（"这个值是否满足条件？"），从而以对数时间复杂度高效解决

这里的有序是广义的有序，如果一个数组中的左侧或者右侧都满足某一种条件，而另一侧都不满足这种条件，也可以看作是一种有序
## 适用类型
### 1. 在list中寻找满足某条件的最值  
*list可能是自己构造的*，（例如计算整数的平方根，就是找满足条件的最大整数）二分答案并验证，往往是最大值最小化/最小值最大化问题    
**注意，这类情形要保证答案在list内**  
把答案维护在 [left,right] 区间内，直到 left==right  
**如何保证区间一定缩小至1？**  
```cpp
while(left!=right){    //l==r时退出并返回l  
   if(check)left=mid; 
   else right=mid-1;
   mid=left+(right-left+1)/2;  //由于分支有left=mid,为了避免死循环，我们向上取整
}

while(left!=right){   
   if(check)right=mid; 
   else left=mid+1;
   mid=left+(right-left)/2;  //反之向下取整
}
```
### 2. 查找并返回
经典的二分查找，一般要求有序list,和第一类不同的是，**target可能不在list内**，因此，我们需要不存在target的退出机制，一般为left>right  
```cpp
while(left<=right){    //答案可能在 [l,r] ,没有找到时退出  注意此处不能写为 < ,不能让l==r直接退出
   if(check1)return mid; 
   else if(check2)right=mid-1;
   else left=mid+1;
   mid=left+(right-left+1)/2;  //此时mid向哪取整都可以，因为left和right都不会赋值为mid
}
return -1;
```

