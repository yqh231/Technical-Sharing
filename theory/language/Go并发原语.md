# Go并发编程元语

本文首先对并发编程模型和元语进行简要介绍，然后结合Tendermint/Cosmos-SDK源代码详细介绍Go语言支持的两种并发编程模型所提供的元语。本文不区分并发（Concurrency）和并行（Parallel），文中会根据情况使用这两个词中的一个。

### 并发编程模型

并发编程模型可以按两种方式分类：按照process交互方式，或者按照问题分解方式。按照process交互方式又可以进一步分为共享内存模型、消息传递模型、隐式交互模型。很多编程语言（比如Java）提供的线程模型就属于共享内存模型，在这种模型下，并行执行的多个线程通过共享内存来进行交互。消息传递模型又可以进一步分为同步和异步两种模式，同步模式比较常用的有Communicating Sequential Processes 1（后文简称CSP）模型，异步模式比较常用的有Actor模型。Actor模型被Scala等语言采用，CSP模型则被Go语言采用并且大放异彩。本文只结合Go语言讨论共享内存和CSP这两种模型，下面给出并发编程模型的分类仅供参考。

```
Parallel programming model：

    Process interaction

        Shared memory
        Message passing
        synchronous
        CSP
        asynchronous
        Actor
        Implicit interation

Problem decomposition

        Task parallelism
        Data parallelism
        Implicit parallelism
```

### 并发编程元语

不管某个编程语言使用哪种并发编程模型，都需要提供一些基本的语法或者API来供程序员在这种模型下编程，这些基本语法或API就叫做并发编程元语（Primitives）。大家都知道在Go语言里可以使用go关键字非常方便的开启协程（Goroutine），Go语言同时支持共享内存和CSP这两种编程模型。共享内存元语主要由sync包提供，包括Mutex和Cond等。CSP元语则是内置在语言内的，主要包括Channel类型和select语句。下面是Go语言提供的（也是本文将要介绍的）并发编程元语。

```
Go concurrency primitives：

    Shared memory
        sync.Locker
        sync.Mutex
        sync.RWMutex
        sync.WaitGroup
        sync.Cond
        sync.Once
        sync.Pool
CSP
        channels
        select
```

这里需要指出，虽然Go语言同时支持共享内存和CSP两种编程模型，但通常还是鼓励使用CSP模型。只有在真正必要时，再使用sync包。Go的口号是：
> Do not communicate by sharing memory; instead, share memory by communicating.

### 共享内存模型元语

Go语言共享内存编程模型主要由标准库sync包支持，这一节对sync包提供的6个元语进行介绍。

#### sync.Mutex

Mutex是Go语言提供的互斥锁实现。Mutex本身的用法非常简单，只有Lock()和Unlock()两个方法。这两个方法也是Locker接口仅有的两个方法，因此Mutex实现了Locker接口：

```go
// A Locker represents an object that can be locked and unlocked.
type Locker interface {
    Lock()
    Unlock()
}
```
在Go语言里，使用Mutex的套路一般是：当需要进入临界区时调先用Lock()方法加锁，然后使用defer语句调用Unlock()方法解锁，最后访问临界区。以Cosmos-SDK提供的cachekv.Store为例：

```go
// Store wraps an in-memory cache around an underlying types.KVStore.
type Store struct {
    mtx           sync.Mutex
    cache         map[string]*cValue
    unsortedCache map[string]struct{}
    sortedCache   *list.List // always ascending sorted
    parent        types.KVStore
}
```

Store的Get()、Set()、Delete()等方法均需要保证并发安全性，所以按照前面描述的套路使用Mutex。以Get()方法为例：

```go
// Implements types.KVStore.
func (store *Store) Get(key []byte) (value []byte) {
    store.mtx.Lock()
    defer store.mtx.Unlock()
    types.AssertValidKey(key)

    cacheValue, ok := store.cache[string(key)]
    if !ok {
        value = store.parent.Get(key)
        store.setCacheValue(key, value, false, false)
    } else {
        value = cacheValue.value
    }

    return value
}
```

#### sync.RWMutex

RWMutex是Go语言提供的读写锁实现。和Mutex一样，RWMutex也实现了Locker接口，用于操作写锁。另外增加了RLock()、RUnlock()、RLocker()三个方法，用于操作读锁。下面是RWMutex的完整API

```go
type RWMutex struct { /* fields */ }
func (rw *RWMutex) Lock()
func (rw *RWMutex) Unlock()
func (rw *RWMutex) RLock()
func (rw *RWMutex) RUnlock()
func (rw *RWMutex) RLocker() Locker
```
RWMutex使用起来也很简单：当需要进入临界区时进行写操作时，使用写锁；如果只读，则使用读锁。以Tendermint里的BlockStore为例：

```go
// BlockStore is a simple low level store for blocks.
type BlockStore struct {
    db dbm.DB

    mtx    sync.RWMutex
    height int64
}
```

Height()方法只读取块的高度，因此使用读锁即可：

```go
// Height returns the last known contiguous block height.
func (bs *BlockStore) Height() int64 {
    bs.mtx.RLock()
    defer bs.mtx.RUnlock()
    return bs.height
}
```

SaveBlock()方法要写高度，因此使用写锁：

```go
func (bs *BlockStore) SaveBlock(block *types.Block, blockParts *types.PartSet, seenCommit *types.Commit) {
    ...

    // Done!
    bs.mtx.Lock()
    bs.height = height
    bs.mtx.Unlock()

    // Flush
    bs.db.SetSync(nil, nil)
}
```

#### sync.WaitGroup

如果要等待n个Goroutine结束，那么可以使用WaitGroup（类似Java提供的CountDownLatch）。WaitGroup内部维护了一个计数器：Add()方法可以对计数器增加任意值（包括负值）；Done()方法对计数器减一；Wait()方法会导致Goroutine被阻塞，直到计数器变为0。下面是WaitGroup的完整API：

```go
type WaitGroup struct { /* fields */ }
func (wg *WaitGroup) Add(delta int)
func (wg *WaitGroup) Done()
func (wg *WaitGroup) Wait()
```

其实Done()方法只是对Add()方法的简单封装而已：

```go
// Done decrements the WaitGroup counter by one.
func (wg *WaitGroup) Done() {
    wg.Add(-1)
}
```

使用WaitGroup的一般套路是：第一步，创建WaitGroup实例；第二步，调用Add()方法设置需要等待的Goroutine数量；第三步，启动Goroutine干活儿，并且在Goroutine内部调用Done()方法；第四步，调用Wait()方法等待全部Goroutine结束。以Tendermint里的p2p.Switch#Broadcast()方法为例：

```go
// Broadcast runs a go routine for each attempted send, which will block trying
// to send for defaultSendTimeoutSeconds. Returns a channel which receives
// success values for each attempted send (false if times out). Channel will be
// closed once msg bytes are sent to all peers (or time out).
//
// NOTE: Broadcast uses goroutines, so order of broadcast may not be preserved.
func (sw *Switch) Broadcast(chID byte, msgBytes []byte) chan bool {
    sw.Logger.Debug("Broadcast", "channel", chID, "msgBytes", fmt.Sprintf("%X", msgBytes))

    peers := sw.peers.List()
    var wg sync.WaitGroup // step#1
    wg.Add(len(peers))    // step#2
    successChan := make(chan bool, len(peers))

    for _, peer := range peers {
        go func(p Peer) {
            defer wg.Done() // step#3
            success := p.Send(chID, msgBytes)
            successChan <- success
        }(peer)
    }

    go func() {
        wg.Wait() // step#4
        close(successChan)
    }()

    return successChan
}
```

#### sync.Cond

如果想等待某种条件（Condition）发生，或者等待某种信号发出，那么可以使用Cond。Cond总是和一个Locker相关联，调用Wait()方法之前需要先锁住这个Locker，Wait()方法内部会自动解锁Locker。所有调用Wait()方法的Goroutine会进入一个等待队列并暂停执行（同时释放锁），调用Signal()方法可以恢复等待队列中的某个Goroutine，调用Broadcast()方法则可以恢复等待队列中的全部Goroutine。Wait()方法返回后，相应的Goroutine会继续持有锁。下面是Cond的完整API：

```go
type Cond struct { 
    L Locker // L is held while observing or changing the condition
    // other fields
}
func NewCond(l Locker) *Cond
func (c *Cond) Wait()
func (c *Cond) Signal()
func (c *Cond) Broadcast()
```

由于Tendermint/Cosmos-SDK并没有用到Cond，所以我们看一下Tendermint依赖的gRPC。gRPC定义了一个Server结构体：

```go
// Server is a gRPC server to serve RPC requests.
type Server struct {
    opts options

    mu     sync.Mutex // guards following
    lis    map[net.Listener]bool
    conns  map[io.Closer]bool
    serve  bool
    drain  bool
    cv     *sync.Cond          // signaled when connections close for GracefulStop
    m      map[string]*service // service name -> service info
    events trace.EventLog

    ... // other fields
}
```

Server实例由NewServer()函数创建，这个函数里会调用NewCond()函数初始化cv字段：

```go
// NewServer creates a gRPC server which has no service registered and has not
// started to accept requests yet.
func NewServer(opt ...ServerOption) *Server {
    opts := defaultServerOptions
    for _, o := range opt {
        o(&opts)
    }
    s := &Server{ ... }
    s.cv = sync.NewCond(&s.mu) // <---
    if EnableTracing { ... }
    if channelz.IsOn() { ... }
    return s
}
```

调用Wait()方法的套路一般是这样：先持有锁，然后循环判断条件并在循环内调用Wait()方法等待，循环退出后条件满足进行操作，最后解锁。下面是这种套路的伪代码：

```go
c.L.Lock()
for !condition() {
    c.Wait()
}
... make use of condition ...
c.L.Unlock()
```

Server的GracefulStop()方法需要“优雅”的关闭服务，所以要等待所有连接都关闭。该方法就是按照套路行事：

```go
// GracefulStop stops the gRPC server gracefully. It stops the server from
// accepting new connections and RPCs and blocks until all the pending RPCs are
// finished.
func (s *Server) GracefulStop() {
    ...
    s.mu.Lock()

    for len(s.conns) != 0 {
        s.cv.Wait() // wait in a loop
    }
    s.conns = nil         // make use of condition
    if s.events != nil {  //
        s.events.Finish() //
        s.events = nil    //
    }                     //
    s.mu.Unlock()
}
```

Server的removeConn()方法在连接关闭之后会调用Broadcast()方法：

```go
func (s *Server) removeConn(c io.Closer) {
    s.mu.Lock()
    defer s.mu.Unlock()
    if s.conns != nil {
        delete(s.conns, c)
        s.cv.Broadcast() // <---
    }
}
```

需要指出的是，调用Singal()和Broadcast()并不一定要持有锁。不过上面的removeConn()方法因为要改变共享状态（conns字段），所以才需要持有锁。

#### sync.Once

如果想在并发情况下保证某操作只被执行一次，那么可以使用Once。Once非常简单，只有一个Do()方法，用于提交操作。下面是Once的完整API：

```go
type Once struct { /* fields */ }
func (o *Once) Do(f func())
```

Tendermint/Cosmos-SDK也没有使用Once，前面介绍gRPC的Server结构体时省略了一些字段，下面给出该结构体的完整定义：

```go
// Server is a gRPC server to serve RPC requests.
type Server struct {
    opts options

    mu     sync.Mutex // guards following
    lis    map[net.Listener]bool
    conns  map[io.Closer]bool
    serve  bool
    drain  bool
    cv     *sync.Cond          // signaled when connections close for GracefulStop
    m      map[string]*service // service name -> service info
    events trace.EventLog

    quit               chan struct{}
    done               chan struct{}
    quitOnce           sync.Once
    doneOnce           sync.Once
    channelzRemoveOnce sync.Once
    serveWG            sync.WaitGroup // counts active Serve goroutines for GracefulStop

    channelzID int64 // channelz unique identification number
    czData     *channelzData
}
```

可以看到，Server结构体包含了三个Once类型的字段。以Stop()方法为例：

```go
func (s *Server) Stop() {
    s.quitOnce.Do(func() { // <---
        close(s.quit)
    })

    defer func() {
        s.serveWG.Wait()
        s.doneOnce.Do(func() { // <---
            close(s.done)
        })
    }()

    ...
}
```
Once的使用保证了quit和done这两个channel仅被关闭一次。

#### sync.Pool

虽然Go语言有自动垃圾回（GC）收机制，但是如果有一些对象创建和销毁代价比较大，则使用一个对象池来重复利用这些对象也是不错的主意。Pool就是来做这件事的，并且保证并发安全性。Pool只有两个方法：Get()从池子里取一个对象，Put()把对象归还给池子。Pool还有一个可选的New字段，类型是函数，当池子空的时候，Get()方法会调用New()函数创建新对象。下面是Pool的完整API：

```go
type Pool struct {
    ... // other fields

    // New optionally specifies a function to generate
    // a value when Get would otherwise return nil.
    // It may not be changed concurrently with calls to Get.
    New func() interface{}
}
func (p *Pool) Put(x interface{})
func (p *Pool) Get() interface{}
```

Tendermint的tmfmtLogger使用Pool来缓存tmfmtEncoder实例：

```go
var tmfmtEncoderPool = sync.Pool{
    New: func() interface{} {
        var enc tmfmtEncoder
        enc.Encoder = logfmt.NewEncoder(&enc.buf)
        return &enc
    },
}
```
Log()方法先调用Get()方法从池子里取出一个encoder，接着使用defer语句调用Put()归还encoder，然后使用encoder。这也是使用Pool的一般套路，代码如下所示：

```go
func (l tmfmtLogger) Log(keyvals ...interface{}) error {
    enc := tmfmtEncoderPool.Get().(*tmfmtEncoder)
    enc.Reset()
    defer tmfmtEncoderPool.Put(enc)
    ... // use enc
}
```

### CSP模型元语

这一节围绕Go语言提供的chan类型以及相关语法介绍CSP编程模型。

#### Channel基本用法

和共享内存模型不同，Go语言从语法上对CSP模型进行支持。从这一点也可以看出，用Go语言编程时，使用CSP模型才更自然一些。CSP模型最重要的概念就是消息传递，为此Go语言提供了特殊的channel类型。顾名思义，channel就是通道，消息可以经由通道在Goroutine之间流通。而在通道里流通的消息，则可以是任意类型（包括channel）。必须使用chan关键字来声明channel类型。由于channel是强类型的，因此chan的后面要跟上允许在通道里流动的消息的类型，例如：

`var mychan chan string // channel of string`

通道的实例由内置函数make()创建，例如：

`mychan = make(chan string)`

要想给channel发送消息，或者从channel里读取消息，可以使用<-运算符。如果<-在channel的左边则表示读取消息，如果在右边则表示发送消息：

```go
s := <-mychan // read from channel
mychan <- s   // write to channel
```

还可以使用for range语句循环从channel里读取消息：

```go
for s := range mychan {
    fmt.Println(s)
}
```

如果channel不再有用，可以使用内置函数close()把它关闭：

`close(mychan)`

Channel非常像其他语言（比如Java）里的阻塞队列（Blocking Queue），只不过容量为0（后面会进一步解释）。如果一个Goroutine试图从channel里读取消息会被阻塞，直到有其他Goroutine往里面发送消息为止。反之，如果一个Goroutine试图从channel里发送消息也会被阻塞，直到有其他Goroutine从里面拿走这个消息为止。Tendermint/Cosmos-SDK代码中大量使用了channel，以Cosmos-SDK提供的iavl包为例：

```go
// Implements types.Iterator.
type iavlIterator struct {
    tree *iavl.ImmutableTree // Underlying store
    start, end []byte        // Domain
    ascending bool           // Iteration order
    iterCh chan cmn.KVPair   // Channel to push iteration values.
    quitCh chan struct{}     // Close this to release goroutine.
    initCh chan struct{}     // Close this to signal that state is initialized.
    ... // other fields
}
```

上面给出了iavlIterator结构体的代码，可以看到，这个结构体包含了三个chan类型的字段。下面是newIAVLIterator()函数的代码，从中可以看到make()函数的使用：

```go
// newIAVLIterator will create a new iavlIterator.
// CONTRACT: Caller must release the iavlIterator, as each one creates a new
// goroutine.
func newIAVLIterator(tree *iavl.ImmutableTree, start, end []byte, ascending bool) *iavlIterator {
	iter := &iavlIterator{
		tree:      tree,
		start:     types.Cp(start),
		end:       types.Cp(end),
		ascending: ascending,
		iterCh:    make(chan cmn.KVPair), // Set capacity > 0?
		quitCh:    make(chan struct{}),
		initCh:    make(chan struct{}),
	}
	go iter.iterateRoutine()
	go iter.initRoutine()
	return iter
}
```

iterateRoutine()函数则展示了<-运算符以及close()函数的使用：

```go
// Run this to funnel items from the tree to iterCh.
func (iter *iavlIterator) iterateRoutine() {
	iter.tree.IterateRange(
		iter.start, iter.end, iter.ascending,
		func(key, value []byte) bool {
			select {
			case <-iter.quitCh:
				return true // done with iteration.
			case iter.iterCh <- cmn.KVPair{Key: key, Value: value}:
				return false // yay.
			}
		},
	)
	close(iter.iterCh) // done.
}
```

#### select语句

如果想同时操作多个channel，那么可以使用select语句（语法和switch语句有点像）。如果有多个channel可用，那么select语句会随机选择一个可用channel进行读或者写。如果没有channel可用，并且有default分支，则会执行此分支。如果没有channel可用也没有提供default分支，那么整个select语句会被阻塞。下面是select语言的一般形式：

```go
select {
    case <-c1:
        // do somethine
    case <-c2:
        // do somethine
    case c3 <- x:
        // do somethine
    case c4 <- y:
        // do somethine
    default:
        // do somethine
}
```

Tendermint/Cosmos-SDK代码中也是大量使用了select语句，前面介绍的iterateRoutine()方法就是一个例子。下面再从Temdermint p2p包中选取一个方法，以供参考：

```go
// Accept implements Transport.
func (mt *MultiplexTransport) Accept(cfg peerConfig) (Peer, error) {
    select {
    // This case should never have any side-effectful/blocking operations to
    // ensure that quality peers are ready to be used.
    case a := <-mt.acceptc:
        if a.err != nil { return nil, a.err }
        cfg.outbound = false
        return mt.wrapPeer(a.conn, a.nodeInfo, cfg, a.netAddr), nil
    case <-mt.closec:
        return nil, ErrTransportClosed{}
    }
}
```

由于select语句的case和default分支均可省略，所以一个惯用法就是通过空的select {}来“永久”阻塞Goroutine。以Tendermint提供的node命令为例：

```go
// NewRunNodeCmd returns the command that allows the CLI to start a node.
// It can be used with a custom PrivValidator and in-process ABCI application.
func NewRunNodeCmd(nodeProvider nm.NodeProvider) *cobra.Command {
	cmd := &cobra.Command{
		Use:   "node",
		Short: "Run the tendermint node",
		RunE: func(cmd *cobra.Command, args []string) error {
			n, err := nodeProvider(config, logger)
			if err != nil {
				return fmt.Errorf("Failed to create node: %v", err)
			}

			// Stop upon receiving SIGTERM or CTRL-C.
			cmn.TrapSignal(logger, func() {
				if n.IsRunning() { n.Stop() }
			})

			if err := n.Start(); err != nil {
				return fmt.Errorf("Failed to start node: %v", err)
			}
			logger.Info("Started node", "nodeInfo", n.Switch().NodeInfo())

			select {} // Run forever.
		},
	}

	AddNodeFlags(cmd)
	return cmd
}
```
