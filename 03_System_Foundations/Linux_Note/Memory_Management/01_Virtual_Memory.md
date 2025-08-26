# 01_Virtual_Memory.md  

| 核心概念 | 说明 |
|--------|------|
| **虚拟地址空间** | 每个进程独立，从 0 到 128TB（x86_64） |
| **分页机制** | 4KB 页，虚拟页 → 物理页帧 |
| **页表** | 4 级结构（PML4 → PT），MMU 查阅 |
| **TLB** | 地址转换缓存，提升性能 |
| **mm_struct** | 内核中进程内存的“总控结构” |
| **vm_area_struct** | 描述一段虚拟内存区域（VMA） |
| **mmap** | 创建内存映射，支持文件映射和匿名映射 |

## 目录
1. [什么是虚拟内存？](#什么是虚拟内存)
2. [为什么需要虚拟内存？](#为什么需要虚拟内存)
3. [虚拟地址 vs 物理地址](#虚拟地址-vs-物理地址)
4. [分页机制（Paging）](#分页机制paging)
5. [Linux 进程地址空间布局](#linux-进程地址空间布局)
6. [页表（Page Table）结构（x86_64）](#页表page-table结构x86_64)
7. [MMU 与 TLB](#mmu-与-tlb)
8. [虚拟内存的核心数据结构（内核视角）](#虚拟内存的核心数据结构内核视角)
9. [mmap 与内存映射](#mmap-与内存映射)
10. [查看虚拟内存信息](#查看虚拟内存信息)

---

## 什么是虚拟内存？

**虚拟内存（Virtual Memory）** 是现代操作系统提供的一种**抽象机制**，它为每个进程创建一个独立的、连续的地址空间，使得进程“以为”自己独占整个内存，而实际上：

- 地址是“虚拟”的，需要通过**页表**转换为物理地址
- 内存可以**部分加载**（如只加载用到的代码段）
- 支持**内存映射文件**、**共享内存**、**swap** 等高级功能

> 虚拟内存 = 地址空间抽象 + 分页 + 硬件支持（MMU）

---

## 为什么需要虚拟内存？

| 问题 | 虚拟内存的解决方案 |
|------|------------------|
| **物理内存不足** | 通过 **swap** 将不常用页换出到磁盘 |
| **内存碎片** | 分页机制消除外部碎片（但有内部碎片） |
| **地址冲突** | 每个进程有独立地址空间，互不干扰 |
| **安全隔离** | 用户进程无法直接访问内核内存 |
| **共享内存** | 多个进程可映射同一物理页（如共享库） |
| **内存映射文件** | 将文件直接映射到内存，高效读写 |

---

## 虚拟地址 vs 物理地址

| 类型 | 说明 |
|------|------|
| **虚拟地址（Virtual Address, VA）** | 进程使用的地址，如 `0x7ffc8a3b4f00` |
| **物理地址（Physical Address, PA）** | 内存条上的真实地址，由 MMU 转换得到 |

> 转换过程：**CPU → VA → MMU → Page Table → PA → RAM**

```c
// 进程看到的是虚拟地址
int *p = malloc(4);
*p = 42;  // CPU 使用虚拟地址访问
// 实际上，MMU 自动将其转换为某个物理页帧
```

---

## 分页机制（Paging）

虚拟内存将地址空间划分为**固定大小的页（Page）**，通常是 **4KB**（`4096` 字节）。

- 虚拟内存 → 虚拟页（Virtual Page, VP）
- 物理内存 → 页帧（Page Frame, PF）

### 分页的好处：
- **简化内存管理**：按页分配/回收
- **支持非连续映射**：虚拟连续，物理可以分散
- **便于 swap**：整页换入换出
- **支持共享**：多个虚拟页映射到同一物理页

> 页大小可配置（如 2MB、1GB 大页），但默认是 4KB。

---

## Linux 进程地址空间布局

在 x86_64 架构下，每个用户进程的虚拟地址空间通常如下（低地址 → 高地址）：

```
+-------------------------+ 0x00007FFFFFFFFFFF (128TB)
|       Stack             | ↑ 向下增长
|                         |
|        ...              |
|                         |
|       Heap              | ↓ 向上增长
|                         |
|    Memory Mappings      | mmap 区域（共享库、匿名映射）
|   (e.g., libc.so)       |
|                         |
|  Shared Libraries       |
|                         |
|     Read-Only Data      | .rodata, 代码段
|       (Text)            |
|                         |
+-------------------------+ 0x0000000000000000
|     Invalid (NULL)      | 0x0000000000000000 ~ 0x00000000FFFFFFFF
+-------------------------+ 0x00000000FFFFFFFF
|      Kernel Space       | 仅内核可访问（通过系统调用进入）
+-------------------------+ 0xFFFFFFFFFFFFFFFF
```

> **关键区域**：
> - **Text**：可执行代码
> - **Data/BSS**：全局变量
> - **Heap**：`malloc`、`brk`、`sbrk` 分配
> - **Memory Mappings**：`mmap()` 映射文件或匿名内存
> - **Stack**：局部变量、函数调用栈
> - **Kernel Space**：用户态无法直接访问

---

## 页表（Page Table）结构（x86_64）

x86_64 使用 **4 级页表**（PML4 → PDP → PD → PT），支持 48 位虚拟地址。

```
虚拟地址（48位）：
[ 16位 |  9位  |  9位  |  9位  |  9位  | 12位 ]
   ↑      ↑      ↑      ↑      ↑      ↑
   |      |      |      |      |      └─── 页内偏移（0~4095）
   |      |      |      |      └─────────── PT Index（页表）
   |      |      |      └────────────────── PD Index（页目录）
   |      |      └───────────────────────── PDP Index（页目录指针）
   |      └──────────────────────────────── PML4 Index（页全局目录）
   └─────────────────────────────────────── 未使用（符号扩展）

每一级都是一个 512 项的数组，每项 8 字节（64位），共 4KB
```

### 页表项（PTE）结构（简化）
```c
struct page_table_entry {
    uint64_t present    : 1;   // 是否在内存中
    uint64_t writable   : 1;   // 是否可写
    uint64_t user       : 1;   // 用户态是否可访问
    uint64_t accessed   : 1;   // 是否被访问过
    uint64_t dirty      : 1;   // 是否被修改（用于 swap）
    uint64_t global     : 1;   // TLB 中是否全局有效
    uint64_t pat        : 1;   // 页属性表
    uint64_t reserved   : 3;   // 保留
    uint64_t page_frame : 40;  // 物理页帧号（PFN）
    uint64_t reserved2  : 11;  // 保留
    uint64_t nx         : 1;   // No-Execute（防止代码注入）
} __attribute__((packed));
```

---

## MMU 与 TLB

### MMU（Memory Management Unit）
- 硬件单元，集成在 CPU 中
- 负责将虚拟地址转换为物理地址
- 查阅页表，检查权限（读/写/执行）

### TLB（Translation Lookaside Buffer）
- MMU 内的**高速缓存**，缓存最近使用的页表项
- 避免每次访问内存都查多级页表（否则太慢）
- 类似 CPU Cache，但专用于地址转换

> **TLB Miss**：需访问内存查页表，性能下降  
> **TLB Flush**：页表修改后必须刷新 TLB（如 `invlpg` 指令）

---

## 虚拟内存的核心数据结构（内核视角）

Linux 内核使用以下结构管理虚拟内存：

### 1. `struct mm_struct`
每个进程一个，描述其整个地址空间。

```c
struct mm_struct {
    struct vm_area_struct *mmap;     // 虚拟内存区域链表
    struct rb_root mm_rb;            // 同上，红黑树加速查找
    struct list_head mmlist;         // 所有 mm_struct 的链表
    pgd_t *pgd;                      // 页全局目录（PML4）基地址
    int map_count;                   // VMA 数量
    unsigned long start_code, end_code; // 代码段范围
    unsigned long start_data, end_data; // 数据段
    unsigned long start_brk, brk;      // 堆范围
    unsigned long start_stack;         // 栈起始
    // ...
};
```

### 2. `struct vm_area_struct`（VMA）
描述一段连续的虚拟内存区域（如堆、栈、mmap 区）。

```c
struct vm_area_struct {
    unsigned long vm_start, vm_end;  // 虚拟地址范围
    struct vm_area_struct *vm_next;  // 链表
    pgprot_t vm_page_prot;           // 内存保护权限
    unsigned long vm_flags;          // 标志：VM_READ, VM_WRITE, VM_EXEC, VM_SHARED
    struct file *vm_file;            // 映射的文件（可选）
    unsigned long vm_pgoff;          // 文件偏移（页单位）
    // ...
};
```

> `mm_struct` → 多个 `vm_area_struct` → 每个 VMA 对应一组页表项

---

## mmap 与内存映射

`mmap()` 系统调用将文件或设备映射到进程的虚拟地址空间。

### 常见用途：
- 加载共享库（`.so` 文件）
- 创建匿名映射（作为 `malloc` 的后备）
- 高效读写大文件（避免 `read/write` 系统调用开销）
- 进程间共享内存

```c
// 映射文件
void *addr = mmap(NULL, len, PROT_READ, MAP_PRIVATE, fd, 0);

// 匿名映射（相当于堆扩展）
void *heap = mmap(NULL, 1MB, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0);
```

> `mmap` 创建一个新的 `vm_area_struct`，并设置页表项为“缺页状态”，首次访问时触发 **缺页中断（Page Fault）** 加载数据。

---

## 查看虚拟内存信息

Linux 提供多种方式查看进程的虚拟内存布局：

### 1. `/proc/<pid>/maps`
显示进程的所有内存映射区域。

```bash
cat /proc/1234/maps
# 输出示例：
00400000-00401000 r-xp 00000000 08:01 123456 /bin/bash
7fff8a3b4000-7fff8a3b5000 rw-p 00000000 00:00 0 [stack]
```

字段含义：`地址范围 权限 偏移 主:次 inode 文件名`

### 2. `/proc/<pid>/smaps`
更详细的内存使用统计（包括脏页、RSS、PSS 等）。

### 3. `pmap <pid>`
人性化显示内存映射。

```bash
pmap 1234
```

### 4. `vmstat`, `sar -r`, `free`
查看系统级内存使用情况。

---



