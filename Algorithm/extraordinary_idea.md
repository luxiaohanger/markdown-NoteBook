# 精彩的处理
算法设计中总有一些惊为天人的设计思路，避免复杂的讨论和边界，等价转换问题

### 延迟初始化
对所给数据进行处理时，不会一次性把数据全放入容器

在相关条件和输入顺序有关时，边输入边判断（注意不要忘了后续无关输入的读入）

在需要避免 `元素自身性检查`的题目中，`处理后再插入元素` 即可避免其与本身操作

典例：LeetCode_1_两数之和
### 分组降维
对于n组数据，要从每个一维数据中选一个，朴素的做法是 `嵌套遍历` n组数据并检验,这必然造成复杂度指数级上升，如果各组数据之间的处理可以以某种方式 `独立` 进行，就能避免嵌套  

其中一种方式就是分组，例如对于四组数据，两两分组，前两组一个 `O(n^2)`,并把结果捆绑在 `unorder_map` 中，后两组遍历并查询 `unorder_map`，即可完成，这样整体复杂度就降为了 `O(n^2)`
### 如何构造不重复答案
如果答案要求返回若干元组并且不能重复，采用多指针法，从排序数组头、尾分别出发， **保证 `a <= b <= c`, 并且每次与上一次不一样**，就可以保证不出现重复答案，并且使用了分组降维
```cpp
//四数之和
vector<vector<int>> fourSum(vector<int>& nums, int target) {
    sort(nums.begin(),nums.end());
    int n=nums.size();
    vector<vector<int>> ans;
    for(int a=0;a<n-3;a++) {
        if(a>0&&nums[a]==nums[a-1])continue;
        for(int d=n-1;d-a>=3;d--) {
            if(d<n-1&&nums[d]==nums[d+1])continue;
            for(int b=a+1;b<d-1;b++) {
                if(b>a+1&&nums[b]==nums[b-1])continue;
                //c无需遍历，和b并列操作即可
                int c=d-1;
                //abd确定时，不需要考虑c是否重复因为只有一个答案
                while(b<c&&(long long)nums[a]+nums[b]+nums[c]+nums[d]>target)c--;
                if(b==c)break;
                if((long long)nums[a]+nums[b]+nums[c]+nums[d]==target)ans.push_back({nums[a],nums[b],nums[c],nums[d]});
            }
        }
    }
    return ans;
}
```
### 返回值选择
当函数返回指针时，可以利用返回的是否为空指针代替布尔值返回，方便检查是否查找到指针


### 逆向思维
在正向寻找可能导致多次重复时，不妨考虑逆向寻找

例如，寻找能到达边界的点，如果依次判断所有点是否符合条件，那么路径上的点就会重复，此时逆向思考，寻找从边界出发能到达的点，简化了思路