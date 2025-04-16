# 9.5 惰性初始化：sync.Once

在并发编程中，惰性初始化（Lazy Initialization）是一种常见模式——将资源的初始化推迟到第一次实际使用时进行。这种技术可以避免不必要的资源消耗，但同时也带来了并发安全的挑战。Go语言通过`sync.Once`类型为惰性初始化提供了优雅且高效的解决方案。

## 基本概念

`sync.Once`的核心特性：
- **保证初始化代码只执行一次**，即使在多个goroutine并发调用的情况下
- **自动处理同步问题**，开发者无需手动管理锁
- **内存同步保证**，确保所有goroutine都能看到完整的初始化结果

## 典型使用模式

```go
var (
    instance *ExpensiveObject  // 需要延迟初始化的对象
    once     sync.Once         // 控制初始化的Once实例
)

func GetInstance() *ExpensiveObject {
    once.Do(func() {
        instance = &ExpensiveObject{
            // 复杂的初始化逻辑
            config: loadConfig(),
            conn:   establishConnection(),
        }
    })
    return instance
}
```

## 工作原理

1. **原子性计数器**：内部使用原子操作记录初始化状态
2. **双重检查锁定**：高效处理常见路径（已初始化情况）
3. **内存屏障**：确保初始化完成后，所有处理器都能看到最新状态

## 优势对比

相比手动实现的惰性初始化：

| 特性               | sync.Once | 手动实现（带锁） |
|--------------------|-----------|----------------|
| 并发安全           | ✅         | ✅（需正确实现） |
| 初始化仅一次       | ✅         | ✅（需正确实现） |
| 无竞争时性能       | ⚡️ 极快    | 🏃 较快        |
| 有竞争时性能       | ⚡️ 稳定    | 🐢 可能阻塞     |
| 内存可见性保证     | ✅         | ❌（容易遗漏）   |
| 代码复杂度         | 简单      | 复杂           |

## 使用场景

1. **单例模式实现**
   ```go
   var (
       logger *log.Logger
       logOnce sync.Once
   )
   
   func GetLogger() *log.Logger {
       logOnce.Do(func() {
           logger = log.New(os.Stderr, "", log.LstdFlags)
       })
       return logger
   }
   ```

2. **配置文件加载**
   ```go
   var (
       appConfig Config
       configOnce sync.Once
   )
   
   func GetConfig() Config {
       configOnce.Do(func() {
           appConfig = loadConfigFromFile("config.json")
       })
       return appConfig
   }
   ```

3. **数据库连接池初始化**
   ```go
   var (
       dbPool *sql.DB
       dbOnce sync.Once
   )
   
   func GetDB() *sql.DB {
       dbOnce.Do(func() {
           var err error
           dbPool, err = sql.Open("mysql", "user:password@/dbname")
           if err != nil {
               log.Fatal(err)
           }
       })
       return dbPool
   }
   ```

## 高级用法

1. **错误处理**
   ```go
   var (
       cache map[string]string
       cacheErr error
       cacheOnce sync.Once
   )
   
   func GetCache() (map[string]string, error) {
       cacheOnce.Do(func() {
           cache, cacheErr = initializeCache()
       })
       return cache, cacheErr
   }
   ```

2. **重新初始化（通过新建Once实例）**
   ```go
   type Service struct {
       initOnce sync.Once
       conn net.Conn
   }
   
   func (s *Service) Reset() {
       s.initOnce = sync.Once{}  // 允许重新初始化
   }
   
   func (s *Service) ensureConn() {
       s.initOnce.Do(func() {
           s.conn = connectToBackend()
       })
   }
   ```

## 性能考量

1. **无竞争情况**：仅需一个原子加载操作
2. **竞争情况**：其他goroutine会短暂阻塞（微秒级）
3. **内存占用**：每个Once实例仅增加约8字节内存

基准测试对比（ns/op）：
```
BenchmarkOnce-8         12.7           // sync.Once
BenchmarkMutex-8         45.2           // 手动互斥锁实现
BenchmarkAtomic-8        15.3           // 原子操作实现
```

## 注意事项

1. **不要复制Once实例**：这会导致初始化多次执行
   ```go
   var once sync.Once
   onceCopy := once  // 错误！复制后失去同步保证
   ```

2. **初始化函数应幂等**：虽然Once保证只执行一次，但良好的实践要求函数本身可重入

3. **避免初始化死锁**：
   ```go
   var (
       a Once
       b Once
   )
   
   func initA() { b.Do(initB) }
   func initB() { a.Do(initA) }  // 死锁！
   ```

4. **长期运行的初始化**：会阻塞所有等待的goroutine

## 替代方案比较

| 方案                | 适用场景                          | 缺点                          |
|---------------------|---------------------------------|-----------------------------|
| sync.Once           | 大多数惰性初始化场景              | 无法重新初始化                |
| 全局变量+init函数   | 程序启动时就必须初始化的资源      | 失去惰性加载优势              |
| 显式锁              | 需要复杂初始化逻辑的情况          | 实现复杂，容易出错            |
| atomic.Value        | 需要热更新的配置                  | 只适合存储简单值              |

## 总结

`sync.Once`是Go并发工具箱中简单而强大的组件，它：
- 以极小的性能开销提供安全的惰性初始化
- 消除开发者手动处理同步问题的负担
- 保证内存可见性，避免微妙的并发错误

在需要延迟初始化共享资源的场景中，`sync.Once`应该是首选方案。它让开发者能够专注于业务逻辑，而将复杂的并发控制交给标准库处理。
