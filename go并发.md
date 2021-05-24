##  Mutex

1. 互斥锁是并发控制的一个基本手段，是为了避免竞争而建立的一种并发控制机制。

2. 临界区就是一个被共享的资源，或者说是一个整体的一组共享资源

3. 使用互斥锁，限定临界区只能同时由一个线程持有。

4. 同步原语的适用场景：

   - 共享资源。并发地读写共享资源，会出现数据竞争（data race）的问题，所以需要 Mutex、RWMutex 这样的并发原语来保护
   - 任务编排。需要 goroutine 按照一定的规律执行，而 goroutine 之间有相互等待或者依赖的顺序关系，我们常常使用 WaitGroup 或者 Channel 来实现
   - 消息传递。信息交流以及不同的 goroutine 之间的线程安全的数据交流，常常使用 Channel 来实现。

5. 互斥锁 Mutex 就提供两个方法 Lock 和 Unlock：进入临界区之前调用 Lock 方法，退出临界区的时候调用 Unlock 方法

6. ##### Mutex 的架构演进分成了四个阶段

   1. “初版”的 Mutex 使用一个 flag 来表示锁是否被持有，实现比较简单；
   2. “给新人机会”后来照顾到新来的 goroutine，所以会让新的 goroutine 也尽可能地先获取到锁
   3. “多给些机会”，照顾新来的和被唤醒的 goroutine
   4. “解决饥饿”目前又加入了饥饿的解决方案

   ![img](C:\Users\king\Desktop\go并发\c28531b47ff7f220d5bc3c9650180835.jpg)

7. CAS 指令将给定的值和一个内存地址中的值进行比较，如果它们是同一个值，就使用新值替换内存地址中的值，这个操作是原子性的。那啥是原子性呢？如果你还不太理解这个概念，那么在这里只需要明确一点就行了，那就是原子性保证这个指令总是基于最新的值进行计算，如果同时有其它线程已经修改了这个值，那么，CAS 会返回失败。

8. ![img](C:\Users\king\Desktop\go并发\825e23e1af96e78f3773e0b45de38e25.jpg)

9. Unlock 方法可以被任意的 goroutine 调用释放锁，即使是没持有这个互斥锁的 goroutine，也可以进行这个操作。这是因为，Mutex 本身并没有包含持有这把锁的 goroutine 的信息，所以，Unlock 也不会对此进行检查。Mutex 的这个设计一直保持至今。

10. 如果临界区只是方法中的一部分，为了尽快释放锁，还是应该第一时间调用 Unlock，而不是一直等到方法返回时才释放。

11. ![img](C:\Users\king\Desktop\go并发\4c4a3dd2310059821f41af7b84925615.jpg)

12. 因为新来的 goroutine 也参与竞争，有可能每次都会被新来的 goroutine 抢到获取锁的机会，在极端情况下，等待中的 goroutine 可能会一直获取不到锁，这就是饥饿问题。

13. ![img](C:\Users\king\Desktop\go并发\e0c23794c8a1d355a7a183400c036276.jpg)

14. 只需要记住，Mutex 绝不容忍一个 goroutine 被落下，永远没有机会获取锁。不抛弃不放弃是它的宗旨，而且它也尽可能地让等待较长的 goroutine 更有机会获取到锁。

15. ##### 饥饿模式和正常模式

    ​	Mutex 可能处于两种操作模式下：正常模式和饥饿模式。接下来我们分析一下 Mutex 对饥饿模式和正常模式的处理。请求锁时调用的 Lock 方法中一开始是 fast path，这是一个幸运的场景，当前的 goroutine 幸运地获得了锁，没有竞争，直接返回，否则就进入了 lockSlow 方法。这样的设计，方便编译器对 Lock 方法进行内联，你也可以在程序开发中应用这个技巧。正常模式下，waiter 都是进入先入先出队列，被唤醒的 waiter 并不会直接持有锁，而是要和新来的 goroutine 进行竞争。新来的 goroutine 有先天的优势，它们正在 CPU 中运行，可能它们的数量还不少，所以，在高并发情况下，被唤醒的 waiter 可能比较悲剧地获取不到锁，这时，它会被插入到队列的前面。如果 waiter 获取不到锁的时间超过阈值 1 毫秒，那么，这个 Mutex 就进入到了饥饿模式。在饥饿模式下，Mutex 的拥有者将直接把锁交给队列最前面的 waiter。新来的 goroutine 不会尝试获取锁，即使看起来锁没有被持有，它也不会去抢，也不会 spin，它会乖乖地加入到等待队列的尾部。如果拥有 Mutex 的 waiter 发现下面两种情况的其中之一，它就会把这个 Mutex 转换成正常模式:此 waiter 已经是队列中的最后一个 waiter 了，没有其它的等待锁的 goroutine 了；此 waiter 的等待时间小于 1 毫秒。正常模式拥有更好的性能，因为即使有等待抢锁的 waiter，goroutine 也可以连续多次获取到锁。饥饿模式是对公平性和性能的一种平衡，它避免了某些 goroutine 长时间的等待锁。在饥饿模式下，优先对待的是那些一直在等待的 waiter。

16. Package sync 的同步原语在使用后是不能复制的。我们知道 Mutex 是最常用的一个同步原语，那它也是不能复制的。

17. mutex常见的 4 种错误场景

    - Lock/Unlock 不是成对出现

    - Copy 已使用的 Mutex

    - 重入

      ```
      
      func foo(l sync.Locker) {
          fmt.Println("in foo")
          l.Lock()
          bar(l)
          l.Unlock()
      }
      
      
      func bar(l sync.Locker) {
          l.Lock()
          fmt.Println("in bar")
          l.Unlock()
      }
      
      
      func main() {
          l := &sync.Mutex{}
          foo(l)
      }
      ```

    - 死锁
    
18. 锁是性能下降的“罪魁祸首”之一，所以，有效地降低锁的竞争，就能够很好地提高性能。因此，监控关键互斥锁上等待的 goroutine 的数量，是我们分析锁竞争的激烈程度的一个重要指标

### TryLock

当一个 goroutine 调用这个 TryLock 方法请求锁的时候，如果这把锁没有被其他 goroutine 所持有，那么，这个 goroutine 就持有了这把锁，并返回 true；如果这把锁已经被其他 goroutine 所持有，或者是正在准备交给某个被唤醒的 goroutine，那么，这个请求锁的 goroutine 就直接返回 false，不会阻塞在方法调用上。

![img](C:\Users\king\Desktop\go并发\e7787d959b60d66cc3a46ee921098865.jpg)

```

// 复制Mutex定义的常量
const (
    mutexLocked = 1 << iota // 加锁标识位置
    mutexWoken              // 唤醒标识位置
    mutexStarving           // 锁饥饿标识位置
    mutexWaiterShift = iota // 标识waiter的起始bit位置
)

// 扩展一个Mutex结构
type Mutex struct {
    sync.Mutex
}

// 尝试获取锁
func (m *Mutex) TryLock() bool {
    // 如果能成功抢到锁
    if atomic.CompareAndSwapInt32((*int32)(unsafe.Pointer(&m.Mutex)), 0, mutexLocked) {
        return true
    }

    // 如果处于唤醒、加锁或者饥饿状态，这次请求就不参与竞争了，返回false
    old := atomic.LoadInt32((*int32)(unsafe.Pointer(&m.Mutex)))
    if old&(mutexLocked|mutexStarving|mutexWoken) != 0 {
        return false
    }

    // 尝试在竞争的状态下请求锁
    new := old | mutexLocked
    return atomic.CompareAndSwapInt32((*int32)(unsafe.Pointer(&m.Mutex)), old, new)
}
```

#### 获取等待者的数量等指标

```

const (
    mutexLocked = 1 << iota // mutex is locked
    mutexWoken
    mutexStarving
    mutexWaiterShift = iota
)

type Mutex struct {
    sync.Mutex
}

func (m *Mutex) Count() int {
    // 获取state字段的值
    v := atomic.LoadInt32((*int32)(unsafe.Pointer(&m.Mutex)))
    v = v >> mutexWaiterShift //得到等待者的数值
    v = v + (v & mutexLocked) //再加上锁持有者的数量，0或者1
    return int(v)
}
```

#### 使用 Mutex 实现一个线程安全的队列

```

type SliceQueue struct {
    data []interface{}
    mu   sync.Mutex
}

func NewSliceQueue(n int) (q *SliceQueue) {
    return &SliceQueue{data: make([]interface{}, 0, n)}
}

// Enqueue 把值放在队尾
func (q *SliceQueue) Enqueue(v interface{}) {
    q.mu.Lock()
    q.data = append(q.data, v)
    q.mu.Unlock()
}

// Dequeue 移去队头并返回
func (q *SliceQueue) Dequeue() interface{} {
    q.mu.Lock()
    if len(q.data) == 0 {
        q.mu.Unlock()
        return nil
    }
    v := q.data[0]
    q.data = q.data[1:]
    q.mu.Unlock()
    return v
}
```

## RWMutex

RWMutex 在某一时刻只能由任意数量的 reader 持有，或者是只被单个的 writer 持有。

- **Lock/Unlock：写操作时调用的方法。**如果锁已经被 reader 或者 writer 持有，那么，Lock 方法会一直阻塞，直到能获取到锁；Unlock 则是配对的释放锁的方法。
- **RLock/RUnlock：读操作时调用的方法。**如果锁已经被 writer 持有的话，RLock 方法会一直阻塞，直到能获取到锁，否则就直接返回；而 RUnlock 是 reader 释放锁的方法。
- **RLocker：这个方法的作用是为读操作返回一个 Locker 接口的对象。**它的 Lock 方法会调用 RWMutex 的 RLock 方法，它的 Unlock 方法会调用 RWMutex 的 RUnlock 方法。

readers-writers 问题一般有三类，基于对读和写操作的优先级，读写锁的设计和实现也分成三类。

- ​	**Read-preferring：**读优先的设计可以提供很高的并发性，但是，在竞争激烈的情况下可能会导致写饥饿。这是因为，如果有大量的读，这种设计会导致只有所有的读都释放了锁之后，写才可能获取到锁。
- **Write-preferring：**写优先的设计意味着，如果已经有一个 writer 在等待请求锁的话，它会阻止新来的请求锁的 reader 获取到锁，所以优先保障 writer。当然，如果有一些 reader 已经请求了锁的话，新请求的 writer 也会等待已经存在的 reader 都释放锁之后才能获取。所以，写优先级设计中的优先权是针对新来的请求而言的。这种设计主要避免了 writer 的饥饿问题。
- **不指定优先级：**这种设计比较简单，不区分 reader 和 writer 优先级，某些场景下这种不指定优先级的设计反而更有效，因为第一类优先级会导致写饥饿，第二类优先级可能会导致读饥饿，这种不指定优先级的访问不再区分读写，大家都是同一个优先级，解决了饥饿的问题。

#### RWMutex 的 3 个踩坑点

	1. 不可复制
 	2. 重入导致死锁
 	3. 释放未加锁的 RWMutex



## WaitGroup

## Cond

Cond 通常应用于等待某个条件的一组 goroutine，等条件变为 true 的时候，其中一个 goroutine 或者所有的 goroutine 都会被唤醒执行。

使用 Cond 常见的两个错误，一个是调用 Wait 的时候没有加锁，另一个是没有检查条件是否满足程序就继续执行了。

在实践中，处理等待 / 通知的场景时，我们常常会使用 Channel 替换 Cond，因为 Channel 类型使用起来更简洁，而且不容易出错。但是对于需要重复调用 Broadcast 的场景，比如上面 Kubernetes 的例子，每次往队列中成功增加了元素后就需要调用 Broadcast 通知所有的等待者，使用 Cond 就再合适不过了。

## Once

Once 可以用来执行且仅仅执行一次动作，常常用于单例对象的初始化场景。

sync.Once 只暴露了一个方法 Do，你可以多次调用 Do 方法，但是只有第一次调用 Do 方法时 f 参数才会执行，这里的 f 是一个无参数无返回值的函数。

在实际的使用中，绝大多数情况下，你会使用闭包的方式去初始化外部的一个资源。

Once 常常用来初始化单例资源，或者并发访问只需初始化一次的共享资源，或者在测试的时候初始化一次测试资源。

**自己实现一个类似 Once 的并发原语，既可以返回当前调用 Do 方法是否正确完成，还可以在初始化失败后调用 Do 方法再次尝试初始化，直到初始化成功才不再初始化了。**

```

// 一个功能更加强大的Once
type Once struct {
    m    sync.Mutex
    done uint32
}
// 传入的函数f有返回值error，如果初始化失败，需要返回失败的error
// Do方法会把这个error返回给调用者
func (o *Once) Do(f func() error) error {
    if atomic.LoadUint32(&o.done) == 1 { //fast path
        return nil
    }
    return o.slowDo(f)
}
// 如果还没有初始化
func (o *Once) slowDo(f func() error) error {
    o.m.Lock()
    defer o.m.Unlock()
    var err error
    if o.done == 0 { // 双检查，还没有初始化
        err = f()
        if err == nil { // 初始化成功才将标记置为已初始化
            atomic.StoreUint32(&o.done, 1)
        }
    }
    return err
}
```

## 线程安全的map类型

1. 使用 map 的 2 种常见错误:

   未初始化和并发读写

2. 加读写锁：扩展 map，支持并发读写

3. 分片加锁：更高效的并发 map。减少锁的粒度常用的方法就是分片（Shard），将一把锁分成几把锁，每个锁控制一个分片。GetShard 是一个关键的方法，能够根据 key 计算出分片索引。

**应对特殊场景的 sync.Map**

只会增长的缓存系统中，一个 key 只写入一次而被读很多次；

多个 goroutine 为不相交的键集读、写和重写键值对。

## sync.Pool

sync.Pool 数据类型用来保存一组可独立访问的临时对象。请注意这里加粗的“临时”这两个字，它说明了 sync.Pool 这个数据类型的特点，也就是说，它池化的对象会在未来的某个时候被毫无预兆地移除掉。而且，如果没有别的对象引用这个被移除的对象的话，这个被移除的对象就会被垃圾回收掉。

1. sync.Pool 本身就是线程安全的，多个 goroutine 可以并发地调用它的方法存取对象；
2. sync.Pool 不可在使用之后再复制使用。

#### sync.Pool 的使用方法

1. New

   Pool struct 包含一个 New 字段，这个字段的类型是函数 func() interface{}。当调用 Pool 的 Get 方法从池中获取元素，没有更多的空闲元素可返回时，就会调用这个 New 方法来创建新的元素。如果你没有设置 New 字段，没有更多的空闲元素可返回时，Get 方法将返回 nil，表明当前没有可用的元素。

2. Get

   如果调用这个方法，就会从 Pool取走一个元素，这也就意味着，这个元素会从 Pool 中移除，返回给调用者。不过，除了返回值是正常实例化的元素，Get 方法的返回值还可能会是一个 nil（Pool.New 字段没有设置，又没有空闲元素可以返回），所以你在使用的时候，可能需要判断。

3. Put

   这个方法用于将一个元素返还给 Pool，Pool 会把这个元素保存到池中，并且可以复用。但如果 Put 一个 nil 值，Pool 就会忽略这个值。

## Context：信息穿透上下文

上下文信息传递 （request-scoped），比如处理 http 请求、在请求处理链路上传递信息；

控制子 goroutine 的运行；

超时控制的方法调用；

可以取消的方法调用。

1. Deadline 方法会返回这个 Context 被取消的截止日期。如果没有设置截止日期，ok 的值是 false。后续每次调用这个对象的 Deadline 方法时，都会返回和第一次调用相同的结果。

2. Done 方法返回一个 Channel 对象。在 Context 被取消时，此 Channel 会被 close，如果没被取消，可能会返回 nil。后续的 Done 调用总是返回相同的结果。当 Done 被 close 的时候，你可以通过 ctx.Err 获取错误信息。Done 这个方法名其实起得并不好，因为名字太过笼统，不能明确反映 Done 被 close 的原因，因为 cancel、timeout、deadline 都可能导致 Done 被 close，不过，目前还没有一个更合适的方法名称。如果 Done 没有被 close，Err 方法返回 nil；如果 Done 被 close，Err 方法会返回 Done 被 close 的原因。

   

3. Value 返回此 ctx 中和指定的 key 相关联的 value。

context.Background()：返回一个非 nil 的、空的 Context，没有任何值，不会被 cancel，不会超时，没有截止日期。一般用在主函数、初始化、测试以及创建根 Context 的时候。

context.TODO()：返回一个非 nil 的、空的 Context，没有任何值，不会被 cancel，不会超时，没有截止日期。当你不清楚是否该用 Context，或者目前还不知道要传递一些什么上下文信息的时候，就可以使用这个方法。

在使用 Context 的时候，有一些约定俗成的规则：

1. 一般函数使用 Context 的时候，会把这个参数放在第一个参数的位置。
2. 从来不把 nil 当做 Context 类型的参数值，可以使用 context.Background() 创建一个空的上下文对象，也不要使用 nil。
3. Context 只用来临时做函数之间的上下文透传，不能持久化 Context 或者把 Context 长久保存。把 Context 持久化到数据库、本地文件或者全局变量、缓存中都是错误的用法。
4. key 的类型不应该是字符串类型或者其它内建类型，否则容易在包之间使用 Context 时候产生冲突。使用 WithValue 时，key 的类型应该是自己定义的类型。
5. 常常使用 struct{}作为底层类型定义 key 的类型。对于 exported key 的静态类型，常常是接口或者指针。这样可以尽量减少内存分配。

**创建特殊用途 Context 的方法**

1. WithValue 基于 parent Context 生成一个新的 Context，保存了一个 key-value 键值对。它常常用来传递上下文。
2. WithCancel 方法返回 parent 的副本，只是副本中的 Done Channel 是新建的对象，它的类型是 cancelCtx。
3. WithTimeout 其实是和 WithDeadline 一样，只不过一个参数是超时时间，一个参数是截止时间。超时时间加上当前时间，其实就是截止时间，
4. WithDeadline 会返回一个 parent 的副本，并且设置了一个不晚于参数 d 的截止时间，类型为 timerCtx（或者是 cancelCtx）。

atomic 操作的对象是一个地址，你需要把可寻址的变量的地址作为参数传递给方法，而不是把变量的值传递给方法。

1. Add

你可以利用计算机补码的规则，把减法变成加法

```

AddUint32(&x, ^uint32(c-1)).
```



2. CAS （CompareAndSwap）这个方法会比较当前 addr 地址里的值是不是 old，如果不等于 old，就返回 false；如果等于 old，就把此地址的值替换成 new 值，返回 true。这就相当于“判断相等才替换”。
3. Swap如果不需要比较旧值，只是比较粗暴地替换的话，就可以使用 Swap 方法，它替换后还可以返回旧值
4. LoadLoad 方法会取出 addr 地址中的值，即使在多处理器、多核、有 CPU cache 的情况下，这个操作也能保证 Load 是一个原子操作。
5. Store 方法会把一个值存入到指定的 addr 地址中，即使在多处理器、多核、有 CPU cache 的情况下，这个操作也能保证 Store 是一个原子操作。别的 goroutine 通过 Load 读取出来，不会看到存取了一半的值。

## channel

执行业务处理的 goroutine 不要通过共享内存的方式通信，而是要通过 Channel 通信的方式分享数据。

**Channel 的应用场景分为五种类型**

1. 数据交流：当作并发的 buffer 或者 queue，解决生产者 - 消费者问题。多个 goroutine 可以并发当作生产者（Producer）和消费者（Consumer）。
2. 数据传递：一个 goroutine 将数据交给另一个 goroutine，相当于把数据的拥有权 (引用) 托付出去。
3. 信号通知：一个 goroutine 可以将信号 (closing、closed、data ready 等) 传递给另一个或者另一组 goroutine 。
4. 任务编排：可以让一组 goroutine 按照一定的顺序并发或者串行的执行，这就是编排的功能。
5. 锁：利用 Channel 也可以实现互斥锁的机制。

通过 make，我们可以初始化一个 chan，未初始化的 chan 的零值是 nil。你可以设置它的容量，比如下面的 chan 的容量是 9527，我们把这样的 chan 叫做 buffered chan；如果没有设置，它的容量是 0，我们把这样的 chan 叫做 unbuffered chan。

nil 是 chan 的零值，是一种特殊的 chan，对值是 nil 的 chan 的发送接收调用者总是会阻塞。



1. 发送数据

```

ch <- 2000

```

2. 接收数据

   ```
   
     x := <-ch // 把接收的一条数据赋值给变量x
     foo(<-ch) // 把接收的一个的数据作为参数传给函数
     <-ch // 丢弃接收的一条数据
   ```

   接收数据时，还可以返回两个值。第一个值是返回的 chan 中的元素，很多人不太熟悉的是第二个值。第二个值是 bool 类型，代表是否成功地从 chan 中读取到一个值，如果第二个参数是 false，chan 已经被 close 而且 chan 中没有缓存的数据，这个时候，第一个值是零值。所以，如果从 chan 读取到一个零值，可能是 sender 真正发送的零值，也可能是 closed 的并且没有缓存元素产生的零值。

3. 其它操作

   ```
   
   func main() {
       var ch = make(chan int, 10)
       for i := 0; i < 10; i++ {
           select {
           case ch <- i:
           case v := <-ch:
               fmt.Println(v)
           }
       }
   }
   
   
   
   
   
       for v := range ch {
           fmt.Println(v)
       }
       
       
       
       忽略读取的值，只是清空 chan：
       
       for range ch {
       }
   
   ```

   

   1. 共享资源的并发访问使用传统并发原语；
   2. 复杂的任务编排和消息传递使用 Channel；
   3. 消息通知机制使用 Channel，除非只想 signal 一个 goroutine，才使用 Cond；
   4. 简单等待所有任务的完成用 WaitGroup，也有 Channel 的推崇者用 Channel，都可以；
   5. 需要和 Select 语句结合，使用 Channel；
   6. 需要和超时配合时，使用 Channel 和 Context。

   

   ![img](C:\Users\king\Desktop\go并发\分布式\5108954ea36559860e5e5aaa42b2f998.jpg)

   

### 使用反射操作 Channel

```

func main() {
    var ch1 = make(chan int, 10)
    var ch2 = make(chan int, 10)

    // 创建SelectCase
    var cases = createCases(ch1, ch2)

    // 执行10次select
    for i := 0; i < 10; i++ {
        chosen, recv, ok := reflect.Select(cases)
        if recv.IsValid() { // recv case
            fmt.Println("recv:", cases[chosen].Dir, recv, ok)
        } else { // send case
            fmt.Println("send:", cases[chosen].Dir, ok)
        }
    }
}

func createCases(chs ...chan int) []reflect.SelectCase {
    var cases []reflect.SelectCase


    // 创建recv case
    for _, ch := range chs {
        cases = append(cases, reflect.SelectCase{
            Dir:  reflect.SelectRecv,
            Chan: reflect.ValueOf(ch),
        })
    }

    // 创建send case
    for i, ch := range chs {
        v := reflect.ValueOf(i)
        cases = append(cases, reflect.SelectCase{
            Dir:  reflect.SelectSend,
            Chan: reflect.ValueOf(ch),
            Send: v,
        })
    }

    return cases
}
```

### 消息交流

1. 第一个例子是 worker 池的例子。

### 数据传递

```

type Token struct{}

func newWorker(id int, ch chan Token, nextCh chan Token) {
    for {
        token := <-ch         // 取得令牌
        fmt.Println((id + 1)) // id从1开始
        time.Sleep(time.Second)
        nextCh <- token
    }
}
func main() {
    chs := []chan Token{make(chan Token), make(chan Token), make(chan Token), make(chan Token)}

    // 创建4个worker
    for i := 0; i < 4; i++ {
        go newWorker(i, chs[i], chs[(i+1)%4])
    }

    //首先把令牌交给第一个worker
    chs[0] <- struct{}{}
  
    select {}
}
```

### 信号通知

```

func main() {
    var closing = make(chan struct{})
    var closed = make(chan struct{})

    go func() {
        // 模拟业务处理
        for {
            select {
            case <-closing:
                return
            default:
                // ....... 业务计算
                time.Sleep(100 * time.Millisecond)
            }
        }
    }()

    // 处理CTRL+C等中断信号
    termChan := make(chan os.Signal)
    signal.Notify(termChan, syscall.SIGINT, syscall.SIGTERM)
    <-termChan

    close(closing)
    // 执行退出之前的清理动作
    go doCleanup(closed)

    select {
    case <-closed:
    case <-time.After(time.Second):
        fmt.Println("清理超时，不等了")
    }
    fmt.Println("优雅退出")
}

func doCleanup(closed chan struct{}) {
    time.Sleep((time.Minute))
    close(closed)
}
```

### 锁

```

// 使用chan实现互斥锁
type Mutex struct {
    ch chan struct{}
}

// 使用锁需要初始化
func NewMutex() *Mutex {
    mu := &Mutex{make(chan struct{}, 1)}
    mu.ch <- struct{}{}
    return mu
}

// 请求锁，直到获取到
func (m *Mutex) Lock() {
    <-m.ch
}

// 解锁
func (m *Mutex) Unlock() {
    select {
    case m.ch <- struct{}{}:
    default:
        panic("unlock of unlocked mutex")
    }
}

// 尝试获取锁
func (m *Mutex) TryLock() bool {
    select {
    case <-m.ch:
        return true
    default:
    }
    return false
}

// 加入一个超时的设置
func (m *Mutex) LockTimeout(timeout time.Duration) bool {
    timer := time.NewTimer(timeout)
    select {
    case <-m.ch:
        timer.Stop()
        return true
    case <-timer.C:
    }
    return false
}

// 锁是否已被持有
func (m *Mutex) IsLocked() bool {
    return len(m.ch) == 0
}


func main() {
    m := NewMutex()
    ok := m.TryLock()
    fmt.Printf("locked v %v\n", ok)
    ok = m.TryLock()
    fmt.Printf("locked %v\n", ok)
}
```

### Or-Done 模式

```

func or(channels ...<-chan interface{}) <-chan interface{} {
    //特殊情况，只有0个或者1个
    switch len(channels) {
    case 0:
        return nil
    case 1:
        return channels[0]
    }

    orDone := make(chan interface{})
    go func() {
        defer close(orDone)
        // 利用反射构建SelectCase
        var cases []reflect.SelectCase
        for _, c := range channels {
            cases = append(cases, reflect.SelectCase{
                Dir:  reflect.SelectRecv,
                Chan: reflect.ValueOf(c),
            })
        }

        // 随机选择一个可用的case
        reflect.Select(cases)
    }()


    return orDone
}
```

### 扇入模式

```

func fanInReflect(chans ...<-chan interface{}) <-chan interface{} {
    out := make(chan interface{})
    go func() {
        defer close(out)
        // 构造SelectCase slice
        var cases []reflect.SelectCase
        for _, c := range chans {
            cases = append(cases, reflect.SelectCase{
                Dir:  reflect.SelectRecv,
                Chan: reflect.ValueOf(c),
            })
        }
        
        // 循环，从cases中选择一个可用的
        for len(cases) > 0 {
            i, v, ok := reflect.Select(cases)
            if !ok { // 此channel已经close
                cases = append(cases[:i], cases[i+1:]...)
                continue
            }
            out <- v.Interface()
        }
    }()
    return out
}
```

### 扇出模式

```

```

### map-reduce

这个程序使用 map-reduce 模式处理一组整数，map 函数就是为每个整数乘以 10，reduce 函数就是把 map 处理的结果累加起来：

```
package main

import "fmt"

func mapChan(in <-chan interface{}, fn func(interface{}) interface{}) <-chan interface{} {
	out := make(chan interface{}) //创建一个输出chan
	if in == nil { // 异常检查
		close(out)
		return out
	}

	go func() { // 启动一个goroutine,实现map的主要逻辑
		defer close(out)
		for v := range in { // 从输入chan读取数据，执行业务操作，也就是map操作
			out <- fn(v)
		}
	}()

	return out
}



func reduce(in <-chan interface{}, fn func(r, v interface{}) interface{}) interface{} {
	if in == nil { // 异常检查
		return nil
	}

	out := <-in // 先读取第一个元素
	for v := range in { // 实现reduce的主要逻辑
		out = fn(out, v)
	}

	return out
}


// 生成一个数据流
func asStream(done <-chan struct{}) <-chan interface{} {
	s := make(chan interface{})
	values := []int{1, 2, 3, 4, 5}
	go func() {
		defer close(s)
		for _, v := range values { // 从数组生成
			select {
			case <-done:
				return
			case s <- v:
			}
		}
	}()
	return s
}

func main() {
	in := asStream(nil)

	// map操作: 乘以10
	mapFn := func(v interface{}) interface{} {
		return v.(int) * 10
	}

	// reduce操作: 对map的结果进行累加
	reduceFn := func(r, v interface{}) interface{} {
		return r.(int) + v.(int)
	}

	sum := reduce(mapChan(in, mapFn), reduceFn) //返回累加结果
	fmt.Println(sum)
}
```

