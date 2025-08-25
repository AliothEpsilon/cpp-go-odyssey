# `std::vector` 动态扩容机制深度剖析

`std::vector` 作为最常用的 STL 容器，其“动态数组”的特性极大地简化了内存管理。然而，当 `push_back` 或 `insert` 导致容量不足时，`vector` 内部究竟发生了什么？扩容的代价是多少？不同标准库实现（如 libstdc++ 和 libc++）有何异同？本文将深入源码，揭示 `vector` 扩容的底层机制。

## 1. 扩容的触发条件

`vector` 维护着两个关键状态：
*   **`size()`**: 当前存储的元素数量。
*   **`capacity()`**: 当前分配的存储空间能容纳的元素总数。

当 `size() == capacity()` 时，任何增加元素的操作（如 `push_back`, `insert`, `resize` 到更大尺寸）都将触发扩容。

## 2. 扩容的核心步骤

一次典型的扩容过程包含以下步骤：

1.  **计算新容量 (`new_capacity`):** 这是最关键的一步，决定了扩容策略。
2.  **分配新内存块:** 使用 `allocator_traits::allocate` 申请足够容纳 `new_capacity` 个元素的内存。
3.  **移动/复制元素:** 将旧内存块中的所有元素**移动**（如果元素类型支持移动构造）或**复制**到新内存块中。这是最耗时的操作，时间复杂度为 O(n)。
4.  **析构并释放旧内存:** 在旧内存块上调用所有元素的析构函数，然后使用 `allocator_traits::deallocate` 释放内存。
5.  **更新内部指针:** 将 `vector` 的内部指针（指向数据的开始、结束和容量末尾）指向新的内存块。

## 3. 扩容策略：几何级数增长

为了平衡内存使用效率和操作性能，`vector` 采用**几何级数增长**策略。最常见的是**倍增策略**（增长因子约为 2），但具体实现有所不同。

### 3.1 典型实现分析

*   **libstdc++ (GNU):**
    *   **增长因子:** 通常为 **2**。
    *   **源码线索 (`bits/stl_vector.h`):** `void _M_realloc_insert(iterator __position, _Args&&... __args)` 方法中，`_M_check_len` 函数负责计算新长度。其逻辑通常是 `max(_M_impl._M_finish - _M_impl._M_start, __n) + __n` 或类似变体，旨在确保新容量至少是当前大小加上所需增量，且倾向于翻倍。
    *   **特点:** 简单直接，摊还时间复杂度优秀。

*   **libc++ (LLVM):**
    *   **增长因子:** 采用 **1.5** 倍策略（更精确地说，是 `size + size / 2`）。
    *   **源码线索 (`__container/vector.h`):** `__recommend` 函数（或类似名称）是关键。它计算推荐的容量，逻辑通常是 `max(_capacity + _capacity / 2, __new_size)`。
    *   **动机:** 理论研究表明，增长因子为黄金比例 (≈1.618) 时，内存碎片化程度可能更低，且旧内存块在后续分配中更有可能被重新利用。1.5 是一个接近的实用选择，旨在减少内存浪费和碎片。

### 3.2 为什么是几何增长？

*   **摊还分析 (Amortized Analysis):** 虽然单次 `push_back` 可能触发 O(n) 的扩容，但通过几何增长，**n 次 `push_back` 操作的总时间复杂度是 O(n)**。这意味着 `push_back` 的**摊还时间复杂度为 O(1)**。
*   **避免频繁分配:** 线性增长（每次只增加固定大小）会导致分配操作过于频繁，性能急剧下降。

## 4. 源码片段示例 (概念性)

以下是一个简化版的 `push_back` 触发扩容的逻辑：

```cpp
template<typename T, typename Allocator = std::allocator<T>>
class vector {
    T* start_;
    T* finish_;
    T* end_of_storage_;
    Allocator alloc_;

    void push_back(const T& value) {
        if (finish_ == end_of_storage_) { // 容量不足
            size_t old_capacity = capacity();
            size_t new_capacity = grow_capacity(old_capacity); // 计算新容量
            T* new_start = alloc_.allocate(new_capacity);      // 分配新内存

            try {
                // 移动旧元素到新内存 (使用移动语义)
                std::uninitialized_move(start_, finish_, new_start);
            } catch (...) {
                alloc_.deallocate(new_start, new_capacity); // 异常安全：释放新内存
                throw;
            }

            // 析构并释放旧内存
            destroy_elements(start_, finish_); // 调用析构函数
            if (start_) alloc_.deallocate(start_, old_capacity);

            // 更新指针
            start_ = new_start;
            finish_ = new_start + old_capacity; // 旧元素已移动
            end_of_storage_ = new_start + new_capacity;

            // 在新位置构造新元素
            alloc_.construct(finish_, value);
            ++finish_;
        } else {
            // 直接在末尾构造
            alloc_.construct(finish_, value);
            ++finish_;
        }
    }

    size_t grow_capacity(size_t current_capacity) {
        // libstdc++ 风格：倾向于翻倍
        // return current_capacity > 0 ? current_capacity * 2 : 1;

        // libc++ 风格：1.5倍
        return current_capacity > 0 ? current_capacity + current_capacity / 2 : 1;
    }
};
```

## 5. 注意

*   **迭代器、指针、引用失效:** 扩容必然导致**所有指向 `vector` 元素的迭代器、指针和引用全部失效**！这是使用 `vector` 时必须牢记的铁律。
*   **预留空间 (`reserve`):** 如果能预估元素数量，务必在大量 `push_back` 前调用 `reserve(n)`。这可以避免多次不必要的扩容和元素移动，显著提升性能。
*   **`shrink_to_fit` (C++11):** 当 `vector` 包含的元素远少于其容量时，可调用此方法（非强制）请求释放多余内存。
*   **移动语义的重要性:** C++11 引入的移动语义极大地降低了扩容时复制大型对象的开销。确保类型提供高效的移动构造函数。
