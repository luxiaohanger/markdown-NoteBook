## 双连通分量 割点/桥
在一张连通的无向图中，对于两个点 𝑢 和 𝑣，如果无论删去哪条边（只能删去一条）都不能使它们不连通，我们就说 𝑢 和 𝑣 边双连通

对于一个极大的边双连通子图，称之为一个边双连通分量

对于一个无向图，如果把一个点删除后这个图的极大连通分量数增加了，那么这个点就是这个图的割点（又称割顶） 

如果删除一条边后连通分量增加了，就称之为割边/桥（没有桥的极大连通子图就是一个边双连通分量）
### 双连通分量
#### 边双连通分量
DFS树中边分两类，`树边`：构成DFS树的边；`回边`：指向非父祖先节点的边

我们利用DFS过程判断邻接节点是否可以通过 回边/子树的回边 直接连接其祖先节点，如果可以则是边双连通分量，反之是桥
```cpp
vector<vector<int>> ebccs;//边双连通分量组
int time = 1;//时间戳从1开始 0代表未访问
vector<int> dfn(n+1,0);//节点对应访问时间戳 非父节点 && 时间戳大小 => 祖先 
//除了父节点，此节点能到达的已经被访问的节点就是祖先
vector<int> low(n+1);//节点及其的子树节点，能通过回边到达的最早祖先的dfn值，注意不是low值
stack<int> st;//用栈存放访问的节点

void tarjan(int u,int pa,G& graph){
  dfn[u] = time++;
  low[u] = dfn[u];//初始化为自身
  st.push(u);

  for(int v : graph.adj[u]){
    //遍历邻接表，寻找u回边 和 u子树节点回边，确定low[u]
    if(v == pa)continue;//跳过树边

    if(dfn[v] == 0){
      //未访问,v是子节点,(u,v)是树边,递归到v
      tarjan(v,u,graph);
      low[u] = min(low[u],low[v]);//子树递归返回，将子树回边祖先传给u
    }else{
      //已经访问，v是祖先,(u,v)是回边
      low[u] = min(low[u],dfn[v]);//这里low可能已经更新了，要比较后赋值
    }

    //判断桥，更新EBCC
    //判桥应当在循环内完成，因为每个循环分支是一个dfs分支，判定应当在分支内完成，否则节点栈中会存在多个分支的节点
    // 写在条件分支内是等价的
    if(low[v] > dfn[u]){
      //v不能通过回边到达u及其祖先 (u,v)是桥
      //弹出 v以及在其之后入栈的所有节点 为一个ebcc
      vector<int> ebcc;
      while(st.top() != v){
        ebcc.push_back(st.top());
        st.pop();
      }
      //v也属于此分量
      ebcc.push_back(st.top());
      st.pop();
      ebccs.push_back(ebcc);
    }
  }

  if(dfn[u] == 1){
    //起始节点，将栈中剩余ebcc处理
    vector<int> ebcc;
    while(!st.empty()){
      ebcc.push_back(st.top());
      st.pop();
    }
    ebccs.push_back(ebcc);
  }
}


void tarjan(int u,int pa,G& graph){
  dfn[u] = time++;
  low[u] = dfn[u];//初始化为自身
  st.push(u);

  for(int v : graph.adj[u]){
    //遍历邻接表，寻找u回边 和 u子树节点回边，确定low[u]
    if(v == pa)continue;//跳过树边

    if(dfn[v] == 0){
      //未访问,v是子节点,(u,v)是树边,递归到v
      tarjan(v,u,graph);
      low[u] = min(low[u],low[v]);//子树递归返回，将子树回边祖先传给u
    }else{
      //已经访问，v是祖先,(u,v)是回边
      low[u] = min(low[u],dfn[v]);//这里low可能已经更新了，要比较后赋值
    }
  }

  if(dfn[u] == low[u]){
    //u 是 ebcc 的根节点，从 u 开始的剩余栈顶是一组ebcc
    vector<int> ebcc;
    while(true){
      int x = st.top();
      ebcc.push_back(x);
      st.pop();
      if(x == u)break;
    }
    ebccs.push_back(ebcc);
  }
}
```
#### 点双连通分量
一个没有割点的极大连通点集我们称之为一个点双连通分量

分量中任意两点存在没有交集的路径相互到达

在仙人掌图：每条边最多属于一个简单环 中，点双连通分量为环或者连接的双点

```cpp
void tarjan(int u,int pa,G& graph){
  dfn[u] = time++;
  low[u] = dfn[u];//初始化为自身
  st.push(u);

  for(int v : graph.adj[u]){
    //遍历邻接表，确定不通过父节点能到达的最小时间戳
    if(v == pa)continue;//跳过树边

    if(dfn[v] == 0){
      //未访问,v是子节点,(u,v)是树边,递归到v
      tarjan(v,u,graph);
      low[u] = min(low[u],low[v]);//子树递归返回，将子树回边祖先传给u
    }else{
      //已经访问，v是祖先,(u,v)是回边
      low[u] = min(low[u],dfn[v]);//这里low可能已经更新了，要比较后赋值
    }
  }

  //对于所有邻接节点，如果存在v必须通过父节点u才能到达更早节点，则u是一个割点
}
```