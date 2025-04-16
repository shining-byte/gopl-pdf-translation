# 第9章 共享变量的并发

在前一章中，我们介绍了几个使用goroutine和channel直接自然地表达并发的程序。然而，在这个过程中我们忽略了一些重要而微妙的问题，程序员在编写并发代码时必须牢记这些问题。在本章中，我们将更仔细地研究并发的机制。特别是，我们将指出在多个goroutine之间共享变量时可能出现的一些问题，识别这些问题的分析技术，以及解决这些问题的模式。最后，我们将解释goroutine和操作系统线程之间的一些技术差异。

## 9.1 竞态条件

在顺序程序中（即只有一个goroutine的程序），程序的步骤按照程序逻辑确定的熟悉执行顺序发生。例如，在一系列语句中，第一条语句在第二条语句之前执行，依此类推。在一个有两个或更多goroutine的程序中，每个goroutine内的步骤按照熟悉的顺序发生，但一般来说，我们不知道一个goroutine中的事件x是在另一个goroutine中的事件y之前发生，还是在之后发生，或是同时发生。当我们不能自信地说一个事件在另一个事件之前发生时，事件x和y是并发的。

考虑一个在顺序程序中工作正常的函数。如果该函数在并发调用时（即从两个或多个goroutine调用，没有额外的同步）仍然能正确工作，那么该函数是并发安全的。我们可以将这个概念推广到一组协作的函数，例如特定类型的方法和操作。如果一个类型的所有可访问方法和操作都是并发安全的，那么该类型是并发安全的。

我们可以使程序并发安全，而不必使程序中的每个具体类型都是并发安全的。事实上，并发安全的类型是例外而非规则，因此只有在类型的文档说明这是安全的情况下，才应该并发访问变量。我们通过将大多数变量限制在单个goroutine中或维护更高级别的互斥不变量来避免对大多数变量的并发访问。我们将在本章中解释这些术语。相比之下，导出的包级函数通常预期是并发安全的。由于包级变量不能被限制在单个goroutine中，修改这些变量的函数必须强制执行互斥。

函数在并发调用时可能无法工作的原因有很多，包括死锁、活锁和资源饥饿。我们没有篇幅讨论所有这些原因，因此我们将重点关注最重要的一个：竞态条件。

竞态条件是指程序在某些goroutine操作的交叉执行下不能给出正确结果的情况。竞态条件非常隐蔽，因为它们可能潜伏在程序中，很少出现，可能只在重负载或使用某些编译器、平台或架构时才会显现。这使得它们难以重现和诊断。

传统上，通过财务损失的比喻来解释竞态条件的严重性，因此我们将考虑一个简单的银行账户程序。

```go
// Package bank 实现一个只有一个账户的银行。
package bank

var balance int

func Deposit(amount int) {
    balance = balance + amount
}

func Balance() int {
    return balance
}
```

（我们可以将Deposit函数的主体写成`balance += amount`，这是等价的，但较长的形式将简化解释。）对于这样一个简单的程序，我们一眼就能看出对Deposit和Balance的任何调用序列都会给出正确的答案，即Balance会报告之前所有存入金额的总和。然而，如果我们不是按顺序调用这些函数，而是并发调用，Balance就不再保证给出正确的答案。

考虑以下两个goroutine，它们代表一个联合银行账户上的两笔交易：

```go
// Alice:
go func() {
    bank.Deposit(200)                // A1
    fmt.Println("=", bank.Balance()) // A2
}()

// Bob:
go bank.Deposit(100)                // B
```

Alice存入200美元，然后检查她的余额，而Bob存入100美元。由于步骤A1和A2与B并发发生，我们无法预测它们的执行顺序。直观上，似乎只有三种可能的顺序，我们称之为“Alice优先”、“Bob优先”和“Alice/Bob/Alice”。下表显示了每个步骤后balance变量的值。引号中的字符串表示打印的余额单。

| Alice优先 | Bob优先 | Alice/Bob/Alice |
|-----------|---------|-----------------|
| 0         | 0       | 0               |
| A1 200    | B 100   | A1 200          |
| A2 "= 200" | A1 300 | B 300           |
| B 300     | A2 "= 300" | A2 "= 300"     |

在所有情况下，最终余额都是300美元。唯一的区别是Alice的余额单是否包括Bob的交易，但客户对这两种情况都满意。然而，这种直觉是错误的。还有第四种可能的结果，即Bob的存款发生在Alice存款的中间，在读取余额（balance + amount）之后但在更新（balance = ...）之前，导致Bob的交易消失。这是因为Alice的存款操作A1实际上是由两个操作组成的：读取和写入，我们称之为A1r和A1w。以下是问题交织的情况：

| 数据竞态 | 0         |
|----------|-----------|
| A1r      | 0         |
| ... = balance + amount | B 100 |
| A1w      | 200       |
| balance = ... | A2 "= 200" |

在A1r之后，表达式balance + amount计算为200，因此这是A1w期间写入的值，尽管中间有存款。最终余额只有200美元。银行从Bob那里多赚了100美元。

这个程序包含一种特定类型的竞态条件，称为数据竞态。数据竞态发生在两个goroutine并发访问同一个变量且至少有一个访问是写入时。如果数据竞态涉及的类型大于单个机器字（如接口、字符串或切片），情况会更加混乱。以下代码将x并发更新为两个不同长度的切片：

```go
var x []int
go func() { x = make([]int, 10) }()
go func() { x = make([]int, 1000000) }()
x[999999] = 1 // 注意：未定义行为；可能的内存损坏！
```

x在最后一条语句中的值未定义；它可能是nil，长度为10的切片，或长度为1,000,000的切片。但回想一下，切片有三个部分：指针、长度和容量。如果指针来自第一个make调用，长度来自第二个调用，x将是一个嵌合体，一个名义长度为1,000,000但底层数组只有10个元素的切片。在这种情况下，存储到元素999,999将覆盖任意远处的内存位置，其后果无法预测且难以调试和定位。这个语义雷区称为未定义行为，C程序员对此非常熟悉；幸运的是，它在Go中不像在C中那样麻烦。

甚至认为并发程序是几个顺序程序的交织执行的直觉也是错误的。正如我们将在9.4节中看到的，数据竞态可能导致更奇怪的结果。许多程序员——甚至一些非常聪明的人——偶尔会为他们程序中已知的数据竞态提供理由：“互斥的成本太高”、“这个逻辑只是用于日志记录”、“我不介意丢失一些消息”等等。在给定的编译器和平台上没有出现问题可能会给他们虚假的信心。一个好的经验法则是：没有良性的数据竞态。

那么，我们如何在程序中避免数据竞态呢？我们将重复定义，因为它非常重要：数据竞态发生在两个goroutine并发访问同一个变量且至少有一个访问是写入时。从这个定义可以得出，有三种方法可以避免数据竞态。

第一种方法是不写入变量。考虑下面的映射，它在每个键第一次被请求时惰性填充。如果Icon是顺序调用的，程序工作正常，但如果Icon是并发调用的，就会出现访问映射的数据竞态。

```go
var icons = make(map[string]image.Image)

func loadIcon(name string) image.Image

// 注意：不是并发安全的！
func Icon(name string) image.Image {
    icon, ok := icons[name]
    if !ok {
        icon = loadIcon(name)
        icons[name] = icon
    }
    return icon
}
```

相反，如果我们在创建额外的goroutine之前用所有必要的条目初始化映射，并且之后不再修改它，那么任何数量的goroutine都可以安全地并发调用Icon，因为每个调用只读取映射。

```go
var icons = map[string]image.Image{
    "spades.png":   loadIcon("spades.png"),
    "hearts.png":   loadIcon("hearts.png"),
    "diamonds.png": loadIcon("diamonds.png"),
    "clubs.png":    loadIcon("clubs.png"),
}

// 并发安全。
func Icon(name string) image.Image { return icons[name] }
```

在上面的例子中，icons变量在包初始化期间被赋值，这发生在程序的main函数开始运行之前。一旦初始化，icons永远不会被修改。从未被修改或不可变的数据结构本质上是并发安全的，不需要同步。但显然，如果更新是必要的，我们不能使用这种方法，比如银行账户。

第二种方法是避免从多个goroutine访问变量。这是前一章中许多程序采用的方法。例如，并发网络爬虫（§8.6）中的主goroutine是唯一访问seen映射的goroutine，聊天服务器（§8.10）中的broadcaster goroutine是唯一访问clients映射的goroutine。这些变量被限制在单个goroutine中。由于其他goroutine不能直接访问变量，它们必须使用通道向限制goroutine发送查询或更新变量的请求。这就是Go格言“不要通过共享内存来通信；相反，通过通信来共享内存”的含义。使用通道请求代理访问限制变量的goroutine称为该变量的监视goroutine。例如，broadcaster goroutine监视对clients映射的访问。

以下是银行示例，将balance变量限制在名为teller的监视goroutine中：

```go
// Package bank 提供一个并发安全的银行，只有一个账户。
package bank

var deposits = make(chan int) // 发送存款金额
var balances = make(chan int) // 接收余额

func Deposit(amount int) { deposits <- amount }
func Balance() int       { return <-balances }

func teller() {
    var balance int // balance 被限制在 teller goroutine 中
    for {
        select {
        case amount := <-deposits:
            balance += amount
        case balances <- balance:
        }
    }
}

func init() {
    go teller() // 启动监视 goroutine
}
```

即使变量不能在整个生命周期内限制在单个goroutine中，限制仍然是解决并发访问问题的一种方法。例如，在管道中，通过通道将变量的地址从一个阶段传递到下一个阶段来共享变量。如果管道的每个阶段在将变量发送到下一个阶段后避免访问该变量，那么对该变量的所有访问都是顺序的。实际上，变量首先被限制在管道的一个阶段，然后限制在下一个阶段，依此类推。这种纪律有时称为串行限制。在下面的例子中，Cakes首先被限制在baker goroutine中，然后限制在icer goroutine中：

```go
type Cake struct{ state string }

func baker(cooked chan<- *Cake) {
    for {
        cake := new(Cake)
        cake.state = "cooked"
        cooked <- cake // baker 永远不会再碰这个 cake
    }
}

func icer(iced chan<- *Cake, cooked <-chan *Cake) {
    for cake := range cooked {
        cake.state = "iced"
        iced <- cake // icer 永远不会再碰这个 cake
    }
}
```

第三种方法是允许多个goroutine访问变量，但一次只允许一个goroutine访问。这种方法称为互斥，是下一节的主题。

**练习9.1**：向gopl.io/ch9/bank1程序添加一个函数Withdraw(amount int) bool。结果应指示交易是否成功或因资金不足而失败。发送给监视goroutine的消息必须包含要提取的金额和一个新的通道，监视goroutine可以通过该通道将布尔结果发送回Withdraw。