title: golang拾遗一概念
author: 江小渔
tags:
  - golang
categories:
  - golang
date: 2019-12-09 14:36:00
---
# 接口
1. 倾向于是用小的接口定义,很多接口只包含一个方法
```
type Reader interface{
    Read(p []byte) (n int, err error)
}
type Writer interface{
    Write(p []byte) (n int, err error)
}
```
2. 较大的接口定义,可以由多个小接口定义组合而成
```
type ReadWriter interface{
    Reader
    Write
}
```
3. 只依赖于必要功能的最小接口
```
func StoreData(reader Reader) error{
    -
}
```

# 类型断言
1. `value, ok := interface{}(container).([]string)`
2. switch语句
```golang
func main() {
	container := map[int]string{0: "zero", 1: "one", 2: "two"}

	// 方式1。
	_, ok1 := interface{}(container).([]string)
	_, ok2 := interface{}(container).(map[int]string)
	if !(ok1 || ok2) {
		fmt.Printf("Error: unsupported container type: %T\n", container)
		return
	}
	fmt.Printf("The element is %q. (container type: %T)\n",
		container[1], container)

	// 方式2。
	elem, err := getElement(container)
	if err != nil {
		fmt.Printf("Error: %s\n", err)
		return
	}
	fmt.Printf("The element is %q. (container type: %T)\n",
		elem, container)
}

func getElement(containerI interface{}) (elem string, err error) {
	switch t := containerI.(type) {
	case []string:
		elem = t[1]
	case map[int]string:
		elem = t[1]
	default:
		err = fmt.Errorf("unsupported container type: %T", containerI)
		return
	}
	return
}
```
# 通道select
```go
// 准备好几个通道。
intChannels := [3]chan int{
make(chan int, 1),
make(chan int, 1),
make(chan int, 1),
}
// 随机选择一个通道，并向它发送元素值。
index := rand.Intn(3)
fmt.Printf("The index: %d\n", index)
intChannels[index] <- index
// 哪一个通道中有可取的元素值，哪个对应的分支就会被执行。
select {
case <-intChannels[0]:
fmt.Println("The first candidate case is selected.")
case <-intChannels[1]:
fmt.Println("The second candidate case is selected.")
case elem := <-intChannels[2]:
fmt.Printf("The third candidate case is selected, the element is %d.\n", elem)
default:
fmt.Println("No candidate case is selected!")
}
```
- 多渠道选择
```go
select {
    case ret := <-retCh1:
        t.Logf("result %s", ret)
    case ret := <-retCh2:
        t.Logf("result %s", ret)
    default:
        t.Error("No one returned")
}
```
- 超时控制
```go
select {
    case ret := <-retCh:
        t.Logf("result %s", ret)
    case <-time.After(time.Second * 1:
        t.Error("time out")
}
```

# 值类型和引用类型
```
1.值类型：变量直接存储值，内存通常在栈中分配。
值类型：基本数据类型int、float、bool、string以及数组和struct

2.引用类型：变量存储的是一个地址，这个地址存储最终的值。内存通常在 堆上分配。通过GC回收。
引用类型：指针、slice、map、chan等都是引用类型。
```
# 类型之间的组合
- Go 语言中根本没有继承的概念，它所做的是通过嵌入字段的方式实现了类型之间的组合
```golang
type Animal struct {
scientificName string // 学名。
}
type Cat struct {
Name string // 名字
Animal
}
```
# 指针
1. 不可变的值不可寻址
2. 绝大多数被视为临时结果的值都是不可寻址的
3. 若拿到某值的指针可能会破坏程序的一致性，那么就是不安全的，该值就不可寻址
```go
New("little pig").SetName("monster")
//因此，上边这行链式调用会让编译器报告两个错误，一个是果，即：不能在New("little pig")的结果值上调用指针方法。一个是因，即：不能取得New("little pig")的地址。
```
- 通过unsafe.Pointer操纵可寻址的值
```
dog := Dog{"little pig"}
dogP := &dog
dogPtr := uintptr(unsafe.Pointer(dogP))
//先把dogP转换成了一个unsafe.Pointer类型的值，然后紧接着又把后者转换成了一个uintptr的值，并把它赋给了变量dogPtr

namePtr := dogPtr + unsafe.Offsetof(dogP.name)
//unsafe.Offsetof函数用于获取两个值在内存中的起始存储地址之间的偏移量，以字节为单位。
//这两个值一个是某个字段的值，另一个是该字段值所属的那个结构体值

nameP := (*string)(unsafe.Pointer(namePtr))

///unsafe.Pointer+ uintptr突破私有成员访问
```
# Error
- 怎样判断一个错误值具体代表的是哪一类错误？
1. 对于类型在已知范围内的一系列错误值，一般使用类型断言表达式或类型switch语句来判断；
2. 对于已有相应变量且类型相同的一系列错误值，一般直接使用判等操作来判断；
3. 对于没有相应变量且类型未知的一系列错误值，只能使用其错误信息的字符串表示形式来做判断。

# recover
- Go 语言的内建函数recover专用于恢复 panic，或者说平息运行时恐慌。recover函数无需任何参数，并且会返回一个空接口类型的值。


# context.Context类型(单章)
- https://www.flysnow.org/2017/05/12/go-in-action-go-context.html
- Context 使用原则
1. 不要把Context放在结构体中，要以参数的方式传递
2. 以Context作为参数的函数方法，应该把Context作为第一个参数，放在第一位。
3. 给一个函数方法传递Context的时候，不要传递nil，如果不知道传递什么，就使用context.TODO
4. Context的Value相关方法应该传递必须的数据，不要什么数据都使用这个传递
5. Context是线程安全的，可以放心的在多个goroutine中传递

# unicode与字符编码
Q:一个string类型的值在底层是怎样被表达的？

A:在底层，一个string类型的值是由一系列相对应的 Unicode 代码点的 UTF-8 编码值来表达的。

# string
strings.Builder类型的值（以下简称Builder值）的优势有下面的三种：

- 已存在的内容不可变，但可以拼接更多的内容；
- 减少了内存分配和内容拷贝的次数；
- 可将内容重置，可重用值。