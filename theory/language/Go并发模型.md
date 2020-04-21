# Go并发编程模式

本文介绍一些常用的Go并发编程模式（Concurrency Design Patterns），以及这些模式在Tendermint和Cosmos-SDK项目中的应用。本文重点介绍的模式包括：

* Run Forever
* for-select
* Timing out, moving on
* Quit Channel
* Generator
* Producer/Consumer
* Publisher/Subscriber

## Run Forever

使用Go语言编写的程序至少会运行一个goroutine，那就是主goroutine；如果主goroutine执行结束，整个程序就终止了。对于服务器程序，通常会启动其他goroutine来提供服务，因此主goroutine不能终止。通常有两种做法。第一种做法是创建一个unbuffered channel，然后从中读取消息（由于没有goroutine往里面写消息，所以会一直阻塞）：

```go
func main() {
	... // start other goroutines
	var blockForever chan struct{}
	<-blockForever
}
```

第二种做法更简单，直接写一个空的select语句即可（由于没有case分支，也没有default分支，因此会一直阻塞）：
```go
func main() {
	... // start other goroutines
	select {} // block forever
}
```
Tendermint/Cosmos-SDK项目多处使用了第二种做法。以Cosmos-SDK的StartCmd命令为例，这个命令执行时会调用startStandAlone()或startInProcess()函数，这两个函数都需要无限阻塞：

```go
func startStandAlone(ctx *Context, appCreator AppCreator) error {
	...
	// run forever (the node will not be returned)
	select {}
	return nil
}

func startInProcess(ctx *Context, appCreator AppCreator) (*node.Node, error) {
	...
	// run forever (the node will not be returned)
	select {}
}
```

## for-select

在for循环中调用select语句也是一种常用的模式，这种模式看起来是下面这样：

```go
for { // Either loop infinitely or range over something 
  select {
    // Do some work with channels
	} 
}
```
这种模式在Tendermint里用的比较多，以Monitor的listen()方法为例：

```go
// main loop where we listen for events from the node
func (m *Monitor) listen(nodeName string, blockCh <-chan tmtypes.Header, blockLatencyCh <-chan float64, disconnectCh <-chan bool, quit <-chan struct{}) {
	logger := m.logger.With("node", nodeName)

	for {
		select {
		case <-quit:
			return
		case b := <-blockCh:
			m.Network.NewBlock(b)
			m.Network.NodeIsOnline(nodeName)
			m.NodeIsOnline(nodeName)
		case l := <-blockLatencyCh:
			m.Network.NewBlockLatency(l)
			m.Network.NodeIsOnline(nodeName)
			m.NodeIsOnline(nodeName)
		case disconnected := <-disconnectCh:
			if disconnected {
				m.Network.NodeIsDown(nodeName)
			} else {
				m.Network.NodeIsOnline(nodeName)
				m.NodeIsOnline(nodeName)
			}
		case <-time.After(nodeLivenessTimeout):
			logger.Info("event", fmt.Sprintf("node was not responding for %v", nodeLivenessTimeout))
			m.Network.NodeIsDown(nodeName)
		}
	}
}
```

## Timing out, moving on

select语句是这样工作的：

1. 如果有分支（channel）可读或者可写，则随机挑选一个分支进行读/写
2. 如果所有的分支（channel）都不可读/写，但是有default分支，则执行default分支
3. 否则阻塞，直到有分支可读/写为止

如果我们想在select语句无法立即读/写channel时等待一定时间，则可以使用“Timeout”模式。这个模式通常借助标准库time.After()函数实现：

```go
// After waits for the duration to elapse and then sends the current time
// on the returned channel.
// It is equivalent to NewTimer(d).C.
// The underlying Timer is not recovered by the garbage collector
// until the timer fires. If efficiency is a concern, use NewTimer
// instead and call Timer.Stop if the timer is no longer needed.
func After(d Duration) <-chan Time {
	return NewTimer(d).C
}
```
这个模式又可以分两种用法：timeout whole和timeout per event。第一种用法一般和for-select模式配合使用，整个for-select语句会有一个超时时间，类似下面这样：

```go
func main() {
    c := boring("Joe")
    timeout := time.After(5 * time.Second) // <---
    for {
        select {
        case s := <-c:
            fmt.Println(s)
        case <-timeout:
            fmt.Println("You talk too much.")
            return
        }
    }
}
```
第二种用法会给select语句单独设置超时时间，类似下面这样：

```go
func main() {
    c := boring("Joe")
    for {
        select {
        case s := <-c:
            fmt.Println(s)
        case <-time.After(1 * time.Second): // <---
            fmt.Println("You're too slow.")
            return
        }
    }
}
```
Tendermint大量使用了上面介绍的第二种用法，比如前面给出的Monitor.send()方法就是一个例子。再比如Channel.sendBytes()方法：

```go
// Queues message to send to this channel.
// Goroutine-safe
// Times out (and returns false) after defaultSendTimeout
func (ch *Channel) sendBytes(bytes []byte) bool {
	select {
	case ch.sendQueue <- bytes:
		atomic.AddInt32(&ch.sendQueueSize, 1)
		return true
	case <-time.After(defaultSendTimeout): // <---
		return false
	}
}
```

## Quit channel

在使用for-select模式时，可能会需要一种退出机制，以便在合适的时机退出循环，这时候就可以搭配quit channel模式一起使用。这种模式也有两种用法，普通的用法类似下面这样：
```go
select {
  case c <- fmt.Sprintf("%s: %d", msg, i):
    // do nothing
  case <-quit:
    return
  }
```
Receive on quit channel用法类似下面这样:

```go
  select {
  case c <- fmt.Sprintf("%s: %d", msg, i):
    // do nothing
  case <-quit:
    cleanup()
    quit <- "See you!"
    return
  }
```
Cosmos-SDK的IAVL迭代器使用了第一种用法：

```go
// Run this to funnel items from the tree to iterCh.
func (iter *iavlIterator) iterateRoutine() {
	iter.tree.IterateRange(
		iter.start, iter.end, iter.ascending,
		func(key, value []byte) bool {
			select {
			case <-iter.quitCh: // <---
				return true // done with iteration.
			case iter.iterCh <- cmn.KVPair{Key: key, Value: value}:
				return false // yay.
			}
		},
	)
	close(iter.iterCh) // done.
}
```

## Generator

Generator模式的一般表现为一个函数，这个函数返回一个只读channel，函数内部开一个新的goroutine产生并往channel里写消息。类似下面这样

```go
func generateXXX() <-chan XXX {
  ch := make(chan XXX)
  for {
    xxx := getXXX()
    ch <- xxx
  }
}
```
Tendermint项目consensus包common_test.go文件里面有一个具体的例子：

```go
func subscribeToVoter(cs *ConsensusState, addr []byte) <-chan tmpubsub.Message {
	votesSub, err := cs.eventBus.SubscribeUnbuffered(context.Background(), testSubscriber, types.EventQueryVote)
	if err != nil {
		panic(fmt.Sprintf("failed to subscribe %s to %v", testSubscriber, types.EventQueryVote))
	}
	ch := make(chan tmpubsub.Message)
	go func() {
		for msg := range votesSub.Out() {
			vote := msg.Data().(types.EventDataVote)
			// we only fire for our own votes
			if bytes.Equal(addr, vote.Vote.ValidatorAddress) {
				ch <- msg
			}
		}
	}()
	return ch
}
```

## Producer-Consumer

Producer-Consumer是一种比较经典的模式，这种模式也有很多不同的用法，下面给出一个简单的例子：

```go
package main

import "fmt"

var done = make(chan bool)
var msgs = make(chan int)

func produce() {
	for i := 0; i < 10; i++ {
		msgs <- i
	}
	done <- true
}

func consume() {
	for {
		msg := <-msgs
		fmt.Println(msg)
	}
}

func main() {
	go produce()
	go consume()
	<-done
}
```
前面提到过的Cosmos-SDK里的iavlIterator用了这个模式，先来看一下newIAVLIterator()函数：
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
	go iter.iterateRoutine() // <---
	go iter.initRoutine()
	return iter
}
```
忽略其他细节，可以看到go iter.iterateRoutine()这行代码，这实际上就是一个Producer，负责往iterCh里写KVPair：
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
Consumer则可以调用Next()方法推进迭代器（具体用法可以参考tm-db提供的Iterator接口），Next()方法内部会调用receiveNext()方法从iterCh里读取kvPair：
```go
func (iter *iavlIterator) receiveNext() {
	kvPair, ok := <-iter.iterCh
	if ok {
		iter.setNext(kvPair.Key, kvPair.Value)
	} else {
		iter.setInvalid()
	}
}
```