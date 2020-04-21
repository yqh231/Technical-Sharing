# Go函数式编程

函数式编程（Functional Programming）实际是非常古老的概念，不过近几年大有越来越流行之势，连很多老牌语言（比如Java）也增加了对函数式编程的支持。本文结合Temdermint/Cosmos-SDK源代码，介绍函数式编程中最重要的一些概念，以及如何使用Go语言进行函数式编程。以下是本文将要讨论的主要内容：

* 一等函数
* 高阶函数
* 匿名函数
* 闭包
* λ表达式

## 一等函数

如果在一门编程语言里，函数（或者方法、过程、子例程等，各种语言叫法不同）享有一等公民的待遇，那么我们就说这门语言里的函数是一等函数（First-class Function ）。那怎样才能算是“一等公民”呢？简单来说就是和其他数据类型待遇差不多，不会被区别对待。比如说：可以把函数赋值给变量或者结构体字段；可以把函数存在array、map等数据结构里；可以把函数作为参数传递给其他函数；也可以把函数当其他函数的返回值返回；等等。下面结合Cosmos-SDK源代码举几个具体的例子。

### 普通变量

以auth模块的AccountKeeper为例，这个Keeper提供了一个GetAllAccounts()方法，返回系统中所有的账号：

```go
// GetAllAccounts returns all accounts in the accountKeeper.
func (ak AccountKeeper) GetAllAccounts(ctx sdk.Context) []Account {
	accounts := []Account{}
	appendAccount := func(acc Account) (stop bool) {
		accounts = append(accounts, acc)
		return false
	}
	ak.IterateAccounts(ctx, appendAccount)
	return accounts
}
```
从代码可以看到，函数被赋值给了普通的变量appendAccount，进而又被传递给了IterateAccounts()方法。

### 结构体字段

Cosmos-SDK提供了BaseApp结构体，作为构建区块链App的"基础"：

```go
// BaseApp reflects the ABCI application implementation.
type BaseApp struct {
	// 省略无关字段
	anteHandler    sdk.AnteHandler  // ante handler for fee and auth
	initChainer    sdk.InitChainer  // initialize state with validators and state blob
	beginBlocker   sdk.BeginBlocker // logic to run before any txs
	endBlocker     sdk.EndBlocker   // logic to run after all txs, and to determine valset changes
	addrPeerFilter sdk.PeerFilter   // filter peers by address and port
	idPeerFilter   sdk.PeerFilter   // filter peers by node ID
	// 省略无关字段
}
```
这个结构体定义了大量的字段，其中有6个字段是函数类型。这些函数起到callback或者hook的作用，影响具体区块链app的行为。下面是这些函数的类型定义：
```go
// cosmos-sdk/types/handler.go
type AnteHandler func(ctx Context, tx Tx, simulate bool) (newCtx Context, result Result, abort bool)
// cosmos-sdk/types/abci.go
type InitChainer func(ctx Context, req abci.RequestInitChain) abci.ResponseInitChain
type BeginBlocker func(ctx Context, req abci.RequestBeginBlock) abci.ResponseBeginBlock
type EndBlocker func(ctx Context, req abci.RequestEndBlock) abci.ResponseEndBlock
type PeerFilter func(info string) abci.ResponseQuery
```

### Slice元素

Cosmos-SDK提供了Int类型，用来表示256比特整数。下面是从int_test.go文件中摘出来的一个单元测试，演示了如何把函数保存在slice里：

```go
func TestImmutabilityAllInt(t *testing.T) {
	ops := []func(*Int){
		func(i *Int) { _ = i.Add(randint()) },
		func(i *Int) { _ = i.Sub(randint()) },
		func(i *Int) { _ = i.Mul(randint()) },
		func(i *Int) { _ = i.Quo(randint()) },
		func(i *Int) { _ = i.AddRaw(rand.Int63()) },
		func(i *Int) { _ = i.SubRaw(rand.Int63()) },
		func(i *Int) { _ = i.MulRaw(rand.Int63()) },
		func(i *Int) { _ = i.QuoRaw(rand.Int63()) },
		func(i *Int) { _ = i.Neg() },
		func(i *Int) { _ = i.IsZero() },
		func(i *Int) { _ = i.Sign() },
		func(i *Int) { _ = i.Equal(randint()) },
		func(i *Int) { _ = i.GT(randint()) },
		func(i *Int) { _ = i.LT(randint()) },
		func(i *Int) { _ = i.String() },
	}

	for i := 0; i < 1000; i++ {
		n := rand.Int63()
		ni := NewInt(n)

		for opnum, op := range ops {
			op(&ni) // 调用函数

			require.Equal(t, n, ni.Int64(), "Int is modified by operation. tc #%d", opnum)
			require.Equal(t, NewInt(n), ni, "Int is modified by operation. tc #%d", opnum)
		}
	}
}
```

### Map值

还是以BaseApp为例，这个包里定义了一个queryRouter结构体，用来表示“查询路由”：

```go
type queryRouter struct {
	routes map[string]sdk.Querier
}
```

从代码可以看出，这个结构体的routes字段是一个map，值是函数类型，在queryable.go文件中定义：

```go
// Type for querier functions on keepers to implement to handle custom queries
type Querier = func(ctx Context, path []string, req abci.RequestQuery) (res []byte, err Error)
```
把函数作为其他函数的参数和返回值的例子在下一小节中给出。

## 高阶函数

高阶函数（Higher Order Function）听起来很高大上，但其实概念也很简单：如果一个函数有函数类型的参数，或者返回值是函数类型，那么这个函数就是高阶函数。以前面出现过的AccountKeeper的IterateAccounts()方法为例：

```go
func (ak AccountKeeper) IterateAccounts(ctx sdk.Context, process func(Account) (stop bool)) {
	store := ctx.KVStore(ak.key)
	iter := sdk.KVStorePrefixIterator(store, AddressStoreKeyPrefix)
	defer iter.Close()
	for {
		if !iter.Valid() { return }
		val := iter.Value()
		acc := ak.decodeAccount(val)
		if process(acc) { return }
		iter.Next()
	}
}
```
由于它的第二个参数是函数类型，所以它是一个高阶函数（或者更严谨一些，高阶方法）。同样是在auth模块里，有一个NewAnteHandler()函数：

```go
// NewAnteHandler returns an AnteHandler that checks and increments sequence
// numbers, checks signatures & account numbers, and deducts fees from the first
// signer.
func NewAnteHandler(ak AccountKeeper, fck FeeCollectionKeeper) sdk.AnteHandler {
	return func(ctx sdk.Context, tx sdk.Tx, simulate bool) (newCtx sdk.Context, res sdk.Result, abort bool) {
		// 代码省略
  }
}
```
这个函数的返回值是函数类型，所以它也是一个高阶函数。

## 匿名函数

像上面例子中的NewAnteHandler()函数是有自己的名字的，但是在定义和使用高阶函数时，使用匿名函数（Anonymous Function）更方便一些。比如NewAnteHandler()函数里的返回值就是一个匿名函数。匿名函数在Go代码里面非常常见，比如很多函数都需要使用defer关键字来确保某些逻辑推迟到函数返回前执行，这个时候用匿名函数就很方便。仍然以NewAnteHandler函数为例：

```go
// NewAnteHandler returns an AnteHandler that checks and increments sequence
// numbers, checks signatures & account numbers, and deducts fees from the first
// signer.
func NewAnteHandler(ak AccountKeeper, fck FeeCollectionKeeper) sdk.AnteHandler {
	return func(ctx sdk.Context, tx sdk.Tx, simulate bool) (newCtx sdk.Context, res sdk.Result, abort bool) {
		// 前面的代码省略

		// AnteHandlers must have their own defer/recover in order for the BaseApp
		// to know how much gas was used! This is because the GasMeter is created in
		// the AnteHandler, but if it panics the context won't be set properly in
		// runTx's recover call.
		defer func() {
			if r := recover(); r != nil {
				switch rType := r.(type) {
				case sdk.ErrorOutOfGas:
					log := fmt.Sprintf(
						"out of gas in location: %v; gasWanted: %d, gasUsed: %d",
						rType.Descriptor, stdTx.Fee.Gas, newCtx.GasMeter().GasConsumed(),
					)
					res = sdk.ErrOutOfGas(log).Result()

					res.GasWanted = stdTx.Fee.Gas
					res.GasUsed = newCtx.GasMeter().GasConsumed()
					abort = true
				default:
					panic(r)
				}
			}
		}() // 就地执行匿名函数

		// 后面的代码也省略
	}
}
```

再比如使用go关键字执行goroutine，具体例子参见定义在cosmos-sdk/server/util.go文件中的TrapSignal()函数：

```go
// TrapSignal traps SIGINT and SIGTERM and terminates the server correctly.
func TrapSignal(cleanupFunc func()) {
	sigs := make(chan os.Signal, 1)
	signal.Notify(sigs, syscall.SIGINT, syscall.SIGTERM)
	go func() {
		sig := <-sigs
		switch sig {
		case syscall.SIGTERM:
			defer cleanupFunc()
			os.Exit(128 + int(syscall.SIGTERM))
		case syscall.SIGINT:
			defer cleanupFunc()
			os.Exit(128 + int(syscall.SIGINT))
		}
	}()
}
```

## 闭包

如果匿名函数能够捕捉到词法作用域（Lexical Scope）内的变量，那么匿名函数就可以成为闭包（Closure）。闭包在Cosmos-SDK/Temdermint代码里也可谓比比皆是，以bank模块的NewHandler()函数为例：

```go
// NewHandler returns a handler for "bank" type messages.
func NewHandler(k Keeper) sdk.Handler {
	return func(ctx sdk.Context, msg sdk.Msg) sdk.Result {
		switch msg := msg.(type) {
		case MsgSend:
			return handleMsgSend(ctx, k, msg) // 捕捉到k
		case MsgMultiSend:
			return handleMsgMultiSend(ctx, k, msg) // 捕捉到k
		default:
			errMsg := "Unrecognized bank Msg type: %s" + msg.Type()
			return sdk.ErrUnknownRequest(errMsg).Result()
		}
	}
}
```

从代码不难看出：NewHandler()所返回的匿名函数捕捉到了外围函数的参数k，因此返回的其实是个闭包。

## λ表达式

匿名函数也叫做λ表达式（Lambda Expression），不过很多时候当我们说λ表达式时，一般指更简洁的写法。以前面出现过的TestImmutabilityAllInt()函数为例，下面是它的部分代码：

```go
ops := []func(*Int){
	func(i *Int) { _ = i.Add(randint()) },
	func(i *Int) { _ = i.Sub(randint()) },
	func(i *Int) { _ = i.Mul(randint()) },
	// 其他代码省略
}
```
从这个简单的例子不难看出，Go的匿名函数写法还是有一定冗余的。如果把上面的代码翻译成Python的话，看起来像是下面这样：

```python
ops = [
  lambda i: i.add(randint()),
  lambda i: i.sub(randint()),
  lambda i: i.mul(randint()),
  # 其他代码省略
]
```

