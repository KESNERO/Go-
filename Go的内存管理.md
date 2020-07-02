# Go 内存管理

[TOC]

## 建议

- 如果程序要修改的数据还会被其他的Go程访问，那么就需要将这些对数据的访问序列化
- 可以使用信道通信来序列化、保护访问， 也可以使用其他本质的方法，比如定义在sync, sync/atomic 包里的那些

## Happens Before

- 在一个Go程中，读和写必须按照程序指定的顺序执行。编译器或者处理器可能会对程序的读写操作重新排序，但是这个排序也是在不影响程序达到的效果下执行的，比如：a = 1; b = 2;一个Go程可能看到的是a先被赋值，b后被赋值；但是另一个Go程可能看到的是b先被赋值，a后被赋值
- 在一个Go程中， happens-before 就是程序表达顺序的方式
- 满足下面条件下读操作`r`**被允许观测**v的写入`w`
    1. r 没有在 w 之前发生
    2. 没有任何额外的读v的写操作w'发生在w之后，r前面
- 为了保证读操作`r`读到的`v`变量观测到的确实是`w`的操作，我们要保证`w`是对于`r`能观测到的唯一写操作，也就是`r`**被保证观测到**`w`要满足下面的条件
    1. `w`在`r`前发生
    2. 任何其他的对共享变量`v`的写入要发生在`w`之前或者`r`之后
- **被保证观测到**条件要比**被允许观测**条件苛刻，保证了任何并发操作都不能和w、r同时发生
- 在一个Go程中没有任何并发，所以两种条件是等价的。当多个Go程共享对变量v的访问的时候，它们必须使用同步事件来建立happens-before条件来保证读操作观测到的是预期的写操作
- 在Go内存模型中初始化一个变量为零值可以视为一次对该变量的写入操作
- `machine-word-sized`等于一个机器一次可以处理的bit数，现代计算机中是根据cpu类型通常是32位或64位
- 对大于一个`machine-word-sized`的读写操作是按照未知的顺序执行

## 同步

### 初始化

- 程序的初始化会在一个单独的Go程中跑，但这个Go程可能会创建其他同时运行的Go程
- 如果一个包`p`导入了包`q`，那么`q`包的初始化会发生在`p`之前
- `main.main`函数会在所有的`init`结束之后才执行

### Go程创建

- 使用`go`语句创建新Go程的操作会发生在Go程执行之前（废话，但是很严谨）

### Go程销毁

- Go程的结束不保证会在主程序的某个事件之前发生，比如：
```go
var a string

func hello() {
	go func() { a = "hello" }()
	print(a)
}
```
- 新的Go程对于a的写操作不一定在主程序的print函数之前发生
- 一些疯狂的编译器可能会把整个Go程部分的代码移除
- 如果想让一个Go程的影响被其他Go程观测到， 那么必须使用一些同步机制，比如使用锁、信道来建立一个相对顺序

### 信道通信

- 信道通信是Go程之间同步的主流方式，每个对特定信道的数据发送对应一个对该信道的接收
- 一个对信道的发送发生在对应该信道的接收之前
```go
var c = make(chan int, 10)
var a string

func f() {
	a = "hello, world"
	c <- 0
}

func main() {
	go f()
	<-c
	print(a)
}
```
- 上面函数中可以保证print函数输出的是`"hello, world"`。因为对变量`a`的写入发生在发送数据到信道`c`之前，print函数的执行发生在从信道接收数据之后
- 如果替换上面例子中的 `c<-0` 为 `close(c)` 也能保证预期的行为
- 从unbuffered信道接收数据的行为会发生在关闭信道之前
```go
var c = make(chan int)
var a string

func f() {
	a = "hello, world"
	<-c
}
func main() {
	go f()
	c <- 0
	print(a)
}
```
- 上述例子也能保证print函数可以正常输出`"hello, world"`。
- 如果信道是一个buffered信道， 那么程序不能保证正确输出`"hello, world"`
- 一个容量为C的信道的第k个接收，一定会发生在k+C的发送之前（这个之前并不等于是k+C-1的发送之后）
- 这个规则泛化了之前的规则到buffered信道，该规则允许对buffered信道进行计数信号量，容量的大小相当于同时使用该信道的用例，向信道发送数据需要获取信号量，从信道接收数据会释放信号量，这是一个比较常用的控制并发数的方式
```go
var limit = make(chan int, 3)

func main() {
	for _, w := range work {
		go func(w func()) {
			limit <- 1
			w()
			<-limit
		}(w)
	}
	select{}
}
```
- 上述函数会从work list获取工作w并发的进行， 但是会通过信道`limit`限制仅3个工作在同时进行

### 锁

- `sync`包实现了两个锁数据类型，`sync.Mutex` 和 `sync.RWMutex`
- 对于所有的 `sync.Mutex` 或 `sync.RWMutex` 变量 `l`， 当 `n<m` 的时候 `n` 个 `l.Unlock()` 会发生在 `m` 个 `l.Lock()` 返回之前（也就是说当执行Lock()之后，没有触发Unlock()之前，新的Lock()不会被执行）
- 对于 `sync.RWMutex` 变量 `l`， 第n个`l.RLock()`发生在第n个`l.Unlock()`之后， 第n个`l.RUnlock()`发生在第n+1个`l.Lock()`之前（比较绕口，但是注意表达中的之后和之前即可）

### Once

- `sync`包通过`Once`数据类型在多个Go程同时存在的时候提供一个安全的初始化机制
- 多个线程可以调用`once`， 对于特定的`f`，Do(f)只有一个f()将会被执行，其他的调用都会被阻挡，直到f()执行完毕

```go
package main

import (
	"fmt"
	"sync"
	"time"
)

var a string
var once sync.Once

func setup() {
	a = "hello world"
}

func setdown() {
	a = "goodbye world"
}

func doprint(i int) {
	setup()
	once.Do(setdown) // 这里可以使用 setdown() 来替代看看会发生什么变化
	fmt.Println(i, a)
}

func main() {
	for i := 0; i < 20; i++ {
		go doprint(i)
	}
	time.Sleep(time.Second * 2)
}
```

## 错误的同步

- 假设，读操作`r`在观测同时进行的写操作`w`的数据，这并不保证`r`之前的读操作能读到`w`写操作之前的写入
```go
var a, b int

func f() {
	a = 1
	b = 2
}

func g() {
	print(b)
	print(a)
}

func main() {
	go f()
	g()
}
```
- 上述代码可能输出2和0， 这可能使部分认知作废
- 可能有些人想双重验证来减少锁机制带来的开销， 比如
```go
package main

import (
	"fmt"
	"sync"
	"time"
)

var once sync.Once
var a string
var done bool

func setup() {
	a = "hello, world" // 可能第三个执行
	done = true        // 可能第一个执行
}

func doprint() {
	if !done { // 可能第二个执行
		once.Do(setup)
	}
	fmt.Println(a) // 可能在第二个执行之后执行
}

func main() {
	go doprint()
	go doprint()
	time.Sleep(time.Second * 1)
}
```
- 上述代码无法保证观测done等于观测a， 这可能会产生非预期的空字符串输出
- 另一个错误的认知可能是忙等待一个变量
```go
package main

import (
	"fmt"
)

var a string
var done bool

func setup() {
	a = "hello, world"
	done = true
}

func main() {
	go setup()
	for !done {
	}
	fmt.Println(a)
}
```
- 上述代码并不保证done的初始化暗示着a已经初始化好，所以这个程序可能输出一个空字符串。或者更糟糕的情况，done的写入可能不会被main函数观测到（因为bool变量的初始化零值是false，setup函数未开始执行前就已经读取并判断完done的值）
- 存在一个该主题的微妙的变体
```go
package main

import (
	"fmt"
)

// T comment
type T struct {
	msg string
}

var g *T

func setup() {
	t := new(T)
	t.msg = "hello, world"
	g = t
}

func main() {
	go setup()
	for g == nil {
	}
	fmt.Println(g.msg)
}
```
- 即使main函数观测到了`g != nil`而跳出空循环，但是也未必能观测到 `t.msg = "hello, world"`
- 对应以上的所有情况，解决办法只有一个，显式使用同步机制