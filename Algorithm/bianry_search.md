# 二分查找
## 适用类型
### 1.在list中寻找满足某条件的最值  
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
### 2.查找并返回
经典的二分查找，和第一类不同的是，**target可能不在list内**，因此，我们需要不存在target的退出机制，一般为left>right  
```cpp
while(left<=right){    //答案可能在 [l,r] ,没有找到时退出
   if(check1)return mid; 
   else if(check2)right=mid-1;
   else left=mid+1;
   mid=left+(right-left+1)/2;  //此时mid向哪取整都可以，因为left和right都不会赋值为mid
}
return -1;
```
