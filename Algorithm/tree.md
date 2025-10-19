# 树
## 二叉树
### 树高 （从上向下递归深度） 
二叉树的树高复杂度是 `O(logn) --> O(n)` 之间，取决于是否接近完美二叉树，因此不能默认树高是对数级别，极端情况下退化为链时就是 `O(n)`
### 树的重心
可以证明树上一定存在节点使得删除该节点形成的所有新树的节点数都不大于原来节点总数的一半，这个点称之为二叉树的重心
#### 重心的寻找
利用 `后序遍历可以记录子树 节点数/权值和` 的特点，
先记录每个节点作为根节点时的节点总数，再使用任意顺序遍历树即可
```cpp
//后序遍历记录子树 节点数/权值和
int countNode(const vector<node> &tree, vector<int> &nNode, int root) {
    if (root == 0)return 0;
    int nleft = countNode(tree, nNode, tree[root].left);
    int nright = countNode(tree, nNode, tree[root].right);
    int ans = nleft + nright + 1;
    nNode[root] = ans;
    return ans;
}

int findG(const vector<node> &tree, const vector<int> &nNode, int root, int size) {
    int nleft = nNode[tree[root].left];
    int nright = nNode[tree[root].right];
    int nparent = size - nNode[root];
    if (nleft <= size / 2 && nright <= size / 2 && nparent <= size / 2) {
        return root;
    }
    //由于我们从根节点向下遍历，因此重心不会在父树中
    //重心在子树中节点更多的一边
    if (nleft > size / 2)return findG(tree, nNode, tree[root].left, size);
    return findG(tree, nNode, tree[root].right, size);
}
```