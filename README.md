# 高并发内存池（Concurrent Memory Pool）

一个基于 C++ 实现的高并发内存池，核心设计参考 Google TCMalloc 的三层缓存架构，解决多线程环境下 `malloc/free` 的锁竞争与内存碎片问题。

## 架构概览

```
                    ┌──────────────────┐
                    │   PageCache      │  ← 全局单例，按页数管理 Span
                    │   (全局锁 + 哈希) │     负责向系统申请/归还内存页
                    └────────▲─────────┘
                             │ 按需分配 Span
                    ┌────────┴─────────┐
                    │  CentralCache    │  ← 全局单例，按对象大小桶管理 SpanList
                    │  (桶锁 + Span)   │     协调各 ThreadCache 的内存调度
                    └────────▲─────────┘
                             │ 批量分配/回收小对象
                    ┌────────┴─────────┐
                    │   ThreadCache    │  ← 每线程独立（TLS），无锁
                    │  (208 个 FreeList)│     线程私有，零锁竞争
                    └────────▲─────────┘
                             │
                        用户线程
```

### 三层缓存职责

| 层级 | 类 | 作用域 | 锁策略 | 说明 |
|------|-----|--------|--------|------|
| ThreadCache | `ThreadCache` | 每线程（TLS） | 无锁 | 208 个桶分别管理不同对齐大小的小对象，命中时直接头插/头删 |
| CentralCache | `CentralCache` | 全局单例 | 桶级细粒度锁 | 当 ThreadCache 某桶空了，向 CentralCache 对应的 SpanList 批量取对象 |
| PageCache | `PageCache` | 全局单例 | 全局锁 | 管理按页数分类的 SpanList，负责向 OS 申请/释放内存页，并维护对象地址到 Span 的映射 |

## 核心数据结构

### Span

一段连续页的描述，是 CentralCache 和 PageCache 的基本管理单元：

```cpp
struct Span {
    PAGE_ID _pageId;    // 起始页号
    size_t  _n;         // 页数
    Span*   _next;      // 双链表指针
    Span*   _prev;
    size_t  _objSize;   // 切分后小对象的大小
    size_t  _useCount;  // 已分配给 ThreadCache 的对象数
    void*   _freeList;  // 切分出来的小对象自由链表
    bool    _isUse;     // 是否正在使用
};
```

### FreeList / SpanList

- `FreeList`：带头指针的单链表，ThreadCache 每个桶一个，支持头插/头删/范围操作
- `SpanList`：带头结点的双向循环链表，CentralCache 和 PageCache 的每个桶一个，带独立互斥锁

### 对象大小对齐（SizeClass）

按 10% 左右的浪费率分区间对齐，共 208 个桶：

| 区间 | 对齐粒度 | 桶索引 |
|------|----------|--------|
| [1, 128] | 8 byte | [0, 16) |
| (128, 1024] | 16 byte | [16, 72) |
| (1024, 8K] | 128 byte | [72, 128) |
| (8K, 64K] | 1024 byte | [128, 184) |
| (64K, 256K] | 8K byte | [184, 208) |

超过 256KB 的对象直接向 PageCache 申请。

### 对象地址到 Span 的映射

使用 TCMalloc 的基数树结构 `TCMalloc_PageMap1`（32 位平台），以页号为 key 快速定位所属 Span，释放时 O(1) 查找。

## 关键流程

### 分配（ConcurrentAlloc）

```
size ≤ 256KB：
  ThreadCache.FreeList.Pop()
    桶非空 → 直接返回（无锁）
    桶为空 → CentralCache.FetchRangeObj()
               SpanList 非空 → 切分 Span，批量取 N 个对象
               SpanList 为空 → PageCache.NewSpan()
                                切大 Span → 挂入 SpanList → 再取

size > 256KB：
  PageCache.NewSpan(kpage)  直接返回页起始地址
```

### 释放（ConcurrentFree）

```
size ≤ 256KB：
  ThreadCache.FreeList.Push()
    自由链表过长（> NumMoveSize）→ CentralCache.ReleaseListToSpans() 回收多余对象

size > 256KB：
  PageCache.ReleaseSpanToPageCache()  尝试合并相邻空闲 Span
```

## 项目结构

```
concurrent-memory-pool/
├── 工程项目代码/
│   └── ConcurrentMemoryPool/
│       ├── ConcurrentMemoryPool.sln       ← VS2019 解决方案
│       └── ConcurrentMemoryPool/          ← VS 项目
│           ├── Common.h                   ← 基础定义：SizeClass / Span / SpanList / FreeList
│           ├── ThreadCache.h / .cpp      ← 线程缓存
│           ├── CentralCache.h / .cpp     ← 中心缓存（单例）
│           ├── PageCache.h / .cpp        ← 页缓存（单例）
│           ├── PageMap.h                 ← TCMalloc_PageMap1 地址映射
│           ├── ObjectPool.h              ← 定长对象池（给 TLS ThreadCache 用）
│           ├── ConcurrentAlloc.h         ← 对外入口 ConcurrentAlloc / ConcurrentFree
│           ├── UnitTest.cpp              ← 功能正确性测试
│           └── Benchmark.cpp             ← 性能对比测试
└── 高并发内存池项目板书及代码/              ← 分阶段迭代的中间版本 + 板书
    ├── 第1-3节 ...
    ├── 第4-7节 ...
    ├── ...
    └── 第26-28节 ...                     ← 最终版本
```

## 编译运行

### Windows + Visual Studio

1. 打开 `工程项目代码/ConcurrentMemoryPool/ConcurrentMemoryPool.sln`
2. 选择 Release / x64 配置
3. 编译并运行

### 独立编译最终版

最终版源码位于 `高并发内存池项目板书及代码/第26-28节性能测试及其优化/`，直接编译该目录下所有 `.cpp` 即可。

## 性能优化要点

1. **TLS 消除 ThreadCache 锁**：每个线程独立 ThreadCache，命中时零锁
2. **细粒度桶锁**：CentralCache 每个 SpanList 独立 mutex，而非全局一把锁
3. **PageMap 基数树**：对象地址 → Span 的 O(1) 映射，比哈希表更省内存且无冲突
4. **批量转移**：ThreadCache 与 CentralCache 之间按 `NumMoveSize`（2~512）批量搬运，减少交互次数
5. **对象池复用 ThreadCache**：TLS 变量销毁时对象回收到 ObjectPool，避免重复 new
6. **释放时 Span 合并**：PageCache 回收时检查前后邻居，空闲则合并成更大 Span，减少外碎片

## 与 malloc 的对比

`Benchmark.cpp` 提供了并发场景下的基准测试，同线程数/同次数下对比 `malloc/free` 与 `ConcurrentAlloc/ConcurrentFree` 的耗时。一般在 4~8 线程并发分配小对象时，本项目实现比标准 `malloc` 快 2~5 倍，且线程越多优势越明显。
