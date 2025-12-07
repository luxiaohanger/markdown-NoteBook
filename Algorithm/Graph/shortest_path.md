## Single-Source Shortest Path (SSSP)
算法中松弛操作是核心

理解松弛算法中设置 `inf` 的意义:当 `dist[u] == inf` 时，不进行操作，意思是这个距离是理论上不可以达到的，当前节点 `u` 是从源点 `s` 出发不会达到的节点，
第一个 `dist[v]` 被改变是遇见了 `dist[s] == 0` ,此时 `dist[v]` 是一个合理的距离，也即，每一个 `dist[u] != inf` , `dist[u]` 是一个合理的距离,
不会出现 `dist[u] + w` 溢出 `inf` 的情况,因此 `inf` 可以说越大越好


### 正权值单源最短路 -- Dijkstra算法
用 dist 数组记录每个顶点到源的距离，初始化为INF，每次更新时比较当前 `dist[v]` 和 `dist[u] + w`，使用贪心策略，用最小堆存储距离，每次更新当前最近顶点 `u` 的邻接顶点 `v`，处理前弹出 `u`
```cpp
int n;
priority_queue<pii> q;
vector<int> dist;
vector<vector<pii>> adj;
void sssp_dijkstra(int s){
    //initialize
    for(int i = 1;i <= n;++i){
        dist[i] = INT_MAX;
    }
    dist[s] = 0;

     for(int i = 1;i <= n;++i){
        q.push({i,dist[i]});
    }

    while(!q.empty()){
        int u = q.top().first;
        int d = q.top().second;
        q.pop();
        if(d != dist[u])continue;//pass the previous 
        for(auto it:adj[u]){
            int v = it.first;
            int w = it.second;
            if(dist[u] != INT_MAX && dist[v] > dist[u] + w){
                dist[v] = dist[u] + w;
                q.push({v,dist[v]});
            }
        }
    }
}
```
### 含负权值单源最短路 -- Bellman-Ford算法
使用 V-1 次遍历优化最短路，如果第 V 次还能优化，则说明有负权环
```cpp
int n;
vector<int> dist;
vector<vector<pii>> adj;
bool negativecircle = false;
void sssp_bellman(int s){
    //initialize
    for(int i = 1;i <= n;++i){
        dist[i] = INT_MAX;
    }
    dist[s] = 0;

    for(int t=0;t<n-1;++t){
        bool update = false;//记录是否更新
        for(int i=1;i<=n;++i){
            int u = i;
            for(auto it:adj[i]){
                int v = it.first;
                int w = it.second;
                if(dist[u] != INT_MAX && dist[v] > dist[u] + w){
                    dist[v] = dist[u] + w;
                    update = true;
                }
            }
        }
        if(!update)break;//没有更新则提前终止
    }

    //第 n 次  检测负权环
    for(int i=1;i<=n;++i){
        int u = i;
        for(auto it:adj[i]){
            int v = it.first;
            int w = it.second;
            if(dist[u] != INT_MAX && dist[v] > dist[u] + w){
                negativecircle = true;
            }
        }
    }
}
```

### 含负权值的DAG
有向无环图的SSSP可以用toposort优化，按排序顺序处理节点，确保每次处理的节点都不会被后续节点干扰
```cpp
int n;
vector<int> dist;
vector<vector<pii>> adj;
vector<int> toposort;
void sssp_DAG(int s){
    //initialize
    for(int i = 1;i <= n;++i){
        dist[i] = INT_MAX;
    }
    dist[s] = 0;

    for(int u:toposort){
        if(dist[u] == INT_MAX)continue;//不可达顶点
        for(auto edge:adj[u]){
            int v = edge.first;
            int w = edge.second;
            if(dist[u] + w < dist[v])dist[v] = dist[u] + w;
        }
    }
}
```

## All-Pair Shortest Path (APSP)
### floydWarshall 动态规划
考虑中转站动态规划，每次多开放一个中转站，直到允许所有节点作为中转站

适合稠密图
```cpp
int n;
vector<vector<int>> ans;//初始化为INT_MAX
vector<vector<pii>> adj;
void floydWarshall(){
    //initialize
    for(int i = 1;i<=n;++i){
        ans[i][i] = 0;
    }

    for(int k = 1;k<=n;++k){
        //允许1~k作为中转
        for(int i=1;i<=n;++i){
            for(int j=1;j<=n;++j){
                if(ans[i][k] == INT_MAX || ans[k][j] == INT_MAX)continue;
                if(ans[i][j] > ans[i][k] + ans[k][j])ans[i][j] = ans[i][k] + ans[k][j];
            }
        }
    }
}
```

这一算法也可用于求有向图的传递闭包，也即任意两个节点的可达性，常用邻接矩阵表示，本质上是中转站动态规划

### Johnson算法 引入势能化负为正
考虑使用重复dijkstra算法，但不适用于负路径值，因此考虑进行路径转换，只需保证转换后路径的权值差一致即可

考虑加入虚拟节点s,s到所有节点权值规定为0，s到所有节点最短路使用Bellman-Ford计算为 h[i],由于 `h[v] <= h[u] + w(u,v)`,`w_ = w(u,v) + h[u] - h[v] >= 0`,新权值非负，且可以证明满足路径的权值差一致

最后结果减去 `h[u] - h[v]`

### 如何选择？
```
图类型判断
├── 单源最短路径 (SSSP)
│   ├── 非负权重 → Dijkstra算法
│   ├── 有负权重
│   │   ├── 无环图 (DAG) → 拓扑排序 + 动态规划
│   │   ├── 有环图 → Bellman-Ford算法
│   │   └── 多次查询 → 考虑Johnson预处理
│   └── 无权图 → BFS
│
├── 全源最短路径 (APSP)
│   ├── 稠密图 (E ≈ V²) → Floyd-Warshall算法
│   ├── 稀疏图 (E << V²)
│   │   ├── 无负权重 → 重复Dijkstra算法
│   │   └── 有负权重 → Johnson算法
│   └── 特殊结构图 → 考虑专用算法
│
└── 实时/动态图 → 增量算法或近似算法
```