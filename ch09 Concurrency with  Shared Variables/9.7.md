# 9.7 示例：并发非阻塞缓存  

本节介绍一个**并发非阻塞缓存**的实现，它解决了函数记忆化（memoization）问题——即缓存函数的结果以避免重复计算。该设计适用于并发环境，并支持多个goroutine同时访问缓存，而不会导致阻塞或数据竞争。  

## 问题描述  
假设有一个计算开销较大的函数 `f(key)`，我们希望缓存其结果，使得对相同 `key` 的后续调用可以直接返回缓存值，而无需重新计算。  

## 初始方案：互斥锁保护  
最简单的实现是用一个 `map` 存储缓存结果，并用 `sync.Mutex` 保护并发访问：  

```go
type Memo struct {
    f     Func
    mu    sync.Mutex
    cache map[string]result
}

type Func func(key string) (interface{}, error)

type result struct {
    value interface{}
    err   error
}

func New(f Func) *Memo {
    return &Memo{f: f, cache: make(map[string]result)}
}

func (memo *Memo) Get(key string) (interface{}, error) {
    memo.mu.Lock()
    res, ok := memo.cache[key]
    memo.mu.Unlock()
    if !ok {
        res.value, res.err = memo.f(key) // 计算
        memo.mu.Lock()
        memo.cache[key] = res            // 存储
        memo.mu.Unlock()
    }
    return res.value, res.err
}
```  

**问题**：  
- 如果多个goroutine同时请求同一个未缓存的 `key`，它们会**串行等待** `f(key)` 计算完成，导致性能下降。  

## 改进方案：避免重复计算  
我们可以让每个 `key` 对应一个 `entry`，其中包含计算结果和一个 `sync.Once`，确保 `f(key)` 只计算一次：  

```go
type entry struct {
    res   result
    ready chan struct{} // 用于广播结果就绪
}

func (memo *Memo) Get(key string) (interface{}, error) {
    memo.mu.Lock()
    e := memo.cache[key]
    if e == nil {
        e = &entry{ready: make(chan struct{})}
        memo.cache[key] = e
        memo.mu.Unlock()
        e.res.value, e.res.err = memo.f(key) // 计算
        close(e.ready)                       // 广播结果就绪
    } else {
        memo.mu.Unlock()
        <-e.ready // 等待计算完成
    }
    return e.res.value, e.res.err
}
```  

**优化点**：  
- 首次请求 `key` 时，创建 `entry` 并启动计算，其他goroutine会等待 `e.ready` 关闭。  
- 计算完成后，关闭 `e.ready`，所有等待的goroutine会立即返回缓存值。  

## 最终方案：并发安全的缓存  
进一步封装 `Memo`，使其完全并发安全：  

```go
type Memo struct {
    f     Func
    mu    sync.Mutex
    cache map[string]*entry
}

func (memo *Memo) Get(key string) (interface{}, error) {
    memo.mu.Lock()
    e := memo.cache[key]
    if e == nil {
        e = &entry{ready: make(chan struct{})}
        memo.cache[key] = e
        memo.mu.Unlock()
        e.res.value, e.res.err = memo.f(key)
        close(e.ready)
    } else {
        memo.mu.Unlock()
        <-e.ready
    }
    return e.res.value, e.res.err
}
```  

**特点**：  
- **非阻塞**：多个goroutine可以并发读取已缓存的结果。  
- **避免重复计算**：每个 `key` 的计算仅执行一次。  
- **低竞争**：锁的粒度细化到单个 `entry`，而非整个缓存。  

## 适用场景  
这种模式适用于：  
- 高并发读取、低频更新的缓存（如DNS解析、API响应缓存）。  
- 计算成本高、结果可复用的函数。  

通过合理使用 `sync.Mutex` 和通道，我们实现了高效且线程安全的并发缓存。