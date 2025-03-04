## Lockfree

[English](README_EN.md) | [日本語版](README_JA.md)

> 如果想使用低于 go1.18 版本则可以引入 tag:1.0.9 或 branch：below-version1.18

> 通过条件编译支持 386 平台，但性能测试发现比较差，因此不建议使用

### 1. 简介

#### 1.1. 为什么要写 Lockfree

在 go 语言中一般都是使用 chan 作为消息传递的队列，但在实际高并发的环境下使用发现 chan 存在严重的性能问题，其直接表现就是将对象放入到 chan 中时会特别耗时，
即使 chan 的容量尚未打满，在严重时甚至会产生几百 ms 还无法放入到 chan 中的情况。

![放大.jpg](images%2F%E6%94%BE%E5%A4%A7.jpg)

#### 1.2. chan 基本原理

##### 1.2.1. chan 基本结构

chan 关键字在编译阶段会被编译为 runtime.hchan 结构，它的结构如下所示：

![chan结构.jpg](images%2Fchan%E7%BB%93%E6%9E%84.jpg)

其中 sudog 是一个比较常见的结构，是对 goroutine 的一个封装，它的主要结构：

![sudog.jpg](images%2Fsudog.jpg)

##### 1.2.2. chan 写入流程

![write.jpg](images%2Fwrite.jpg)

##### 1.2.3. chan 读取流程

![read.jpg](images%2Fread.jpg)

#### 1.3. 锁

##### 1.3.1. runtime.mutex

chan 结构中包含了一个 lock 字段：`lock mutex`。
这个 lock 看名字就知道是一个锁，当然它不是我们业务中经常使用的 sync.Mutex，而是一个`runtime.mutex`。
这个锁是一个互斥锁，在 linux 系统中它的实现是 futex，在没有竞争的情况下，会退化成为一个自旋操作，速度非常快，但是当竞争比较大时，它就会在内核中休眠。

futex 的基本原理如下图：
![futex.jpg](images%2Ffutex.jpg)

当竞争非常大时，对于 chan 而言其整体大部分时间是出于系统调用上，所以性能下降非常明显。

##### 1.3.2. sync.Mutex

而`sync.Mutex`包中的设计原理如下图：

![锁.jpg](images%2F%E9%94%81.jpg)

### 2. Lockfree 基本原理

#### 2.1. 模块及流程

> 在最新的设计中已经删除了 available 模块，转而使用 ringBuffer 中的对象（e）中的 c（游标）标识写入状态。

![lockfree.jpg](images%2Flockfree.jpg)

#### 2.2. 优化点

##### 1) 无锁实现

内部所有操作都是通过原子变量(atomic)来操作，唯一有可能使用锁的地方，是提供给用户在 RingBuffer 为空时的等待策略，用户可选择使用 chan 阻塞

##### 2) 单一消费协程

屏蔽掉消费端读操作竞争带来的性能损耗

##### 3) 写不等待原则

符合写入快的初衷，当无法写入时会持续通过自旋和任务调度的方式处理，一方面尽量加快写入效率，另一方面则是防止占用太多 CPU 资源

##### 4) 泛型加速

引入泛型，泛型与 interface 有很明显的区别，泛型是在编译阶段确定类型，这样可有效降低在运行时进行类型转换的耗时

##### 5) 一次性内存分配

环状结构 Ringbuffer 实现对象的传递，通过确定大小的切片实现，只需要分配一次内存，不会涉及扩容等操作，可有效提高处理的性能

##### 6) 运算加速

RingBuffer 的容量为 2 的 n 次方，通过与运算来代替取余运算，提高性能

##### 7) 并行位图

用原子位运算实现位图并行操作，在尽量不影响性能的条件下，降低内存消耗

并行位图的思路实现历程：

![bitmap.jpg](images%2Fbitmap.jpg)

##### 8) 缓存行填充

根据 CPU 高速缓存的特点，通过填充缓存行方式屏蔽掉伪共享问题。

缓存行填充应该是一个比较常见的问题，它的本质是因为 CPU 比较快，而内存比较慢，所以增加了高速缓存来处理：

![padding1.jpg](images%2Fpadding1.jpg)

在两个 Core 共享同一个 L3 的情况下，如果同时进行修改，就会出现竞争关系（会涉及到缓存一致性协议：MESI）：

![padding2.jpg](images%2Fpadding2.jpg)

在 Lockfree 中有两个地方用到了填充：

![padding3.jpg](images%2Fpadding3.jpg)

最新版本中只保留了 cursor 中的填充，在 e 中使用了游标。

##### 9) Pointer 替代切片

屏蔽掉切片操作必须要进行越界判断的逻辑，生成更高效机器码。

![pointer.jpg](images%2Fpointer.jpg)

#### 2.3. 核心模块

##### ringBuffer

具体对象的存放区域，通过数组（定长切片）实现环状数据结构，其中的数据对象是具体的结构体而非指针，这样可以一次性进行内存申请。

##### stateDescriptor

> 注：最新的版本已将该对象删除，通过 ringBuffer 中 e 中的游标来描述状态。这样更充分利用了内存，降低了消耗。

状态描述符，定义了对应位置的数据状态，是可读还是可写。提供了三种方式：

- 1.  基于 Uint32 的切片：每个 Uint32 值描述一个位置，性能最高，但内存消耗最大；

- 2.  基于 Bitmap：每个 bit 描述一个位置，性能最低，但内存消耗最小；

- 3.  基于 Uint8 的切片：每个 Uint8 值描述一个位置，性能适中，消耗也适中，最推荐的方式。

##### sequencer

序号产生器，维护读和写两个状态，写状态具体由内部游标（cursor）维护，读取状态由自身维护，一个 uint64 变量维护。它的核心方法是 next()，用于获取下个可以写入的游标。

##### Producer

生产者，核心方法是 Write，通过调用 Write 方法可以将对象写入到队列中。支持多个 g 并发操作，保证加入时处理的效率。

##### consumer

消费者，这个消费者只会有一个 g 操作，这样处理的好处是可以不涉及并发操作，其内部不会涉及到任何锁，对于实际的并发操作由该 g 进行分配。

##### blockStrategy

阻塞策略，该策略用于 buf 中长时间没有数据时，消费者阻塞设计。它有两个方法：block()和 release()。前者用于消费者阻塞，后者用于释放。
系统提供了多种方式，不同的方式 CPU 资源占用和性能会有差别：

- 1. SchedBlockStrategy：调用 runtime.Gosched()进行调度 block，不需要 release，为推荐方式；

- 2. SleepBlockStrategy：调用 time.Sleep(x)进行 block，可自定义休眠时间，不需要 release，为推荐方式；

- 3. ProcYieldBlockStrategy：调用 CPU 空跑指令，可自定义空跑的指令数量，不需要 release；

- 4. OSYieldBlockStrategy：操作系统会将对应 M 调度出去，等时间片重新分配后可执行，不需要 release；

- 5. ChanBlockStrategy：chan 阻塞策略，需要 release，为推荐方式；

- 6. CanditionBlockStrategy：candition 阻塞策略，需要 release，为推荐方式；

其中 1/2/5/6 为推荐方式，如果性能要求比较高，则优先考虑 2 和 1，否则建议试用 5 和 6。

##### EventHandler

事件处理器接口，整个项目中唯一需要用户实现的接口，该接口描述消费端收到消息时该如何处理，它使用泛型，通过编译阶段确定事件类型，提高性能。

### 3. 使用方式

#### 3.1. 导入模块

可使用 `go get github.com/bruceshao/lockfree` 获取最新版本

#### 3.2. 代码调用

为了提升性能，Lockfree 支持 go 版本 1.18 及以上，以便于支持泛型，Lockfree 使用非常简单：

```go
package main

import (
	"fmt"
	"sync"
	"sync/atomic"
	"time"

	"github.com/bruceshao/lockfree"
)

var (
	goSize    = 10000
	sizePerGo = 10000

	total = goSize * sizePerGo
)

func main() {
	// lockfree计时
	now := time.Now()

	// 创建事件处理器
	handler := &eventHandler[uint64]{
		signal: make(chan struct{}, 0),
		now:    now,
	}

	// 创建消费端串行处理的Lockfree
	lf := lockfree.NewLockfree[uint64](
		1024*1024,
		handler,
		lockfree.NewSleepBlockStrategy(time.Millisecond),
	)

	// 启动Lockfree
	if err := lf.Start(); err != nil {
		panic(err)
	}

	// 获取生产者对象
	producer := lf.Producer()

	// 并发写入
	var wg sync.WaitGroup
	wg.Add(goSize)
	for i := 0; i < goSize; i++ {
		go func(start int) {
			for j := 0; j < sizePerGo; j++ {
				err := producer.Write(uint64(start*sizePerGo + j + 1))
				if err != nil {
					panic(err)
				}
			}
			wg.Done()
		}(i)
	}

	// wait for producer
	wg.Wait()

	fmt.Printf("producer has been writed, write count: %v, time cost: %v \n", total, time.Since(now).String())

	// wait for consumer
	handler.wait()

	// 关闭Lockfree
	lf.Close()
}

type eventHandler[T uint64] struct {
	signal   chan struct{}
	gcounter uint64
	now      time.Time
}

func (h *eventHandler[T]) OnEvent(v uint64) {
	cur := atomic.AddUint64(&h.gcounter, 1)
	if cur == uint64(total) {
		fmt.Printf("eventHandler has been consumed already, read count: %v, time cose: %v\n", total, time.Since(h.now))
		close(h.signal)
		return
	}

	if cur%10000000 == 0 {
		fmt.Printf("eventHandler consume %v\n", cur)
	}
}

func (h *eventHandler[T]) wait() {
	<-h.signal
}
```

### 4. 性能对比

#### 4.1. 简述

在实际测试中发现，如果 lockfree 和 chan 同时跑的话会有一些影响，lockfree 的表现基本是正常的，和 chan 同时跑的时候性能基本是下降的。
但 chan 比较奇怪，和 lockfree 一起跑的时候性能比 chan 自身跑性能还高。目前正在排查此问题，但不影响使用。

main 包下提供了测试的程序，可自行进行性能测试（假设编译后的二进制为 lockfree）：

- 1）单独测试 lockfree：./lockfree lockfree [time]，加入 time 会有时间分布；
- 2）单独测试 chan：./lockfree chan [time]，加入 time 会有时间分布；
- 3）合并测试 lockfree 和 chan：./lockfree [all] [time]，使用 time 时，前面必须加 all 参数，只进行测试，不关注时间分布的话，可直接调用./lockfree；

为描述性能，除了时间外，定义了**QR**（Quick Ratio，快速率）的指标，该指标描述的是写入时间在 1 微秒以内的操作占所有操作的比值。
自然的，QR 越大，性能越高。

#### 4.2. 软硬件测试环境

CPU 信息如下(4 \* 2.5GHz)：

```shell
[root@VM]# lscpu
Architecture:          x86_64
CPU op-mode(s):        32-bit, 64-bit
Byte Order:            Little Endian
CPU(s):                4
On-line CPU(s) list:   0-3
Thread(s) per core:    1
Core(s) per socket:    4
Socket(s):             1
NUMA node(s):          1
Vendor ID:             GenuineIntel
CPU family:            6
Model:                 94
Model name:            Intel(R) Xeon(R) Gold 6133 CPU @ 2.50GHz
Stepping:              3
CPU MHz:               2494.120
BogoMIPS:              4988.24
Hypervisor vendor:     KVM
Virtualization type:   full
L1d cache:             32K
L1i cache:             32K
L2 cache:              4096K
L3 cache:              28160K
NUMA node0 CPU(s):     0-3
```

内存信息（8G）：

```shell
[root@VM]# free -m
              total        used        free      shared  buff/cache   available
Mem:           7779         405        6800         116         573        7216
Swap:             0           0           0
```

操作系统（centos 7.2）：

```shell
[root@VM]# cat /etc/centos-release
CentOS Linux release 7.2 (Final)
[root@VM]# uname -a
Linux VM-219-157-centos 3.10.107-1-tlinux2_kvm_guest-0056 #1 SMP Wed Dec 29 14:35:09 CST 2021 x86_64 x86_64 x86_64 GNU/Linux
```

云厂商：腾讯云。

#### 4.3. 写入性能对比

设定 buffer 大小为 1024 \* 1024，无论是 lockfree 还是 chan 都是如此设置。其写入的时间对比如下：

其中 100 \* 10000 表示有 100 个 goroutine，每个 goroutine 写入 10000 次，其他的依次类推。

all（lockfree/chan）表示在 lockfree 和 chan 同时跑的情况下，其分别的时间占比情况。

| 类型            | 100 \* 10000 | 500 \* 10000 | 1000 \* 10000 | 5000 \* 10000 | 10000 \* 10000 |
| --------------- | ------------ | ------------ | ------------- | ------------- | -------------- |
| lockfree        | 67ms         | 306ms        | 676ms         | 3779ms        | 7703ms         |
| chan            | 116ms        | 1991ms       | 4709ms        | 26897ms       | 58509ms        |
| all（lockfree） | 49ms         | 414ms        | 976ms         | 5038ms        | 10946ms        |
| all（chan）     | 83ms         | 859ms        | 3029ms        | 19228ms       | 40473ms        |

#### 4.4. QR 分布

快速率的分布情况如下所示：

| 类型            | 100 \* 10000 | 500 \* 10000 | 1000 \* 10000 | 5000 \* 10000 | 10000 \* 10000 |
| --------------- | ------------ | ------------ | ------------- | ------------- | -------------- |
| lockfree        | 99.23        | 99.78        | 99.81         | 99.49         | 98.99          |
| chan            | 91.67        | 88.99        | 57.79         | 3.98          | 1.6            |
| all（lockfree） | 99.69        | 99.88        | 99.88         | 99.52         | 99.02          |
| all（chan）     | 96.72        | 93.5         | 93.1          | 51.37         | 48.2           |

#### 4.5. 结果

从上面两张表可以很明显看出如下几点：

- 1）在 goroutine 数量比较小时，lockfree 和 chan 性能差别不明显；
- 2）当 goroutine 打到一定数量（大于 1000）后，lockfree 无论从时间还是 QR 都远远超过 chan；
