# 类型转换

go 存在 4 种类型转换分别为：断言、强制、显式、隐式

## 断言类型转换

```go
var s = x.(T)
```

如果 x 不是 nil，且 x 可以转换成 T 类型，就会断言成功，返回 T 类型的变量 s。如果 T 不是接口类型，则要求 x 的类型就是 T，如果 T 是一个接口，要求 x 实现了 T 接口。

如果断言类型成立，则表达式返回值就是 T 类型的 x，如果断言失败就会触发 panic。

另外一种语法

```go
s, ok := x.(T)
```

 ok 会返回是否断言成功，**不会出现 panic**

# Map

## 初始化

map在go里是属于`reference type`，也就是**作为方法的型参或者返回类型的是时候，传递也是这个reference的地址，不是map的本体**。其次，这个**map在申明的时候是nil map，需要如果没有初始化，那么就是nil**

如果你这样定义一个map

```
var m map[string]int
```

就是一个nil的map，可以对其进行任意的取值，返回都是（nil，err），但是如果对其设置一个新的值，就会panic

## 线程安全

go的map不是线程安全的，因此需要加锁，一般的方法是，定义一个embeded的struct，类似于子类

```go
var counter = struct{
    sync.RWMutex
    m map[string]int
}{m: make(map[string]int)}
```

读的时候，调用读锁

```go
counter.RLock()
n := counter.m["some_key"]
counter.RUnlock()
fmt.Println("some_key:", n)
```

写的时候，写锁

```go
counter.Lock()
counter.m["some_key"]++
counter.Unlock()
```

## 查找

```go
if val, ok := m["1234"]; ok {
    fmt.Println(val)
}
```

## struct类型map结构体成员不能修改

```go
type Person struct{
    name string
    sex string
    age int
}

func main(){
    m := map[uint]Person{
        0 : Person{"张无忌", "男", 18},
        1 : Person{"周芷若", "女", 17},
    }

    m[0].age = 20  //这个会报错
```

解决方法1：

整体更新map的value值

```go
func main(){
    m := map[uint]Person{
        0 : Person{"张无忌", "男", 18},
        1 : Person{"周芷若", "女", 17},
    }
    //整体更新结构体
    temp := m[0]
    temp.age += 1
    m[0] = temp
    fmt.Println(m)
}
```

解决方法2：

把map的value部分定义为对应类型的指针类型或是slice或是map时，这样是可以更新v的内部字段的

```go
func main() {
    //定义map的value类型为指针类型
    m := map[uint]*Person{
        0: &Person{"张无忌", "男", 18},
        1: &Person{"周芷若", "女", 17},
    }
    m[0].age += 1
    fmt.Println(*m[0])
}
```

## map排序

使用 sort.Slice 排序

```go
package main

import (
	"fmt"
	"sort"
)

type kv struct {
	Key   string
	Value int
}

type podCPUUseRatio struct {
	podname        string
	podCPUUseRatio float64
}

func SortMap(pods map[string]podCPUUseRatio) []podCPUUseRatio {
	var ss []podCPUUseRatio
	for _, v := range pods {
		ss = append(ss, v)
	}

	sort.Slice(ss, func(i, j int) bool {
		return ss[i].podCPUUseRatio > ss[j].podCPUUseRatio // 降序
		// return ss[i].Value < ss[j].Value  // 升序
	})

	return ss

}

func main() {
	m := map[string]podCPUUseRatio{
		"a": {"pd1", 2.32},
		"b": {"pd2", 4.32},
		"c": {"pd3", 3.32},
	}

	sortedPods := SortMap(m)

	for _, v := range sortedPods {
		fmt.Printf("%s, %d\n", v.podname, v.podCPUUseRatio)
	}

}
```

# 引用类型和值类型

在GO中，我们判断所谓的“传值”或者“传引用”只要看被传递的值的类型就好了。 如果传递的值是引用类型的，那么就是“传引用”。如果传递的值是值类型的，那么就 是“传值”。

* slice、map 和 channel都是引用类型
* 其他是值类型

# Sync

## sync.WaitGroup

 一般用于主goroutine等待子goroutine运行结束，特别是一对多的情况

WaitGroup类型拥有三个指针方法：Add、Done和Wait。你可以想象该类型中有一个计数器，它的默认值是0。我们可以通过调用该类型值的Add方法来增加，或者减少这个计数器的值。 

一般情况下，用这个方法来记录需要等待的goroutine的数量。相对应的，这个类型的Done方法，用于 对其所属值中计数器的值进行减一操作。我们可以在需要等待的goroutine中，通过defer语句调用它。 

而此类型的Wait方法的功能是，阻塞当前的goroutine，直到其所属值中的计数器归零。如果在该方法被调用的时候，那个计数器的值就是0，那么它将不会做任何事情。

```go
func coordinateWithWaitGroup() { 
  var wg sync.WaitGroup wg.Add(2) 
  num := int32(0) 
  fmt.Printf("The number: %d [with sync.WaitGroup]\n", num) 
  max := int32(10) 
  go addNum(&num, 3, max, wg.Done) 
  go addNum(&num, 4, max, wg.Done) 
  wg.Wait() 
}
```

## sync.Once

**ync.Once** 是 Golang package 中使方法只执行一次的对象实现，作用与 **init** 函数类似。但也有所不同。

- **init** 函数是在文件包首次被加载的时候执行，且只执行一次
- **sync.Onc** 是在代码运行中需要的时候执行，且只执行一次

当一个函数不希望程序在一开始的时候就被执行的时候，我们可以使用 **sync.Once** 。

```go
package main

import (
    "fmt"
    "sync"
)

func main() {
    var once sync.Once
    onceBody := func() {
        fmt.Println("Only once")
    }
    done := make(chan bool)
    for i := 0; i < 10; i++ {
        go func() {
            once.Do(onceBody)
            done <- true
        }()
    }
    for i := 0; i < 10; i++ {
        <-done
    }
}

# Output:
Only once
```

# Context

控制并发有两种经典的方式，一种是WaitGroup，另外一种就是Context

1. 不要把Context放在结构体中，要以参数的方式传递
2. 以Context作为参数的函数方法，应该把Context作为第一个参数，放在第一位。
3. 给一个函数方法传递Context的时候，不要传递nil，如果不知道传递什么，就使用context.TODO
4. Context的Value相关方法应该传递必须的数据，不要什么数据都使用这个传递
5. Context是线程安全的，可以放心的在多个goroutine中传递

## context基本用法

```go
func main() {
    ctx, cancel := context.WithCancel(context.Background())
    go func(ctx context.Context) {
        for {
            select {
            case <-ctx.Done():
                fmt.Println("监控退出，停止了...")
                return
            default:
                fmt.Println("goroutine监控中...")
                time.Sleep(2 * time.Second)
            }
        }
    }(ctx)
    time.Sleep(10 * time.Second)
    fmt.Println("可以了，通知监控停止")
    cancel()
    //为了检测监控过是否停止，如果没有监控输出，就表示停止了
    time.Sleep(5 * time.Second)
}
```

`context.Background()` 返回一个空的Context，这个空的Context一般用于整个Context树的根节点。然后我们使用`context.WithCancel(parent)`函数，创建一个可取消的子Context，然后当作参数传给goroutine使用，这样就可以使用这个子Context跟踪这个goroutine。

在goroutine中，使用select调用`<-ctx.Done()`判断是否要结束，如果接受到值的话，就可以返回结束goroutine了；如果接收不到，就会继续进行监控。

那么是如何发送结束指令的呢？这就是上面的`cancel`函数啦，它是我们调用`context.WithCancel(parent)`函数生成子Context的时候返回的，第二个返回值就是这个取消函数，它是`CancelFunc`类型的。我们调用它就可以发出取消指令，然后我们的监控goroutine就会收到信号，就会返回结束。

## Context控制多个goroutine

同样的用法

```go
func main() {
    ctx, cancel := context.WithCancel(context.Background())
    go watch(ctx,"【监控1】")
    go watch(ctx,"【监控2】")
    go watch(ctx,"【监控3】")

    time.Sleep(10 * time.Second)
    fmt.Println("可以了，通知监控停止")
    cancel()
    //为了检测监控过是否停止，如果没有监控输出，就表示停止了
    time.Sleep(5 * time.Second)
}

func watch(ctx context.Context, name string) {
    for {
        select {
        case <-ctx.Done():
            fmt.Println(name,"监控退出，停止了...")
            return
        default:
            fmt.Println(name,"goroutine监控中...")
            time.Sleep(2 * time.Second)
        }
    }
}
```

启动了3个监控goroutine进行不断的监控，每一个都使用了Context进行跟踪，当我们使用`cancel`函数通知取消时，这3个goroutine都会被结束。这就是Context的控制能力，它就像一个控制器一样，按下开关后，所有基于这个Context或者衍生的子Context都会收到通知，这时就可以进行清理操作了，最终释放goroutine，这就优雅的解决了goroutine启动后不可控的问题。

## context 接口

```go
type Context interface {
    Deadline() (deadline time.Time, ok bool)
    Done() <-chan struct{}
    Err() error
    Value(key interface{}) interface{}
}
```

- `Deadline`方法是获取设置的截止时间的意思，第一个返回式是截止时间，到了这个时间点，Context会自动发起取消请求；第二个返回值ok==false时表示没有设置截止时间，如果需要取消的话，需要调用取消函数进行取消。

- `Done`方法返回一个只读的chan，类型为`struct{}`，我们在goroutine中，如果该方法返回的chan可以读取，则意味着parent context已经发起了取消请求，我们通过`Done`方法收到这个信号后，就应该做清理操作，然后退出goroutine，释放资源。

- `Err`方法返回取消的错误原因，因为什么Context被取消。

- `Value`方法获取该Context上绑定的值，是一个键值对，所以要通过一个Key才可以获取对应的值，这个值一般是线程安全的。

以上四个方法中常用的就是`Done`了，如果Context取消的时候，我们就可以得到一个关闭的chan，关闭的chan是可以读取的，所以只要可以读取的时候，就意味着收到Context取消的信号了，以下是这个方法的经典用法。

```go
  func Stream(ctx context.Context, out chan<- Value) error {
      for {
          v, err := DoSomething(ctx)
          if err != nil {
              return err
          }
          select {
          case <-ctx.Done():
              return ctx.Err()
          case out <- v:
          }
      }
  }
```
