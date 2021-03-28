---
layout: post
title: go语言happens-before原则及应用
categories: Go
description: go语言happens-before原则及应用
keywords: go, happens-before, 并发, goroutine 
---
了解go中`happens-before`规则，寻找并发程序不确定性中的确定性。

# 引言

先抛开你所熟知的信号量、锁、同步原语等技术，思考这个问题：如何保证并发读写的准确性？一个没有任何并发编程经验的程序员可能会觉得很简单：这有什么问题呢，同时读写能有什么问题，最多就是读到过期的数据而已。一个理想的世界当然是这样，只可惜实际上的机器世界往往隐藏了很多不容易被察觉的事情。至少有两个行为会影响这个结论：
* 编译器往往有指令重排序的优化；例如程序员看到的源代码是`a=3; b=4;`，而实际上执行的顺序可能是`b=4; a=3；`，这是因为编译器为了优化执行效率可能对指令进行重排序；
* 高级编程语言所支持的运算往往不是原子化的；例如`a += 3`实际上包含了读变量、加运算和写变量三次原子操作。既然整个过程并不是原子化的，就意味着随时有其它“入侵者”侵入修改数据。更为隐藏的例子：对于变量的读写甚至可能都不是原子化的。不同机器读写变量的过程可能是不同的，有些机器可能是64位数据一次性读写，而有些机器是32位数据一次读写。这就意味着一个64位的数据在后者的读写上实际上是分成两次完成的！试想，如果你试图读取一个64位数据的值，先读取了低32的数据，这时另一个线程切进来修改了整个数据的值，最后你再读取高32的值，将高32和低32的数据拼成完整的值，很明显会得到一个预期以外的数据。

看起来，整个并发编程的世界里一切都是不确定的，我们不知道每次读取的变量到底是不是及时、准确的数据。幸运的是，很多语言都有一个**`happens-before`**的规则，能帮助我们在不确定的并发世界里寻找一丝确定性。
# happens-before
你可以把`happens-before`看作一种特殊的比较运算，就好像`>`、`<`一样。对应的，还有`happens-after`，它们之间的关系也好像`>`、`<`一样：
> 如果a happens-before b，那么b happens-after a

那是否存在既不满足`a happens-before b`，也不满足`b happens-before a`的情况呢，就好像既不满足`a>b`，也不满足`b>a`（意味着`b==a`)？当然是肯定的，这种情况称为：a和b `happen concurrently`，也就是同时发生，这就回到我们之前所熟知的世界里了。

`happens-before`有什么用呢？它可以用来帮助我们厘清两个并发读写之间的关系。对于并发读写问题，我们最关心的经常是reader是否能准确观察到writer写入的值。`happens-before`正是为这个问题设计的，具体来说，要想让某次读取r准确观察到某次写入w，只需满足：
1. w `happens-before` r；
2. 对变量的其它写入w1，要么w1 `happens-before` w，要么r `happens-before` w1；简单理解就是没有其它写入覆盖这次写入；

只要满足这两个条件，那我们就可以自信地肯定我们一定能读取到正确的值。

一个新的问题随之诞生：那如何判断`a happens-before b`是否成立呢？你可以类比思考数学里如何判断`a > b`是否成立的过程，我们的做法很简单：
1. 基于一些简单的公理；例如自然数的自然大小：`3>2>1`
2. 基于比较运算符的传递性，也就是如果`a>b且b>c`，则`a>c`

判断`a happens-before b`的过程也是类似的：根据一些简单的明确的`happens-before`关系，再结合`happens-before`的传递性，推导出我们所关心的w和r之间的`happens-before`关系。
> `happens-before`传递性：如果a `happens-before` b，且b `happens-before` c，则a `happens-before` c

因此我们只需要了解这些明确的`happens-before`关系，就能在并发世界里寻找到宝贵的确定性了。

# go语言中的happens-before关系
具体的happens-before关系是因语言而异的，这里只介绍go语言相关的规则，感兴趣可以直接阅读[官方文档](https://golang.org/ref/mem)，有更完整、准确的说明。
## 自然执行
首先，最简单也是最直观的`happens-before`规则：
> 在**同一个goroutine里**，书写在前的代码`happens-before`书写在后的代码。

例如：
```
a = 3; // (1)
b = 4; // (2)
```
则(1) `happens-before` (2)。我们上面提到指令重排序，也就是实际执行的顺序与书写的顺序可能不一致，但happens-before与指令重排序并不矛盾，即使可能发生指令重排序，我们依然可以说（1) `happens-before` (2)。
## 初始化
每个go文件都可以有一个`init`方法，用于执行某些初始化逻辑。当我们开始执行某个`main`方法时，go会先在一个goroutine里做初始化工作，也就是执行所有go文件的`init`方法，这个过程中go可能创建多个goroutine并发地执行，因此通常情况下各个`init`方法是没有`happens-before`关系的。关于`init`方法有两条`happens-before`规则：
> 1.a 包导入了 b包，此时b包的`init`方法`happens-before` a包的所有代码；
> 
> 2.所有`init`方法`happens-before` `main`方法；

## goroutine
goroutine相关的规则主要是其创建和销毁的：

> 1.goroutine的创建 `happens-before` 其执行；
> 
> 2.goroutine的完成**不保证**`happens-before`任何代码；

第一条规则举个简单的例子即可：
```go
var a string

func f() {
	fmt.Println(a) // (1)
}

func hello() {
	a = "hello, world" // (2)
	go f() // (3)
}
```
因为goroutine的创建 `happens-before` 其执行，所以(3) `happens-before` (1)，又因为自然执行的规则(2) `happens-before` (3)，根据传递性，所以(2) `happens-before` (1)，这样保证了我们每次打印出来的都是"hello world"而不是空字符串。

第二条规则是少见的否定句式，同样举个简单的例子：
```go
var a string

func hello() {
	go func() { a = "hello" }() // (1)
	fmt.Println(a) // (2)
}
```
由于goroutine的完成**不保证**`happens-before`任何代码，因此(1) happens-before (2)不成立，这样我们就不能保证每次打印的结果都是"hello"。
## 通道
通道channel是go语言中用于goroutine之间通信的主要渠道，因此理解通道之间的happens-before规则也至关重要。
> 1.对于缓冲通道，向通道发送数据`happens-before`从通道接收到数据

结合一个例子：
```go
var c = make(chan int, 10)
var a string

func f() {
	a = "hello, world" // (1)
	c <- 0 // (2)
}

func main() {
	go f() // (3)
	<-c // (4)
	fmt.Println(a) // (5)
}
```
`c`是一个缓冲通道，因此向通道发送数据`happens-before`从通道接收到数据，也就是(2) `happens-before` (4)，再结合自然执行规则以及传递性不难推导出(1) happens-before (5)，也就是打印的结果保证是"hello world"。
有趣的是，如果我们把c的定义改为`var c = make(chan int)`也就是无缓冲通道，上面的结论就不存在了***（注1）***，打印的结果不一定为"hello world"，这是因为：
> 2.对于无缓冲通道，从通道接收数据`happens-before`向通道发送数据

我们可以将上述例子稍微调整下：
```go
var c = make(chan int)
var a string

func f() {
	a = "hello, world" // (1)
    <- c // (2)
}

func main() {
	go f() // (3)
	c <- 10 // (4)
	fmt.Println(a) // (5)
}
```
对于无缓冲通道，(2) `happens-before` (4)，再根据传递性，(1) `happens-before` (5)，因此依然可以保证打印的结果是"hello world"。

可以这么理解这两者的差异，缓冲通道的目的是缓冲发送方发送的数据，这就意味着发送方很可能先发送数据，过一段时间后接收方才接收，或者发送方发送的速度超过接收方接收的速度，因此缓冲通道的发送`happens-before`接收就自然而然了；相反，非缓冲通道是没有缓冲区的，先发起的发送方和接收方都会阻塞至另一方准备好，如果我们使用了非缓冲通道，则意味着我们认为我们的场景下接收发生在发送之前，否则我们就会使用缓冲通道了，因此非缓冲通道的接收`happens-before`发送。

> 3.对于缓冲通道，第k次接收`happens-before`第`k+C`次发送，`C`是缓冲通道的容量

这条规则是缓冲通道的通用规则（有趣的是，上面针对非缓冲通道的第2条规则也可以看成这个规则的特例：`C`取0）。这个规则看起来复杂，我们看个例子就清晰了：
```go
var limit = make(chan int, 3)

func main() {
    // work是一个worker列表，其中的元素w都是可执行函数
	for _, w := range work {
		go func(w func()) {
			limit <- 1 // (1)
			w() // (2)
			<-limit // (3)
		}(w)
	}
	select{}
}
```
我们先套用一下上面的规则，则：“第1次(3)`happens-before`第4次(1)”、“第2次(3)`happens-before`第5次(1)”、“第3次(3)`happens-before`第6次(1)”……，再结合传递性：“第1次(2)`happens-before`第1次(3)`happens-before`第4次(1)`happens-before`第4次(2)”、“第2次(2)`happens-before`第2次(3)`happens-before`第5次(1)`happens-before`第5次(2)”……，简单地说：“第1次(2)`happens-before`第4次(2)”、“第2次(2)`happens-before`第5次(2)”、“第3次(2)`happens-before`第6次(2)”……这样我们虽然没有做任何分批，却事实上将workers分成三个一批、每批并发地执行。这就是通过这条happens-before规则保证的。

这个规则理解起来其实也很简单，`C`是通道的容量，如果无法保证第k次接收`happens-before`第`k+C`次发送，那通道的缓冲就不够用了。

> 注1：以上是官方文档给的规则和例子，但是笔者在尝试将第一个例子的`c`改成无缓冲通道后发现每次打印的依然稳定是"hello world"，并没有出现预期的空字符串，也就是看起来`happens-before`规则依然成立。但既然官方文档说无法保证，那我们开发时还是按照`happens-before`不成立比较好。

## 锁
锁也是并发编程里非常常用的一个数据结构。go语言中支持的锁主要有两种：`sync.Mutex`和`sync.RWMutex`，即普通锁和读写锁（读写锁的原理可以参见另一篇[文章](https://chensk.github.io//2021/03/26/go-rwlock/)）。普通锁的`happens-before`规则也很直观：

> 1.对锁实例调用`n`次`Unlock` `happens-before` 调用`Lock` `m`次，只要`n < m`

请看这个例子：
```go
var l sync.Mutex
var a string

func f() {
	a = "hello, world" // (1)
	l.Unlock() // (2)
}

func main() {
	l.Lock() // (3)
	go f() // (4)
	l.Lock() // (5)
	print(a) // (6)
}
```
上面调用了`Unlock`一次，`Lock`两次，因此(2) `happens-before` (5)，从而(1) `happens-before` (6)

而读写锁的规则为：
> 2.对读写锁实例的某一次`Unlock`调用，`happens-after`的`RLock`调用对应的`RUnlock`调用`happens-before`下一次`Lock`调用。

其实本质就是读写锁的原理：读写互斥，简单地理解就是写锁释放后先获取了读锁，则读锁的释放会`happens-before` 下一次写锁的获取。注意上面的规则是“存在”，而不是“任意”。

## Once
sync中还提供了一个`Once`的数据结构，用于控制并发编程中只执行一次的逻辑，例如：
```go
var a string
var once sync.Once

func setup() {
   a = "hello, world"
   fmt.Println("set up")
}

func doprint() {
   once.Do(setup)
   fmt.Println(a)
}

func twoprint() {
	go doprint()
	go doprint()
}
```
会打印"hello, world"两次和"set up"一次。`Once`的`happens-before`规则也很直观：
> 第一次执行`Once.Do` `happens-before`其余的`Once.Do`

# 应用
掌握了上述的基本`happens-before`规则，可以结合起来分析更复杂的场景了，来看这个例子：
```go
var a, b int

func f() {
	a = 1 // (1)
	b = 2 // (2)
}

func g() {
	print(b) // (3)
	print(a) // (4)
}

func main() {
	go f()
	g()
}
```
这里(1) `happens-before` (2)，(3) `happens-before`(4)，但是(1)与(3)、(4)之间以及(2)与(3)、(4)之间并没有`happens-before`关系，这时候结果是不确定的，一种有趣的结果是2、0，也就是(1)、(2)之间发生了指令重排序。现在让我们修改一下上面的代码，让它按我们预期的逻辑运行：要么打印0、0，要么打印1、2。
## 使用锁
```go
var a, b int
var lock sync.Mutex

func f() {
    lock.Lock() // (1)
	a = 1 // (2)
	b = 2 // (3)
    lock.Unlock() // (4)
}

func g() {
    lock.Lock() // (5)
	print(b) // (6)
	print(a) // (7)
    lock.Unlock() // (8)
}

func main() {
	go f()
	g()
}
```
回想下锁的规则：
> 1.对锁实例调用`n`次`Unlock` `happens-before` 调用`Lock` `m`次，只要`n < m`

这里存在两种可能：要么(4) `happens-before` (5)，要么(8) `happens-before` (1)，会分别推导出两种结果：(6) `happens-before` (7) `happens-before` (2) `happens-before` (3) ，以及(2) `happens-before` (3) `happens-before` (6) `happens-before` (7)，也就分别对应“0、0”和“1、2”两种结果。
## 使用通道
```go
var a, b int
var c = make(chan int, 1)

func f() {
   <- c
   a = 1 // (2)
   b = 2 // (3)
   c <- 1
}

func g() {
   <- c
   print(b) // (6)
   print(a) // (7)
   c <- 1
}

func test() {
   wg := sync.WaitGroup{}
   wg.Add(3)
   go func(){
      defer wg.Done()
      f()
   }()
   go func(){
      defer wg.Done()
      g()
   }()
   go func(){
      defer wg.Done()
      c <- 1
   }()
   wg.Wait()
   close(c)
}
```
总之，如果无法确定并发读写之间的`happens-before`关系，那么最好使用同步工具明确它们之间的关系，例如锁或者通道。不要给程序留下不确定的可能，毕竟确定性就是编程的魅力！