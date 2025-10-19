# list 双向链表类
```c++
list<int> temp;
// 移除前n个元素到临时链表
auto it = arr.begin();
//使用自动类型匹配 auto声明节点，it++/--即可切换节点

for (int i = 0; i < k; ++i) {
//splice 是操作list元素转移的高效函数，几乎都是O(1)
    temp.splice(temp.end(), arr, it++);
}

// 反转临时链表
temp.reverse();
// 将反转后的元素插回原位置
arr.splice(arr.begin(), temp);
```


## splice函数
```c++
//splice 是操作list元素转移的高效函数，几乎都是O(1)
void splice(const_iterator pos, list& other);
//将 other的所有元素转移到当前链表的 pos位置 前！
void splice(const_iterator pos, list& other, const_iterator it);
//转移单个元素
```

## 迭代器
在 C++ 中，所有容器的迭代器范围都遵循 ​​左闭右开​​ 原则：  
begin()指向第一个元素，end()指向​​最后一个元素的下一个位置
```cpp
for (auto it = container.begin(); it != container.end(); ++it) {
    // 处理 *it
}
```

**注意不要和 front/back 混淆**
## erase 函数
```cpp
//删除单个元素
iterator erase (iterator position);
//删除[first,end)  左闭右开
iterator erase (iterator first, iterator last);

//返回值 返回迭代器，指向（最后一个）被删除元素的下一个元素
 for(auto it = arr.begin();i <= k; ) {
        if(i % 2 == 1)it = arr.erase(it);
        else it ++;
    }
```
由于erase函数会更新迭代器，因此循环条件要注意返回值要不要更新
