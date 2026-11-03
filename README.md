# 高并发内存池（Concurrent Memory Pool）

仿 Google **TCMalloc** 思路实现的三级缓存内存池：`ThreadCache → CentralCache → PageCache`，
把「频繁申请/释放小对象」的加锁开销降到最低，并解决内存碎片与跨线程回收问题。

## 架构

```
        应用层  ConcurrentAlloc / ConcurrentFree
                          |
                  +-------+--------+
                  |                |
        [ <= 256KB ]        [ > 256KB ]
                  |                |
            ThreadCache        PageCache
          (每线程独享,无锁)    (按页分配,大块直取)
                  |
            CentralCache
          (全局共享,按大小类加锁)
                  |
              PageCache
          (8KB/页,向系统申请,可切分/合并)
```

- **ThreadCache**：TLS 线程本地缓存，每个线程一个，小对象分配全无锁（`_declspec(thread)`）。
- **CentralCache**：按 208 个大小类（`NFREELIST`）管理 span 链表，ThreadCache 批量取货时才加锁。
- **PageCache**：以 8KB 页（`PAGE_SHIFT = 13`）为单位向系统申请内存，负责 span 的切分与相邻合并，缓解外碎片。
- **ObjectPool**：定长对象池，专门给上述管理结构（`Span`、`ThreadCache`）自身分配内存，避免递归依赖。
- **PageMap**：页号 → `Span` 的单层基数树映射，用于 `Free` 时由指针反查所属 span。

## 文件说明

| 文件 | 作用 |
|---|---|
| `Common.h` | 公共定义：`SizeClass` 大小类映射、`FreeList` 自由链表、`Span`、`SpanList` |
| `ConcurrentAlloc.h` | 对外接口 `ConcurrentAlloc` / `ConcurrentFree` |
| `ThreadCache.cpp/.h` | 线程缓存：分配、释放、向中心缓存批量取货 |
| `CentralCache.cpp/.h` | 中心缓存：span 管理、按批下发、回收对象到 span |
| `PageCache.cpp/.h` | 页缓存：`NewSpan` 申请、`ReleaseSpanToPageCache` 合并回收 |
| `PageMap.h` | 页号到 span 的映射（参考 TCMalloc 基数树） |
| `ObjectPool.h` | 定长内存池 |
| `Benchmark.cpp` | 与 `malloc/free` 的性能对比测试 |
| `UnitTest.cpp` | 功能单元测试 |

## 编译运行

用 Visual Studio 打开 `ConcurrentMemoryPool.vcxproj`，选择 **Release | x64** 编译：

- 运行 `UnitTest.cpp` 验证功能正确性
- 运行 `Benchmark.cpp` 查看多线程下的性能对比

关键参数可在 `Common.h` 调整：`MAX_BYTES = 256KB`、`NFREELIST = 208`、`NPAGES = 129`、`PAGE_SHIFT = 13`。

## 说明

- 本项目为学习 TCMalloc 原理的练手实现，注释为中文（源文件使用 **GBK** 编码，Visual Studio 直接打开可正常显示）。
- 核心参考：Google TCMalloc 的三层缓存设计与定长内存池思想。