# 9.2 互斥锁：sync.Mutex

在上一节中，我们看到了如何使用通道在多个goroutine之间安全地共享变量。这种方法虽然有效，但本质上是通过顺序化访问来实现并发安全。当变量需要被多个goroutine频繁访问时，通道可能会成为性能瓶颈。对于这种情况，更直接的方法是使用**互斥锁**（mutual exclusion lock）来保证同一时间只有一个goroutine能访问共享变量。

Go标准库中的`sync`包提供了`Mutex`类型来支持这种机制。`Mutex`有两个方法：
- `Lock()`：获取锁。如果锁已被其他goroutine持有，调用者会被阻塞直到锁可用
- `Unlock()`：释放锁

下面我们用互斥锁来重写银行账户的例子：

```go
import "sync"

var (
    mu      sync.Mutex // 保护balance
    balance int
)

func Deposit(amount int) {
    mu.Lock()
    balance = balance + amount
    mu.Unlock()
}

func Balance() int {
    mu.Lock()
    b := balance
    mu.Unlock()
    return b
}
```

在这个版本中，`mu`互斥锁保护了`balance`变量。每次访问`balance`时（无论是读还是写），都必须先获取锁，操作完成后立即释放锁。这样就能确保同一时间只有一个goroutine能访问`balance`。

## 互斥锁的使用惯例

使用互斥锁时，有几个重要惯例需要遵守：

1. **锁保护的变量声明应紧跟在锁声明之后**。这种代码组织方式使得读者能立即看出哪些变量受锁保护。

2. **在获取锁后应立即使用defer释放锁**。特别是在有多个return分支或可能panic的情况下，这样可以确保锁总是会被释放：

```go
func Balance() int {
    mu.Lock()
    defer mu.Unlock()
    return balance
}
```

3. **临界区应尽可能短**。在持有锁期间，应只进行必要的操作，避免执行耗时操作或可能阻塞的操作（如I/O操作）。

## 更复杂的例子：缓存实现

让我们看一个更复杂的例子，实现一个并发安全的缓存：

```go
var cache = struct {
    sync.Mutex
    mapping map[string]string
}{
    mapping: make(map[string]string),
}

func Lookup(key string) string {
    cache.Lock()
    v := cache.mapping[key]
    cache.Unlock()
    return v
}

func Add(key, value string) {
    cache.Lock()
    cache.mapping[key] = value
    cache.Unlock()
}
```

这个例子展示了一种常见的模式：将互斥锁嵌入到结构体中，与其保护的字段放在一起。这种组织方式使得代码更清晰，也更容易维护。

## 读写互斥锁：sync.RWMutex

在某些场景下，读操作远多于写操作。对于这种情况，使用普通的互斥锁会限制并发性能，因为即使只是读取也需要互斥。Go提供了`sync.RWMutex`类型，它支持两种锁：

- **读锁（RLock/RUnlock）**：多个goroutine可以同时持有读锁
- **写锁（Lock/Unlock）**：与普通互斥锁相同，一次只能有一个goroutine持有

使用RWMutex可以显著提高读多写少场景的性能：

```go
var mu sync.RWMutex
var balance int

func Balance() int {
    mu.RLock() // 读锁
    defer mu.RUnlock()
    return balance
}
```

需要注意的是：
1. 写锁会阻塞所有读锁和写锁
2. 读锁会阻塞写锁，但不会阻塞其他读锁
3. 不能将读锁升级为写锁，否则会导致死锁

## 内存同步

互斥锁不仅提供了对共享变量的互斥访问，还确保了**内存可见性**。现代计算机通常有多个处理器，每个处理器都有自己的本地缓存。为了效率，对内存的写入通常会在每个处理器的缓存中暂存，只在必要时才刷回主内存。这种情况下，一个处理器上的goroutine对变量的修改可能不会立即被其他处理器上的goroutine看到。

互斥锁的Lock和Unlock操作会确保：
- Lock操作会获取最新的变量值（刷新处理器缓存）
- Unlock操作会将修改刷回主内存

类似地，通道的发送和接收操作也会进行内存同步。因此，在不需要互斥的场景下，也可以使用通道来实现内存同步。

## 注意事项

1. **避免锁拷贝**：互斥锁类型不应被拷贝，否则会导致锁失效。如果需要传递锁，应使用指针。

2. **避免死锁**：确保锁总是会被释放，特别是在有多个return分支或可能panic的情况下，使用defer是最安全的做法。

3. **锁粒度**：锁的粒度应适中。过粗的锁（保护太多数据）会限制并发性能；过细的锁（保护太少数据）会增加复杂度并可能导致死锁。

4. **性能考虑**：在高并发场景下，锁可能成为性能瓶颈。这时可以考虑使用更细粒度的锁、读写锁，或者无锁数据结构。

通过合理使用互斥锁，我们可以安全地在多个goroutine之间共享变量，同时保持程序的并发性能。在下一节中，我们将探讨另一种同步机制：条件变量。