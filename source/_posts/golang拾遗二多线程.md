title: golang拾遗二多线程
author: 江小渔
tags:
  - golang
categories:
  - golang
date: 2019-12-09 14:38:00
---

# goroutine
- 下面代码会输出什么
```
package main
 
import "fmt"
 
func main() {
	for i := 0; i < 10; i++ {
		go func() {
			fmt.Println(i)
		}()
	}
	//time.Sleep(3*time.Microsecond)
}
```
1. 不会有任何内容被打印出来
    1. go函数的执行时间总是会明显滞后于它所属的go语句的执行时间。当然了，这里所说的“明显滞后”是对于计算机的 CPU 时钟和 Go 程序来说的
    2. 只要go语句本身执行完毕，Go 程序完全不会等待go函数的执行，它会立刻去执行后边的语句。这就是所谓的异步并发地执行
    3. 
2. 打印出 10 个10
    1. 运行一下你就知道了~
- struct{}类型值的表示法只有一个，即：struct{}{}
- 多个 goroutine 按照既定的顺序运行?
```go
package main

import (
	"fmt"
	"sync/atomic"
	"time"
)

func main() {
	var count uint32
	trigger := func(i uint32, fn func()) {
		for {
			if n := atomic.LoadUint32(&count); n == i {
				fn()
				atomic.AddUint32(&count, 1)
				break
			}
			time.Sleep(time.Nanosecond)
		}
	}
	for i := uint32(0); i < 10; i++ {
		go func(i uint32) {
			fn := func() {
				fmt.Println(i)
			}
			trigger(i, fn)
		}(i)
	}
	trigger(10, func() {}) //想让主 goroutine 最后一个运行完毕,等待goroutine
}
```
1. 操作变量count的时候使用的都是原子操作。这是由于trigger函数会被多个 goroutine 并发地调用，所以它用到的非本地变量count，就被多个用户级线程共用了。因此，对它的操作就产生了竞态条件（race condition），破坏了程序的并发安全性


# 条件变量sync.Cond
- 条件变量的初始化离不开互斥锁，并且它的方法有的也是基于互斥锁的。
```go
lock.Lock()
for mailbox == 1 {
sendCond.Wait()
}
mailbox = 1
lock.Unlock()
recvCond.Signal()
```
Q1. 为什么先要锁定条件变量基于的互斥锁，才能调用它的Wait方法？
Q2. 为什么要用for语句来包裹调用其Wait方法的表达式，用if语句不行吗？
- 条件变量的Wait方法主要做了四件事。
1. 把调用它的 goroutine（也就是当前的goroutine）加入到当前条件变量的通知队列中。
2. 解锁当前的条件变量基于的那个互斥锁。
3. 让当前的 goroutine 处于等待状态，等到通知到来时再决定是否唤醒它。此时，这个 goroutine 就会阻塞在调用这个Wait方法的那行代码上。
4. 如果通知到来并且决定唤醒这个 goroutine，那么就在唤醒它之后重新锁定当前条件变量基于的互斥锁。自此之后，当前的 goroutine 就会继续执行后面的代码了。

A1: 因为条件变量的Wait方法在阻塞当前的 goroutine 之前会解锁它基于的互斥锁，所以在调用该Wait方法之前我们必须先锁定那个互斥锁，否则在调用这个Wait方法时，就会引发一个不可恢复的 panic。

A2:  很显然，if语句只会对共享资源的状态检查一次，而for语句却可以做多次检查，直到这个状态改变为止。那为什么要做多次检查呢？

这主要是为了保险起见。如果一个 goroutine 因收到通知而被唤醒，但却发现共享资源的状态，依然不符合它的要求，那么就应该再次调用条件变量的Wait方法，并继续等待下次通知的到来。

这种情况是很有可能发生的，具体如下面所示。

1. 有多个 goroutine 在等待共享资源的同一种状态。比如，它们都在等mailbox变量的值不为0的时候再把它的值变为0，这就相当于有多个人在等着我向信箱里放置情报。虽然等待的 goroutine 有多个，但每次成功的 goroutine 却只可能有一个。别忘了，条件变量的Wait方法会在当前的 goroutine 醒来后先重新锁定那个互斥锁。在成功的 goroutine 最终解锁互斥锁之后，其他的 goroutine 会先后进入临界区，但它们会发现共享资源的状态依然不是它们想要的。这个时候，for循环就很有必要了。

2. 共享资源可能有的状态不是两个，而是更多。比如，mailbox变量的可能值不只有0和1，还有2、3、4。这种情况下，由于状态在每次改变后的结果只可能有一个，所以，在设计合理的前提下，单一的结果一定不可能满足所有 goroutine 的条件。那些未被满足的 goroutine 显然还需要继续等待和检查。

3. 有一种可能，共享资源的状态只有两个，并且每种状态都只有一个 goroutine 在关注，就像我们在主问题当中实现的那个例子那样。不过，即使是这样，使用for语句仍然是有必要的。原因是，在一些多 CPU 核心的计算机系统中，即使没有收到条件变量的通知，调用其Wait方法的 goroutine 也是有可能被唤醒的。这是由计算机硬件层面决定的，即使是操作系统（比如 Linux）本身提供的条件变量也会如此。

综上所述，在包裹条件变量的Wait方法的时候，我们总是应该使用for语句。

Q3: 条件变量的Signal方法和Broadcast方法有哪些异同？
A3: 条件变量的Signal方法和Broadcast方法都是被用来发送通知的，不同的是，前者的通知只会唤醒一个因此而等待的 goroutine，而后者的通知却会唤醒所有为此等待的 goroutine。条件变量的Wait方法总会把当前的 goroutine 添加到通知队列的队尾，而它的Signal方法总会从通知队列的队首开始查找可被唤醒的 goroutine。所以，因Signal方法的通知而被唤醒的 goroutine 一般都是最早等待的那一个。


# 进程线程goroutine

# 原子操作
Q1: sync/atomic包中提供了几种原子操作？可操作的数据类型又有哪些？

A1: sync/atomic包中的函数可以做的原子操作有：加法（add）、比较并交换（compare and swap，简称 CAS）、加载（load）、存储（store）和交换（swap）。

这些函数针对的数据类型并不多。但是，对这些类型中的每一个，sync/atomic包都会有一套函数给予支持。这些数据类型有：int32、int64、uint32、uint64、uintptr，以及unsafe包中的Pointer。不过，针对unsafe.Pointer类型，该包并未提供进行原子加法操作的函数。

此外，sync/atomic包还提供了一个名为Value的类型，它可以被用来存储任意类型的值。

Q2: atomic.AddUint32和atomic.AddUint64函数做原子减法,如何实现
```
    num := uint32(18)
	fmt.Printf("The number: %d\n", num)
	delta := int32(-3)
	atomic.AddUint32(&num, uint32(delta))
	fmt.Printf("The number: %d\n", num)
	atomic.AddUint32(&num, ^uint32(-(-3)-1))
	fmt.Printf("The number: %d\n", num)

	fmt.Printf("The two's complement of %d: %b\n",
		delta, uint32(delta)) // -3的补码。
	fmt.Printf("The equivalent: %b\n", ^uint32(-(-3)-1)) // 与-3的补码相同。
	fmt.Println()
	//uint32(delta)和^uint32(-N-1)) 是两种求补码的方式
```
Q3: 比较并交换操作与交换操作相比有什么不同？优势在哪里？

A3: 比较并交换操作即 CAS操作，是有条件的交换操作，只有在条件满足的情况下才会进行值的交换。
```golang
for {
if atomic.CompareAndSwapInt32(&num2, 10, 0) {
fmt.Println("The second number has gone to zero.")
break
}
time.Sleep(time.Millisecond * 500)
}
```

## 使用建议
1. 不要把内部使用的原子值暴露给外界。比如，声明一个全局的原子变量并不是一个正确的做法。这个变量的访问权限最起码也应该是包级私有的。
2. 如果不得不让包外，或模块外的代码使用你的原子值，那么可以声明一个包级私有的原子变量，然后再通过一个或多个公开的函数，让外界间接地使用到它。注意，这种情况下不要把原子值传递到外界，不论是传递原子值本身还是它的指针值。
3. 如果通过某个函数可以向内部的原子值存储值的话，那么就应该在这个函数中先判断被存储值类型的合法性。若不合法，则应该直接返回对应的错误值，从而避免 panic 的发生。
4. 如果可能的话，我们可以把原子值封装到一个数据类型中，比如一个结构体类型。这样，我们既可以通过该类型的方法更加安全地存储值，又可以在该类型中包含可存储值的合法类型信息。

# sync.WaitGroup和sync.Once
- 使用WaitGroup值:先统一Add，再并发Done，最后Wait
- Once类型使用互斥锁和原子操作实现了功能，而WaitGroup类型中只用到了原子操作。
 