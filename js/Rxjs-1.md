# RxJS 分享

> RxJS 是使用 Observables 的响应式编程的库，它使编写异步或基于回调的代码更容易。

[toc]

## 为什么要用RxJS

让我们的代码，更简洁，更有可读性。可能你会觉得使用Promise或者callback也能实现你要的功能，但是**RxJS** 可以让我们以流的方式，以更简洁优美的代码呈现。

示例：经常遇到多层调用接口的情况，比如先获取商品列表，再根据商品ID取商品对应的图片状态
```javascript
// 不用RxJS
getProductListPromise.then(data=> {
	const productIds = data.map(item => item.id);
	getProductsPicturesPromise.then(pictures => {
		// 可能还有下一层。。。
	})
})
```
```javascript
getProductList$
    .pipe(
        map(data => data.map(item => item.id)),
        switchMap(productIds => {
        	return getProductsPictures$(productIds);
        })
        // 如果还要再请求，继续使用switchMap
    )
	.subscribe(pictures => {})
```

## 基础概念

### Observable、Observer、Subscription
``` javascript
subscription = observable.subscribe(observer)
```

Observable：数据生产者，待订阅者

Observer：数据使用者，观察者

上面的代码返回的是一个订阅对象，代表一个订阅的发生，是Subscription的实例化

Observable.subscribe可以接收三个回调函数，next、error、complete，如果只传一个默认为next的回调。可以订阅，也有取消订阅，也就是unsubscribe()方法。

## RxJS特性
1. 纯净性
使用纯函数隔离应用状态
示例：
- 按钮的点击事件，普通javascript会创建变量，你很难保证这个变量不会被其他代码共享。
```javascript
let count = 0;
const button = document.querySelector('button');
button.addEventListener('click', () => console.log(`Clicked ${++count} times`));
```
- 使用RxJS，可以将应用状态隔离，不需要再创建变量
```javascript
const button = document.querySelector('button');
fromEvent(button, 'click')
	.pipe(
		scan(count => count + 1, 0)
	)
	.subscribe(count => console.log(`Clicked ${++count} times`));
```
2. 流动性
RxJS提供了一整套操作符来帮助我们控制事件如何流经observables
- 创建
- 组合
- 条件
- 过滤
- 错误处理
- 转换
- 多播
- 工具
3. 值的转换
对于流经observables的值，可以对其进行转换，并传入下一个observable
示例：对于普通javascript而言，每一次换算后的数据都需要一个变量来存储，然而 **RxJS** 通过隔离状态避免声明不必要的变量
```javascript
fromEvent(button, 'click')
	.pipe(
		throttleTime(500),
		map(event => event.clientX)
		scan((count, clientX) => count + clientX, 0)
	)
	.subscribe(count => console.log(`Clicked ${++count}`));
```

## 观察者模式
RxJS采用观察者模式实现响应式编程，将逻辑分为发布者和观察者。
![image](https://user-gold-cdn.xitu.io/2019/5/14/16ab462271b47b3b?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)
- 发布者：Observable角色，产生数据或事件
- 观察者：observer角色，接收发布者的数据并响应事件
- 什么时候推送数据，取决于什么订阅，也就是调用subscribe

## Observable与函数
1. Observable与函数都是隋性运算，不调用函数，函数不会执行，不订阅Observable，不会发送数据，都是独立的操作
```javascript
function foo() {
	console.log('foo');
	return 42;
}
var x = foo.call();
console.log(x);
var y = foo.call();
console.log(y);
```
```javascript
var foo = Observable.create(observer => {
	console.log('foo');
	observer.next(42);
});
foo.subscribe(x => console.log(x));
foo.subscribe(y => console.log(y));
```
输出是一样的
```
"foo"
42
"foo"
42
```
2. Observables传递值可以是同步的，也可以是异步的
3. 函数只能返回单个值，Observable可以随着时间推移推送多个值
```javascript
function foo() {
	return 42;
	return 100; // 永远不会执行
}
```
```javascript
var foo = Observable.create(observer => {
	observer.next(42);
	observer.next(100);
	setTimeout(() => {
		observer.next(300);
	}, 1000)
});
foo.subscribe(x => console.log(x));
console.log('after');
```
输出
```
42
100
"after"
300
```
4. 结论
	- func.call() 意思是“同步给我一个值”
	- observable.subscribe() 意思是“给我任意数量的值，无论同步还是异步”

## Observable与Promise
1. Promise是主动的，Observable是懒惰的

2. Promise只能发出单个值，Observable可以发出多个值

3. Promise不能被取消，Observable可以被取消

4. Promise只能是异步的，Observable可异步可同步

5. Promise只能执行一次，Observable可以执行多次
示例：请说出下方代码中，Promise的执行顺序，以及执行次数
```javascript
let time = 0;
const waitOneSecondPromise = new Promise((resolve) => {
	console.log('promise call');
	time = new Date.getTime();
	setTimeout(() => resolve('hello world!'), 1000);
})
waitOneSecondPromise.then(value => {
	console.log('first time', value, new Date().getTime() - time);
});
setTimeout(() => {
	waitOneSecondPromise.then(value => {
		console.log('second time', value, new Date().getTime() - time);
	})
}, 5000);

// 输出结果
promise call
first time hello world! 1007
second time hello world! 5006
```
```javascript
let time = 0;
const waitOneSecondObservable = new Observable(observer => {
	console.log('observable called');
	time = new Date().getTime();
	observer.next('hello world');
}).pipe(delay(1000));
waitOneSecondObservable.subscribe(value => {
	console.log('first time', value, new Date().getTime() - time);
});
setTimeout(() => {
	waitOneSecondObservable.subscribe(value => {
		console.log('second time', value, new Date().getTime() - time);
	});
}, 5000);

// 输出结果
observable called
first time hello world 1001
observable called
second time hello world 1002
```

observable.subscribe的每次调用都会触发针对给定观察者的独立设置，不同于addEventListener，不会给观察者注册监听器，甚至不会维护一个观察者列表。
```javascript
observable.subscribe == Observable.create(function subscribe(observer) {...})
```

## Observer 观察者
观察者只是三个回调函数的对象，每个回调函数对应一种Observable发送的通知类型。
```javascript
var observer = {
	next: x => console.log('Observer get a next value: ' + x),
	error: err => console.log('Observer get an error: ' + err),
	complete: () => console.log('Observer get a complete notify')
};
```
订阅Observable时，可以只提供一个回调函数作为参数，在observable.subscribe内部，它会创建 一个观察者对象并使用第一个回调函数作为next的处理方法。三个回调可以直接作为参数传递。
```javascript
observable.subscribe(
	x => console.log('Observer get a next value: ' + x),
	err => console.log('Observer get an error: ' + err),
	() => console.log('Observer get a complete notify')
)
```
## Subscription 订阅
Subscription表示可清理资源的对象，通常是Observable的执行。它有一个unsubscribe方法，用来取消Observable的执行，清理由Subscription占用的资源。
可以用add方法组合多个subscription，当unsubscribe的时候，会同时取消多个订阅对象。

## Subject 主体
```javascript
const obser = new Observable(observer => {
	observer.next(1);
	observer.next(2);
});
obser.subscribe(x => console.log('from A: ', x));
obser.subscribe(x => console.log('from B: ', x));
// 输出结果
from A: 1
from A: 2
from B: 1
from B: 2
```
```ts
const subject = new Subject();
subject.subscribe(x => console.log('from A: ', x));
subject.subscribe(x => console.log('from B: ', x));
subject.next(1);
subject.next(2);
// 输出结果
from A: 1
from B: 1
from A: 2
from B: 2
```
以上两段代码的区别，体现了Subject的多播性以及Subject即是Observable，也是观察者。
不同于单播的Observable，Subject.subscribe不会调用发送值的新执行，它只是将给定的观察者注册到观察者列表，类似于addListener方法。
因为Subject也是观察者，也可以将它作为参数传递给Observable.subscribe。

- 多播Observable
- BehaviorSubject
- ReplaySubject
- AsyncSubject

## 操作符
> 操作符是允许复杂的异步代码以声明式的方式进行轻松组合的基础代码单元。

操作符基于当前Observable创建一个新的Observable，不影响原Observable，是一个无副作用的 **纯函数** 。
```javascript
from([1, 2, 3, 4])
	.pipe(
		map(x => x * 10)
	)
	.subscribe(x => console.log(x));
// 输出结果
10
20
30
40
```

### 弹珠图
![image](https://cn.rx.js.org/manual/asset/marble-diagram-anatomy.svg)

### 创建操作符
- create
- empty
- from
- fromEvent
- fromPromise
- interval
- of
- range
- throw
- throwError
- timer
1. 我还没有Observable，需要来自Promise或者事件源或数组，使用 **from、fromPromise**
```javascript
const promiseSource = from(new Promise(resolve => resolve('Hello World!')));
const subscribe = promiseSource.subscribe(val => console.log(val));
// 输出: 'Hello World'
```
2. 我还没有Observable，需要轮循http请求，使用 **timer、interval**
```javascript
timer(0, 1000).subscribe(x => console.log(x));
// 输出: 0,1,2,3,...
```
3. 我还没有Observable，它的事件源来自DOM或Node.js，使用 **fromEvent**
```javascript
const source = fromEvent(document, 'click')
	.pipe(
		map(event => `Event time: ${event.timeStamp}`)
	)
	.subscribe(val => console.log(val));
```

### 转换操作符
- buffer
- bufferTime
- bufferCount
- bufferToggle
- bufferWhen
- concatMap
- concatMapTo: 将每个值总是映射为同一个内部 Observable。
- exhaustMap: 映射成内部Observable，忽略源Observable发出的值，直接内部Observable完成，再接收源Observable的值。
- mergeMap: 打平内部 observable 并手动控制内部订阅数量
- switchMap: 同一时间应该只有一个内部 subscription 是有效
- groupBy: 基于提供的值分组成多个Observables
- map/mapTo
- expand: 递归调用提供的函数
- partition: 将一个Observable根据条件分割成两个Observables
- pluck
- reduce/scan
- window: 每当windowBundaries发出值，将源Observable分支成嵌套的Observables
- windowTime
1. 我有一个Observable，然后我想从每个发出的值中选取一个属性，使用 **pluck**
```javascript
from([{ name: 'Joe', age: 30 }, { name: 'Sarah', age: 35 }])
	.pipe(pluck('name'));
	.subscribe(val => console.log(val));
// 输出: "Joe", "Sarah"
```
2. 我有一个Observable，我想收集输出值，直到提供的observable发出值，或者达到数量，或者时间期限达到，使用 **buffer、bufferCount、bufferTime**
```javascript
const bufferBy = fromEvent(document, 'click');
/*
  收集由 myInterval 发出的所有值，直到我们点击页面。此时 bufferBy 会发出值以完成缓存。
  将自上次缓冲以来收集的所有值传递给数组。
*/
interval(1000)
	.pipe(buffer(bufferBy))
	.subscribe(val => console.log(' Buffered Values:', val));
```
3. 我有一个Observable，我想把发出的每个值转换成Observable，并按顺序订阅发出，使用 **concatMap**，如果不关注顺序，可用 **mergeMap**，如果只有一个Observable，并需要取消前一个内部请求，比如响应输入场景，可用 **switchMap**
```javascript
of(2000, 1000)
	.pipe(
		concatMap(val => of(`delay by ${val}ms`).pipe(delay(val)))
	)
	.subscribe(val => console.log('concatMap: ', val));
// 输出结果
concatMap: delay by 2000ms
concatMap: delay by 1000ms
```
```javascript
of(2000, 1000)
	.pipe(
		mergeMap(val => of(`delay by ${val}ms`).pipe(delay(val)))
	)
	.subscribe(val => console.log('mergeMap: ', val));
// 输出结果
mergeMap: delay by 1000ms
mergeMap: delay by 2000ms
```
```javascript
fromEvent(document, 'click')
	.pipe(
		switchMap(val => interval(3000).pipe(mapTo('Hello, I made it!')))
	).subscribe(val => console.log(val));
```
4. 我有一个Observable，我想把所有发出的值进行公式计算，只输出最终的计算值，用 **reduce**，或者随着时间根据发出的值计算，并输出计算值，用 **scan**
```javascript
of(1, 2, 3, 4)
	.pipe(reduce((acc, val) => acc + val))
	.subscribe(val => console.log('Sum:', val));
// 输出结果
Sum: 10

of(1, 2, 3, 4)
	.pipe(scan((acc, curr) => acc + curr,0)
	.subscribe(val => console.log(val));
// 输出结果
1
3
6
10
```

### 过滤操作符
- debounce
- debounceTime
- distinctUntilChanged: 当前值与上一次值不同才会发出
- filter
- first
- ignoreEelements: 忽略所有值，除了complete和error
- last
- sample: 从源Observable发出的值中取样
- single: 发出通过表达式的单一项
- skip
- skipUntil: 提供的Observable发出值
- skipWhile: 提供的表达式为false
- take
- takeUntil
- takeWhile
- throttle
- throttleTime
1. 我有一个Observable，我需要过滤发出的值，只有在特定的时间后Observable发出的值，才不会被过滤。如果这个时间是变量可用 **debounce**，如果不是变量，**debounceTime**使用频率更高。
```javascript
interval(1000)
	.pipe(debounce(val => timer(val*200)))
	.subscribe(val => console.log(val));
// 输出结果
0...1...2...3...4
```
```javascript
fromEvent(document, 'click')
	.pipe(debounceTime(1000))
	.subscribe(val => console.log(val));
```
2. 我有一个Observable，我想基于自定义的逻辑过滤发出的值，可用**filter**；如果通过的值在Observable起始处，并且第一个，用**first**，或者基于给定的数量，用**take**，或者是最后一个用**last**，或者直到另一个Observable发出值并完成，用**takeUntil**。
```javascript
from([1, 2, 3, 4])
	.pipe(filter(val => val % 2 === 0))
	.subscribe(val => console.log(val));
```
```javascript
interval(1000)
	.pipe(takeUntil(timer(5000)))
	.subscribe(val => console.log(val));
// 输出结果
0...1...2...3...4
```
请说下下方示例中takeWhile与filter的区别
```javascript
const source = of(3, 3, 3, 9, 11, 2, 3, 5, 7, 3);
source
	.pipe(takeWhile(it => it === 3))
	.subscribe(val => console.log(val));

source
	.pipe(filter(it => it === 3))
	.subscribe(val => console.log(val));
```
3. 我有一个Observable，我想在源Observable发出值后，再等一段时间，再发出源Observable值，再重复，可用**throttle、throttleTime**。
```javascript
fromEvent(document, 'click')
	.pipe(throttleTime(1000))
	.subscribe(val => console.log(val));
```
ps. 有谁可以说下throttle和debounce之间的区别吗？
![image](https://i.stack.imgur.com/7b7qe.png)

![image](https://i.stack.imgur.com/wW8BX.png)

* debounce和throttle很相似，但是它们有一个非常重要的**区别**：debounce会保留追踪源Observable最后发出的值，并且在指定时间后，源Observable没有发出新值，才会发出这个值。
* debounce更适用于typeahead和autoComplete，控制用户输入请求API频率
* throttle/throttleTime的工作原理：
	* 初始timer是禁用的
	* 当源Observable发出第一个值后，timer启用，并发出第一个值 
	* 在timer发出值或完成前，源Observable发出的值都被忽略
	* 当timer发出值或完成后，timer禁用，并等待源Observable发出新值，开始重复步骤二
* debounce/debounceTime的工作原理:
	* 保留源Observable发出的最新值
	* 一旦时间超过设定时间或者timer发出值，源Observable还没有发出新值，就发出保留的值
	* 当新值出现后，新值发出，并丢弃之前保留的旧值

### 组合操作符
- combineAll: 源Observable完成，对内部的Observables使用combineLatest
- combineLatest: 当任意一个Observable发出值时，发出所有Observables的最新值
- concat
- concatAll: 合并内部Observables的值
- forkJoin
- merge
- mergeAll
- pairwise: 将前一个值和当前值作为数组发出
- race
- startWith
- withLatestFrom: 发出提供的Observable的最新值
- zip
1. 我有两个或以上的Observables，将它们组成一个Observable，并且要等所有完成后再通知我，可用**forkJoin**；或者不等所有完成，输出任何一个Observable的值，可用**merge**；或者我要按顺序订阅每个Observable，可用**concat**。
```javascript
forkJoin(
	of(1), 
	of('hello'), 
	_throw('This will error').pipe(catchError(err => of(error)))
	// 内部Observable报错时得到正常结果
).subscribe(val => console.log(val));
```
```javascript
merge(
	of('First').pipe(delay(2500)),
	of('Second').pipe(delay(1000)),
	of('Third').pipe(delay(1500))
).subscribe(val => console.log(val));
// 输出结果
"Second" "Third" "First"
```
```javascript
concat(
	of(1, 2, 3).pipe(delay(1000)),
	of(4, 5, 6)
).subscribe(val => console.log(val));
```
2. 我有两个或以上的Observables，我想输出的值是根据源Observables的值计算所得，可用**zip**或**combineLatest**。
```javascript
const source = interval(1000);
zip(source, source.pipe(take(2)))
	.subscribe(val => console.log(val));
```
```javascript
combineLatest(
	timer(1000, 4000), timer(2000, 4000), timer(3000, 4000)
).subscribe(latestValues => {
    const [timerValOne, timerValTwo, timerValThree] = latestValues;
    console.log(
        `Timer One Latest: ${timerValOne},
         Timer Two Latest: ${timerValTwo},
         Timer Three Latest: ${timerValThree}`
	);
});
// 输出结果
Timer One Latest: 0,
Timer Two Latest: 0,
Timer Three Latest: 0
Timer One Latest: 1,
Timer Two Latest: 0,
Timer Three Latest: 0
Timer One Latest: 1,
Timer Two Latest: 1,
Timer Three Latest: 0
Timer One Latest: 1,
Timer Two Latest: 1,
Timer Three Latest: 1
```

### 多播操作符
- publish: 共享源Observable，执行connect方法
- multicast: 提供Subject共享源Observable
- share
- shareReplay: 共享源，并重放指定次数，与ReplaySubject同理
1. 我有一个Observable，想在多个观察者之前共享同一个subscription，可用**share**，share比较像是使用Subject和refCount的**multicast**。当有订阅发生，connect()自动执行，当最后一个订阅取消，停止进一步的执行。
```javascript
const source = timer(1000).pipe(
	tap(() => console.log('***SIDE EFFECT***')),
	mapTo('***RESULT***'),
	share()
);
const subscribeOne = source.subscribe(val => console.log(val));
const subscribeTwo = source.subscribe(val => console.log(val))
```
```javascript
const source = timer(1000).pipe(
	tap(() => console.log('***SIDE EFFECT***')),
	mapTo('***RESULT***'),
	// 可指定Subject类型，BehaviorSubject, ReplaySubject, AsyncSubject
	multicast(() => new Subject())
);
const subscribeThree = source.subscribe(val => console.log(val));
const subscribeFour = source.subscribe(val => console.log(val));
source.connect();
```

### 条件操作符
- defaultIfEmpty: 空Observable的默认值
- every
- find
- findIndex
- isEmpty: 源Observable是否为空
1. 我有一个Observable，我想判断发出的值是否都满足条件，可用**every**
```javascript
of(1, 2, 3, 4)
	.pipe(every(val => val % 2 === 0))
	.subscribe(val => console.log(val));
// 输出结果
false
```
2. 我有一个Observable，我想获取第一个满足条件的值，用**find**，或者我想要它的索引，可用**findIndex**
```javascript
of(3, 9, 15, 20)
	.pipe(find(val => val % 5 === 0))
	.subscribe(val => console.log(val));
// 输出结果
15
```

### 错误处理操作符
- catchError
- retry
- retryWhen
1. 我有一个Observable，我想当错误发生时，开启一个新的Observable，用**catchError**。
```javascript
const badPromise = () => new Promise((resolve, reject) => reject('Rejected'));
timer(1000)
	.pipe(
		mergeMap(_ => from(badPromise()).pipe(catchError(err => of(`Bad promise: ${err}`))))
	)
	.subscribe(val => console.log(val));
```
2. 我想一个Observable，我想当错误发生时，重试，可用**retry**，或者重试时机是当另一个Observable发出，用**retryWhen**。
```javascript
const source = interval(1000).pipe(
	mergeMap(val => {
		if (val > 5) {
			return throwError('ERROR!');
		}
		return of(val);
	}),
	// 可指定重试次数
	retry(2)
).subscribe(
	val => console.log(val),
	error => console.log(`${error} retry 2 times and quit.`)
)
```
```javascript
interval(1000).pipe(
	map(val => {
		if (val > 5) {
			throw val; // 错误将由 retryWhen 接收
		}
		return val;
	}),
	retryWhen(errors =>
		errors.pipe(
			tap(val => console.log(`Value ${val} was too high!`))
		)
	)
).subscribe(val => console.log(val));
```

### 工具操作符
- do/tap
- delay
- delayWhen
- dematerialize: 将 notification 对象转换成 notification 值
- let
- timeout
- toPromise
- toArray
1. 我有一个Observable，想延迟一段时间发送，可用**delay**，或者我想根据自定义逻辑延迟，可用**delayWhen**。
```javascript
interval(1000)
	.pipe(
		take(5),
		delayWhen(() => timer(5000))
	).subscribe(val => console.log(val));
// 输出结果在5s后发出
```