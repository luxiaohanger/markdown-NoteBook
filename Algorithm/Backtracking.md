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

对于不能重复的组合问题，可以标记当前元素位置，递归时从标记位置向后遍历
```cpp
// 和全排列不同之处在于不能有重复序列，因此可以考虑用顺序排除重复序列
// 增加枚举起始下标，确保每次枚举的数字都比之前大
void back_track_78(int n, vector<int>& nums, vector<int>& temp,
                   vector<vector<int>>& ans, vector<bool>& vis, int start) {
    if (temp.size() == n) {
        ans.push_back(temp);
        return;
    }
    for (int i = start; i < nums.size(); ++i) {
        if (!vis[i]) {
            temp.push_back(nums[i]);
            vis[i] = true;
            back_track_78(n, nums, temp, ans, vis, i + 1);
            temp.pop_back();
            vis[i] = false;
        }
    }
}
```