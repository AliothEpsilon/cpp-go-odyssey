# Hash Tables & Sets in C++  

---

## 目录
1. [C++ 中的核心容器](#c-中的核心容器)
2. [基本操作速查表](#基本操作速查表)
3. [时间复杂度分析](#时间复杂度分析)
4. [典型应用场景 + 代码模板](#典型应用场景--代码模板)
5. [自定义键（Custom Key）如何哈希？](#自定义键custom-key如何哈希)
6. [性能优化技巧](#性能优化技巧)
7. [常见陷阱与注意事项（C++ 专属）](#常见陷阱与注意事项c-专属)
8. [与 `map` / `set` 的对比](#与-map--set-的对比)

---

## C++ 中的核心容器

C++ 标准库在 `<unordered_map>` 和 `<unordered_set>` 头文件中提供了基于**哈希表**实现的容器：

| 容器 | 用途 | 头文件 |
|------|------|--------|
| `std::unordered_map<Key, T>` | 存储键值对，支持 O(1) 查找 | `<unordered_map>` |
| `std::unordered_set<Key>` | 存储唯一键，用于去重和存在性判断 | `<unordered_set>` |

> 底层实现：**开散列（拉链法）**，每个桶是一个链表或动态数组（具体由编译器实现）。

---

## 基本操作速查表

```cpp
#include <unordered_map>
#include <unordered_set>
#include <iostream>
using namespace std;

// ===== unordered_map 示例 =====
unordered_map<int, string> mp;
mp[1] = "Alice";                    // 插入或更新
mp.insert({2, "Bob"});              // 插入（若已存在则不插入）
mp.emplace(3, "Charlie");           // 原地构造，更高效

if (mp.find(1) != mp.end()) {       // 查找是否存在
    cout << mp[1] << endl;          // 访问值（若 key 不存在会自动插入默认值！）
}

mp.erase(2);                        // 删除 key=2
mp.clear();                         // 清空

// 遍历
for (const auto& [k, v] : mp) {     // C++17 结构化绑定
    cout << k << ": " << v << endl;
}
```

```cpp
// ===== unordered_set 示例 =====
unordered_set<int> st;
st.insert(1);
st.insert(2);
st.insert(1); // 重复插入无效

if (st.count(1)) {                  // 判断是否存在（返回 0 或 1）
    cout << "Found!" << endl;
}

st.erase(2);
```

---

## 时间复杂度分析

| 操作 | 平均情况 | 最坏情况 | 说明 |
|------|---------|----------|------|
| `insert`, `emplace` | O(1) | O(n) | 所有元素哈希冲突 |
| `find`, `count` | O(1) | O(n) | 同上 |
| `erase(key)` | O(1) | O(n) | 查找 + 删除 |
| `erase(iterator)` | O(1) | O(1) | 给定迭代器时是常数时间 |
| 空间 | O(n) | O(n) | n 为元素个数 |

> **注意**：最坏情况极少发生，STL 实现会自动**rehash**（扩容）以控制负载。

---

## 典型应用场景 + 代码模板

### 1. 两数之和（Two Sum）
```cpp
vector<int> twoSum(vector<int>& nums, int target) {
    unordered_map<int, int> mp; // 值 -> 索引
    for (int i = 0; i < nums.size(); ++i) {
        int complement = target - nums[i];
        if (mp.count(complement)) {
            return {mp[complement], i};
        }
        mp[nums[i]] = i;
    }
    return {};
}
```

### 2. 字符频次统计
```cpp
unordered_map<char, int> freq;
for (char c : str) {
    freq[c]++;
}
// 找最大频次字符
auto it = max_element(freq.begin(), freq.end(),
    [](auto& a, auto& b) { return a.second < b.second; });
```

### 3. 数组去重
```cpp
unordered_set<int> unique_nums(nums.begin(), nums.end());
// 转回 vector
vector<int> res(unique_nums.begin(), unique_nums.end());
```

### 4. 判断是否存在重复元素
```cpp
bool hasDuplicate(vector<int>& nums) {
    unordered_set<int> seen;
    for (int x : nums) {
        if (seen.count(x)) return true;
        seen.insert(x);
    }
    return false;
}
```

---

## 自定义键（Custom Key）如何哈希？

当你想用 `pair<int, int>`、`vector<int>` 或自定义结构体作为 key 时，**默认没有哈希函数**，需要手动提供。

### 方法 1：提供 `std::hash` 特化（推荐）

```cpp
struct Point {
    int x, y;
    bool operator==(const Point& other) const {
        return x == other.x && y == other.y;
    }
};

// 自定义哈希函数
struct HashPoint {
    size_t operator()(const Point& p) const {
        return hash<int>()(p.x) ^ (hash<int>()(p.y) << 1);
    }
};

// 使用
unordered_map<Point, int, HashPoint> pointMap;
pointMap[{1, 2}] = 100;
```

> **技巧**：用 `^ (shift)` 避免 `x=1,y=2` 和 `x=2,y=1` 哈希冲突。

### 方法 2：用 `std::pair`（内置支持）
```cpp
unordered_map<pair<int, int>, int> mp; // 编译错误！
// 正确做法：仍需自定义哈希，或用其他方式
```

> C++ 标准库**不为 `pair` 提供默认哈希**！必须自定义。

### 方法 3：转换为字符串或整数（简单场景）
```cpp
// 对于 0 ≤ x,y < 1000
long long key = x * 10000LL + y;
unordered_map<long long, int> mp;
mp[key] = value;
```

---

## 性能优化技巧

| 技巧 | 说明 |
|------|------|
| **预分配空间** | `mp.reserve(N)` 避免多次 rehash |
| **使用 `emplace` 而非 `insert`** | 减少拷贝，原地构造 |
| **避免 `mp[key]` 无意创建默认值** | 用 `find()` 或 `count()` 判断存在性 |
| **选择合适的 bucket 数** | `mp.bucket_count()` 可调优（一般不用） |
| **关闭同步（竞赛用）** | `ios::sync_with_stdio(false);` 加速 I/O |

```cpp
// 优化示例
unordered_map<int, string> mp;
mp.reserve(10000); // 预分配空间
mp.max_load_factor(0.5); // 更早扩容，减少冲突

for (...) {
    auto it = mp.find(key); // 比 mp[key] 更安全
    if (it != mp.end()) { ... }
}
```

---

## 常见陷阱与注意事项（C++ 专属）

| 陷阱 | 解决方案 |
|------|----------|
| `mp[key]` 会**默认插入** `T{}` | 用 `find()` 或 `count()` 判断存在性 |
| 迭代器失效：`insert` 可能触发 rehash | 避免在遍历时插入大量元素 |
| `unordered_map` **不保证顺序** | 需要有序用 `map`（红黑树） |
| 自定义类型必须提供 `==` 和哈希函数 | 否则编译报错 |
| 哈希碰撞可能导致 TLE | 竞赛中可加随机盐或换 `map` |

> **竞赛警告**：某些 OJ 可能被哈希碰撞攻击，可用 `gp_hash_table`（PBDS）替代。

---

## 与 `map` / `set` 的对比

| 特性 | `unordered_map/set` | `map/set` |
|------|---------------------|-----------|
| 底层结构 | 哈希表 | 红黑树（平衡二叉搜索树） |
| 查找/插入/删除 | 平均 O(1)，最坏 O(n) | O(log n) |
| 是否有序 | 否 | 是（按键排序） |
| 内存开销 | 较低 | 较高（每个节点有指针） |
| 哈希函数要求 | 需要可哈希类型 | 只需支持 `<` 比较 |
| 推荐使用场景 | 快速查找、计数、去重 | 需要有序遍历、范围查询 |

> **选择建议**：
> - 要快？用 `unordered_*`
> - 要有序？用 `map/set`

---

## 总结（C++ 版）

| 要点 | 建议 |
|------|------|
| **首选容器** | `unordered_map`, `unordered_set` |
| **插入优先** | `emplace` > `insert` > `operator[]` |
| **查找安全** | `find()` 或 `count()`，避免 `[]` 意外插入 |
| **自定义 key** | 必须提供 `==` 和哈希函数对象 |
| **性能调优** | `reserve()`, `max_load_factor()` |
| **避免陷阱** | 迭代器失效、默认插入、无序性 |
