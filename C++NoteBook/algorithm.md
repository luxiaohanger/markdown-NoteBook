# Algorithm
## 二分查找
### binary_search
```cpp
// 使用默认 < 比较
bool binary_search(RandomAccessIterator first, RandomAccessIterator last,const T& value);

// 使用自定义比较器
bool binary_search(RandomAccessIterator first, RandomAccessIterator last,const T& value, Compare comp);
```
其他扩展
```cpp
//以下三个函数都可以添加自定义cmp函数作为第四个参数

//返回 第一个 >= value 的元素位置（迭代器）。如果所有元素都比 value 小，返回 last。
RandomAccessIterator lower_bound(RandomAccessIterator first,RandomAccessIterator last,const T& value);
//返回 第一个 > value 的元素位置（迭代器）。如果所有元素都 <= value，返回 last。
RandomAccessIterator upper_bound(RandomAccessIterator first,RandomAccessIterator last,const T& value);
//返回一对迭代器 (lower_bound, upper_bound)，表示所有等于 value 的范围。
pair<RandomAccessIterator, RandomAccessIterator>
equal_range(RandomAccessIterator first,RandomAccessIterator last,const T& value);
```



## 排序
```cpp
//参数为 随机访问迭代器 .begin()/.end()，不能是 list(双向迭代器)
//范围左闭右开 [first,last)
//无cmp函数，默认升序
void sort (RandomAccessIterator first, RandomAccessIterator last);
```
```cpp
//自定义 cmp 函数，一般用于对自定义数据类型排序
void sort (RandomAccessIterator first, RandomAccessIterator last, Compare comp);
//cmp(a, b) 返回 true → 表示 a 应该排在 b 前面
struct Person {
    std::string name;
    int age;
};

bool cmp(const Person &a, const Person &b) {
    return a.age < b.age; // 年龄小的排前面
}
```
**把相关数据封装在一起，用某个数据对 结构体vector 排序,可以保持卫星数据的相关性，便于比较或者反向访问**  
反向访问：由 a 成员检索 b --> 由 b 成员检索 a;