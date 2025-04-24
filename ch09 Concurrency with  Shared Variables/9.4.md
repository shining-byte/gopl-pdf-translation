# 9.4 内存同步

在前面的章节中，我们讨论了互斥锁和读写锁如何保护共享变量免受并发访问的影响。然而，并发编程中还有一个更深层次的问题需要考虑——**内存同步**（Memory Synchronization）。这个问题与现代计算机的多层次内存体系结构密切相关。

## 现代计算机的内存模型

现代计算机通常采用多层次的存储结构：
1. **每个CPU核心有自己的缓存**（L1、L2缓存）
2. **共享的最后一级缓存**（LLC）
3. **主内存**

这种架构带来了**内存可见性**问题：一个处理器对变量的修改可能不会立即被其他处理器看到，因为：
- 写入可能暂时停留在处理器缓存中
- 编译器可能对指令进行重排序以优化性能

## Go的内存模型

Go的内存模型规定了：
1. **在一个goroutine中**，读写的执行顺序与代码顺序一致
2. **在多个goroutine之间**，如果没有同步机制，无法保证一个goroutine能看到另一个goroutine的写入

考虑以下示例：

```go
var x, y int

// goroutine 1
go func() {
    x = 1
    fmt.Print(y)
}()

// goroutine 2
go func() {
    y = 1
    fmt.Print(x)
}()
```

这段代码可能输出：
- `0 0`（两个goroutine都看不到对方的写入）
- `0 1`或`1 0`（一个goroutine看到了另一个的写入）
- `1 1`（两个goroutine都看到了对方的写入）

## 同步原语的内存效应

Go的同步原语不仅提供互斥功能，还确保内存可见性：

1. **互斥锁（sync.Mutex）**：
   - `Unlock()`保证之前的写入对后续`Lock()`可见
   - `Lock()`保证获取最新的内存状态

2. **通道（channel）**：
   - 发送操作保证之前的写入对接收方可见
   - 接收操作保证能看到发送前的所有写入

3. **原子操作（sync/atomic）**：
   - 提供原子性的同时，也保证内存顺序

## 初始化安全

`sync.Once`提供了一种安全初始化共享变量的方式：

```go
var (
    instance *SomeType
    once     sync.Once
)

func GetInstance() *SomeType {
    once.Do(func() {
        instance = &SomeType{}
    })
    return instance
}
```

`once.Do`保证：
1. 初始化函数只执行一次
2. 其他goroutine能看到完整的初始化结果

## 缓存一致性

考虑以下缓存实现：

```go
var (
    mu     sync.Mutex
    cache  map[string]string
    inited bool
)

func Get(key string) string {
    mu.Lock()
    defer mu.Unlock()
    
    if !inited {
        cache = make(map[string]string)
        inited = true
    }
    
    return cache[key]
}
```

这里的互斥锁不仅保护并发访问，还确保：
1. `cache`的初始化对所有goroutine可见
2. `inited`标志的检查与设置是原子的

## 最佳实践

1. **使用同步原语共享数据**：不要依赖无保护的共享变量
2. **避免数据竞争**：使用`-race`标志检测竞争条件
3. **最小化共享**：尽可能减少需要同步的数据量
4. **明确同步点**：清晰地标识出同步发生的位置

## 总结

内存同步是并发编程中最微妙的问题之一。Go通过同步原语提供了清晰的内存模型，但开发者仍需理解：
- 多处理器环境下的内存可见性问题
- 同步操作的内存效应
- 如何正确使用同步机制

在下一节中，我们将探讨另一种并发模式：惰性初始化。
