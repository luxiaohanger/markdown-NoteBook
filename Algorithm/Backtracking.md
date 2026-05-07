# 回溯
回溯算法本质上是利用递归进行穷举，适用于求所有可能解的情形

狭义上，回溯也是DFS的一种，只不过是加了一些处理（加入节点、删除节点）和判断终止

## 模板
```cpp
void backtracking(){
    if( 符合条件 ){
        存储答案
        return;
    }

    for(遍历可能分支){
        加入分支到预存储的答案
        backtracking(进入分支)
        回溯（删除此分支）
    }
}
```

对于路径寻找类型的递归，可以使用标识位标记是否寻找结束
```cpp
void save_path(T* now,T* target,vector<T*>& path,bool& find){
    if(find || !now)return;
    path.push_back(now);
    if(now == target){
        find = true;
        return;
    }
    for(auto x : next){
        save_path(x,target,path,find);
    }
    if(!find)path.pop_back();
}
```