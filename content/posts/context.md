---
title: "Context源码分析"
date: 2021-06-01T15:08:31+08:00
tags: ["Go", "标准库", "并发编程"]
author: zoulingbin
description: "Context是 Go 语言中非常有特色的一个特性， 
其主要的作用是在 goroutine 中进行上下文的传递，而在传递信息中又包含了 goroutine 的运行控制、上下文信息传递等功能。"

---
<!--more-->
### 什么是Context

上下文（Context）是 Go 语言中非常有特色的一个特性， 在 Go 1.7 版本中正式引入新标准库 context。

其主要的作用是在 goroutine 中进行上下文的传递，而在传递信息中又包含了 goroutine 的运行控制、上下文信息传递等功能。

![](context.png)

`context`有以下几种函数：
- WithCancel：基于父级 context，创建一个可以取消的新 context。
- WithDeadline：基于父级 context，创建一个具有截止时间（Deadline）的新 context。
- WithTimeout：基于父级 context，创建一个具有超时时间（Timeout）的新 context。
- Background：创建一个空的 context，一般常用于作为根的父级 context。
- TODO：创建一个空的 context，一般用于未确定时的声明使用。
- WithValue：基于某个 context 创建并存储对应的上下文信息。

常见用法示例：
```go
package main

func main() {
	ctx, cancel := context.WithCancel(context.Background())
	go func(ctx context.Context) {
		for  {
			select {
			case <- ctx.Done():
				fmt.Println("监控退出")
				return
			default:
				fmt.Println("监控中")
				time.Sleep(2 *time.Second)
			}
		}
	}(ctx)

	time.Sleep(10 *time.Second)
	fmt.Println("通知取消监控")
	cancel()
	//为了检测监控过是否停止，如果没有监控输出，就表示停止了
	time.Sleep(3 * time.Second)

} 
```

### 接口
1. context接口：
```go

type Context interface {

	Deadline() (deadline time.Time, ok bool)
	
	Done() <-chan struct{}

	Err() error
	
	Value(key interface{}) interface{}
}
```

context接口包含了四个方法：
- **`Deadline()(deadline time.Time, ok bool)`**

  `Deadline`方法返回结果有两个，第一个是截止时间，到了这个截止时间，Context 会自动取消；第二个是一个`bool`类型的值，如果`Context` 没有设置截止时间，第二个返回结果是`false`，如果需要取消这个 `Context`，就需要调用取消函数。

- **`Done() <-chan struct{}`**

  `Done`方法返回一个只读的`channel`对象，类型是`struct{}`,在`goroutine`中，如果`Done`方法返回的结果可以被读取，代表父`Context`调用了取消函数。

- **`Err() error`**

  Err 方法返回 Context 被取消的原因。

- **`Value(key interface{}) interface{}`**

  Value 方法返回此 Context 绑定的值。它是一个 kv 键值对，通过 key 获取对应 value 的值。

2. Canceler接口：
 ```go
 type canceler interface {
    cancel(removeFromParent bool, err error)
    Done() <-chan struct{}
 }
 ```
- cancel:调用当前 context 的取消方法。
- Done：与前面一致，可用于识别当前 channel 是否已经被关闭。

### 接口体
在标准库`context`的设计上，一共提供了四类`context`类型来实现上述接口。分别是`emptyCtx`、`cancelCtx`、`timerCtx`以及`valueCtx`。

#### emptyCtx
源码中定义了 Context 接口后，并且给出了一个实现：
```go
type emptyCtx int

func (*emptyCtx) Deadline() (deadline time.Time, ok bool) {
	return
}

func (*emptyCtx) Done() <-chan struct{} {
	return nil
}

func (*emptyCtx) Err() error {
	return nil
}

func (*emptyCtx) Value(key interface{}) interface{} {
	return nil
}
```
> **Backgroud()和TODO()**:

使用`context.Backgroud`或者`Context.TODO`生成一个空的`context`定义，其实就是对`emptyCtx`的封装。

`background`通常用在`main`函数中，作为所有`context`的根节点。

`todo`通常用在并不知道传递什么`context`的情形。例如，调用一个需要传递`context`参数的函数，你手头并没有其他 context 可以传递，这时就可以传递`todo`。这常常发生在重构进行中，给一些函数添加了一个 Context 参数，但不知道要传什么，就用`todo`“占个位子”，最终要换成其他`context`。

```go
var (
    backgroud = new(emptyCtx)
    todo = new(emptyCtx)
)

func Backgroud() Context {
    return backgroud
}

func TODO() Context {
    return todo
}
```

#### CancelCtx
通过`WithCancel`创建一个可取消的`context`方法:
```go
func WithCancel(parent Context) (ctx Context, cancel CancelFunc) {
	c := newCancelCtx(parent)
	propagateCancel(parent, &c)
	return &c, func() { c.cancel(true, Canceled) }
}
```
传入一个父`Context`（这通常是一个 background，作为根节点），返回新建的`context`，新`context`的`done` ` channel`是新建的。
```go
type cancelCtx struct {
	Context

	mu       sync.Mutex            // protects following fields
	done     chan struct{}         // created lazily, closed by first cancel call
	children map[canceler]struct{} // set to nil by the first cancel call 包含了该context对应的所有子context，发生取消事件时逐一通知。
	err      error                 // set to non-nil by the first cancel call
}
```

> `cancelCtx`的`Done`方法的实现：

```go
func (c *cancelCtx) Done() <-chan struct{} {
	c.mu.Lock()
	if c.done == nil {
		c.done = make(chan struct{})
	}
	d := c.done
	c.mu.Unlock()
	return d
}
```
`c.done`是“懒汉式”创建，只有调用了`Done()`方法的时候才会被创建。函数返回一个只读的`channel`,并没有任何地方往这个channel写入值，所以，直接调用读这个 channel，协程会被`block`住。一般通过搭配`select`来使用。一旦关闭，就会立即读出零值。

> `cancelCtx`的`cancel`方法：

```go
// closedchan is a reusable closed channel.
var closedchan = make(chan struct{})

func init() {
	close(closedchan)
}

/**
cancel closes c.done, cancels each of c's children, and, if
removeFromParent is true, removes c from its parent's children.
*/
func (c *cancelCtx) cancel(removeFromParent bool, err error) {
    if err == nil {
		panic("context: internal error: missing cancel error")
	}
	c.mu.Lock()
	if c.err != nil {
		c.mu.Unlock()
		return // already canceled
	}
	c.err = err
	if c.done == nil {
		c.done = closedchan
	} else {
		close(c.done)
	}
	for child := range c.children {
		// NOTE: acquiring the child's lock while holding parent's lock.
		child.cancel(false, err)   
	}
	
    // 将子节点置空
	c.children = nil
	c.mu.Unlock()
	if removeFromParent {
	    // 从父节点中移除自己
		removeChild(c.Context, c)
	}
}
```
`cancel()`方法关闭channel：c.clone;for循环取消所有的子节点，然后从父节点删除自己。关闭c.done,所有子节点收到取消信号。(`case <- c.Done()`)

> **WithCancel()**:

```go
func WithCancel(parent Context) (ctx Context, cancel CancelFunc) {
	c := newCancelCtx(parent)
	propagateCancel(parent, &c)
	return &c, func() { c.cancel(true, Canceled) }
}

// newCancelCtx returns an initialized cancelCtx.
func newCancelCtx(parent Context) cancelCtx {
	return cancelCtx{Context: parent}
}
```
传入一个父`Context`（这通常是一个 background，作为根节点），返回新建的`context`，新`context`的`<-done`是新建的。

当`WithCancel`函数返回的`CancelFUnc`被调用或者父节点的`<-done`被关闭，此context的`<-done`也会被关闭。`newCancelCtx()`方法将会生成出一个可以取消的新 `context`，如果该`context`执行取消，与其相关联的子`context`以及对应的`goroutine`也会收到取消信息。

当`removeFromParent`为`true`时，会将当前节点的`context`从父节点`context`中删除(调用WithCancel方法时，`return &c, func() { c.cancel(true, Canceled) }`):
```go
func removeChild(parent Context, child canceler) {
	p, ok := parentCancelCtx(parent)
	if !ok {
		return
	}
	p.mu.Lock()
	if p.children != nil {
		delete(p.children, child)
	}
	p.mu.Unlock()
}
```
当调用返回的`cancelFunc`时，会将这个`context`从它的父节点里删除，因为父节点可能有很多子节点，但仅该`context`删除，对其他子context没影响。
![WithCancel](https://note.youdao.com/yws/api/personal/file/WEB65efffd9e27ed3942e4fa0cafc31fb9e?method=download&shareKey=6c16cdae55adc8360b72199510c448c9)

#### timerCtx
`timerCtx`基于`cancelCtx`，只是多了一个`time.Timer` 和一个`deadline`。`Timer`会在`deadline`到来时，自动取消 context。
```go
// A timerCtx carries a timer and a deadline. It embeds a cancelCtx to
// implement Done and Err. It implements cancel by stopping its timer then
// delegating to cancelCtx.cancel.
type timerCtx struct {
	cancelCtx
	timer *time.Timer // Under cancelCtx.mu.

	deadline time.Time
}
```
从源码可以看到`timerCtx`是基于`cancelCtx`,只是多了一个 `time.Timer`和一个`deadline`。Timer 会在`deadline` 到来时，自动取消`context`。

> `timerCtx`的`cancel()`方法

```go
func (c *timerCtx) cancel(removeFromParent bool, err error) {
	c.cancelCtx.cancel(false, err)
	if removeFromParent {
		// Remove this timerCtx from its parent cancelCtx's children.
		removeChild(c.cancelCtx.Context, c)
	}
	c.mu.Lock()
	if c.timer != nil {
		c.timer.Stop()
		c.timer = nil
	}
	c.mu.Unlock()
}
```

> **WithTimeout()**

```go
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc) {
	return WithDeadline(parent, time.Now().Add(timeout))
}
```
`WithTimeout`函数调用了`WithDeadline`，传入的`deadline`是当前时间加上`timeout`的时间，也就是从现在开始再经过 `timeout`时间就算超时。也就是说， `WithDeadline` 需要用的是绝对时间。

> **WithDeadline()**

```go
func WithDeadline(parent Context, d time.Time) (Context, CancelFunc) {
	if cur, ok := parent.Deadline(); ok && cur.Before(d) {
		// 如果父节点 context 的 deadline 早于指定时间。直接构建一个可取消的 context。
        // 原因是一旦父节点超时，自动调用 cancel 函数，子节点也会随之取消。
        // 所以不用单独处理子节点的计时器时间到了之后，自动调用 cancel 函数
		return WithCancel(parent)
	}
	c := &timerCtx{
		cancelCtx: newCancelCtx(parent),
		deadline:  d,
	}
	propagateCancel(parent, c)
	dur := time.Until(d)
	if dur <= 0 {
		c.cancel(true, DeadlineExceeded) // deadline has already passed
		return c, func() { c.cancel(false, Canceled) }
	}
	c.mu.Lock()
	defer c.mu.Unlock()
	if c.err == nil {
	    //dur时间后，timer会自动调用cancel执行取消
		c.timer = time.AfterFunc(dur, func() {
			c.cancel(true, DeadlineExceeded)
		})
	}
	return c, func() { c.cancel(true, Canceled) }
}
```
也就是说仍然要把子节点挂靠到父节点，一旦父节点取消了，会把取消信号向下传递到子节点，子节点随之取消。

有一个特殊情况是，如果要创建的这个子节点的`deadline` 比父节点要晚，也就是说如果父节点是时间到自动取消，那么一定会取消这个子节点，导致子节点的`deadline` 根本不起作用，因为子节点在`deadline` 到来之前就已经被父节点取消了。

#### valueCtx
```go
type valueCtx struct{
    Context
    key, val interface{}
}
```
`valueCtx`主要用于上下文的信息传递，核心就是键值对。

`valueCtx`实现了两个方法：
```go
func (c *valueCtx) String() string {
	return contextName(c.Context) + ".WithValue(type " +
		reflectlite.TypeOf(c.key).String() +
		", val " + stringify(c.val) + ")"
}
//key的递归取值过程
func (c *valueCtx) Value(key interface{}) interface{} {
	if c.key == key {
		return c.val
	}
	return c.Context.Value(key)
}
```

> **WithValue**

`WithValue`创建`context`节点的过程实际上就是创建链表节点的过程。
```go
func WithValue(parent Context, key, val interface{}) Context {
	if key == nil {
		panic("nil key")
	}
	if !reflectlite.TypeOf(key).Comparable() {
		panic("key is not comparable")
	}
	return &valueCtx{parent, key, val}
}
```
对`key`的要求是可比较，因为之后需要通过 key 取出 context 中的值，可比较是必须的，然后就是存储匹配。

![context](https://note.youdao.com/yws/api/personal/file/WEBe7111b18412a97f106f896be043b03bf?method=download&shareKey=b0b5c7723ef70fb4ba578983631cdbb6)


### context取消事件

在`WithCancel()`和`WithDeadline()`里，都调用了`propagateCancel()`方法：
```go
/**
当父级上下文（parent）的 Done 结果为 nil 时，将会直接返回，因为其不会具备取消事件的基本条件，可能该 context 是 Background、TODO 等方法产生的空白 context。
当父级上下文（parent）的 Done 结果不为 nil 时，则发现父级上下文已经被取消，作为其子级，该 context 将会触发取消事件并返回父级上下文的取消原因。
*/
func propagateCancel(parent Context, child canceler) {
	done := parent.Done()
	if done == nil {
		return // parent is never canceled
	}

	select {
	case <-done:
		// parent is already canceled
		child.cancel(false, parent.Err())
		return
	default:
	}

	if p, ok := parentCancelCtx(parent); ok {
		p.mu.Lock()
		if p.err != nil {
			// parent has already been canceled
			child.cancel(false, p.err)
		} else {
			if p.children == nil {
				p.children = make(map[canceler]struct{})
			}
			p.children[child] = struct{}{}
		}
		p.mu.Unlock()
	} else {
		atomic.AddInt32(&goroutines, +1)
		go func() {
			select {
			case <-parent.Done():
				child.cancel(false, parent.Err())
			case <-child.Done():
			}
		}()
	}
}


// &cancelCtxKey is the key that a cancelCtx returns itself for.
var cancelCtxKey int

func parentCancelCtx(parent Context) (*cancelCtx, bool) {
	done := parent.Done()
	if done == closedchan || done == nil {
		return nil, false
	}
	p, ok := parent.Value(&cancelCtxKey).(*cancelCtx)
	if !ok {
		return nil, false
	}
	p.mu.Lock()
	ok = p.done == done
	p.mu.Unlock()
	if !ok {
		return nil, false
	}
	return p, true
}
```

`WithCancel`和`WithDeadline`都会涉及到`propagateCancel()` 方法，其作用是构建父子级的上下文的关联关系，若出现取消事件时，就会进行处理。



### Reference
https://mp.weixin.qq.com/s/GpVy1eB5Cz_t-dhVC6BJNw

https://mp.weixin.qq.com/s/uz5JqhRFSoZaEgjebFm0WA

https://mp.weixin.qq.com/s/A03G3_kCvVFN3TxB-92GVw

https://zhuanlan.zhihu.com/p/110085652

