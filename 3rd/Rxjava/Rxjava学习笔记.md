## Rxjava学习笔记      

### Rxjava优点
  Rxjava是一种响应式编程，内部是有观察者模式实现的，是一个实现异步操作的库，常与Retrofit相结合实现网络异步请求。相对于传统编程自己主动去获取数据，主动获取数据的流向，然后将数据跟数据的流向代码结合起来；Rxjava采用的是被观察者拿到数据后主动通知观察者，将展示层与数据层进行分离，通过线程的切换，减少在主线程做耗时操作的错误，解耦了各个模块，通过各种API的拦截，来进行数据流的传输。异步，简洁，内部支持多线程操作。Rxjava大大减少接口的使用，丰富的Api基本满足需求，大大减少代码修改，保持代码整洁，逻辑清晰，便于阅读。
### Rxjava操作符使用场景
* create() Rxjava最基本的创建事件序列的方法
* just()将传入的参数依次发送出来
* from() 将传入的数组或者Iterable拆成具体对象后，依次发送出来
* map()/flatmap() map与flatmap都是传入一个参数，参数一般是func1<I,O>,I为输入类型，O为输出类型，实现func1的call方法把I类型转化为O类型进行输出。但是flatmap输出参数一般为observable类型。
* delay/timer 都是做延迟操作的 delay是延迟某段时间后，第一个数据延迟数据发送，其他数据不执行延迟发送；timer 返回一个 Observable , 它在延迟一段给定的时间后发射一个简单的数字0。timer 操作符默认在computation调度器上执行，当然也可以用 Scheduler 在定义执行的线程，常用于页面跳转。delay延时发射，发射后可以连续发射多次数据；timer延时发射，发射完成后，只能发射一次数据。
* interval 轮询操作符 可以循环发送数据 可用于支付时，轮询服务器去获取支付结果或者发送短信验证码时，读取验证码信息
* merge：合并并观察对象，把两个集合合并成一个集合
* filter：过滤数据 例如过滤空数据或者不符合条件的数据
* take：取前n条数据 例如取集合首条数据
* startWith：插入数据 在集合前面添加自定义数据
* doOnNext在onNext之前调用，主要用在网络数据请求之后，数据展示之前，把网络数据缓存到本地
* throttleFirst:在一段时间内，只取第一个事件，其他事件丢失；适用于button按钮防抖操作，防止多次点击；百度关键词联想，在一段时间内只访问一次，防止频繁请求网络，减缓服务器压力
* debounce:一段时间内没有变化，就会发送数据。百度联想关键词提示，输入400毫秒后，请求数据，减缓服务器压力。
* doOnSubscribe() 可以在事件发出之前做一些初始化的工作，比如弹出进度条等等
  * <Strong>注意：</Strong>
  
    1.doOnSubscribe() 默认运行在事件产生的线程里面，然而事件产生的线程一般都会运行在 io 线程里。那么这个时候做一些，更新UI的操作，是线程不安全的。所以如果事件产生的线程是io线程，但是我们又要在doOnSubscribe()更新UI这时候就需要线程切换

     2.如果在 doOnSubscribe() 之后有 subscribeOn() 的话，它将执行在离它最近的 subscribeOn() 所指定的线程。


### Rxjava生命周期控制
Rxjava使用大量的订阅事件，会造成内存吃紧，易造成内存溢出。observable.subscribe时会产生对应的subscription，当需要释放时，直接在onDestory()中调用subscription.unsubscribe() 对资源进行释放(有些资料说onDestory()调用时机太晚，可根据实际情况在onPause(),onStop()释放资源，不建议在onPause()中做过多耗时操作，特别是在页面跳转之前，这样会导致跳转页面延迟，因为页面跳转势必会调用onPause(),只有当onPause()中操作执行完毕后，才会跳转)。


### Rxjava线程调度 
![Rxjava线程调度图](https://github.com/hushendian/pic/blob/master/2.png?raw=true)

* subscribeOn() 事件产生的线程 ； observeOn() : 事件消费的线程
* observeOn:<Strong>可以多次执行</Strong>，每执行一次切换一次线程
* subscribeOn：用来确定数据发射所在的线程，<Strong>只能执行一次</Strong>，可以在任意位置
 .subscribeOn(Schedulers.io()) 和 .observeOn(AndroidSchedulers.mainThread()) 写的位置不一样，造成的结果也不一样。
 用于请求数据后，在主线程中展示数据时，切换线程。

* 对于 create() , just() , from()等                ----- 事件产生   
*  map() , flapMap() , scan() , filter()等 ------  事件加工
* subscribe()  ----  事件消费
* 事件产生：默认运行在当前线程，可以由 subscribeOn()  自定义线程
*  事件加工：默认跟事件产生的线程保持一致, 可以由 observeOn() 自定义线程
* 事件消费：默认运行在当前线程，可以有observeOn() 自定义  
* 只规定事件产生的线程，则Rxjava事件的产生与消费都运行在规定的线程中
* 只规定事件消费的线程则Rxjava事件的产生运行在当前线程，而事件的消费运行在规定的线程中
* subscribeOn事件产生线程的5中模式：
1. immediate()默认运行在当前线程，相当于不指定线程
2. newThread()启动新的线程，并在新的线程中启动
3. io() I/o操作（文件读写，数据库读写，网络信息交互），行为模式与newThread相似，IO内部维护一个无数量上限的线程池，可以复用空闲的线程，性能比newThread高
4. computation() 计算所使用的scheduler，一般不常用
5. mainThread() 指定的操作将运行在主线程


### Rxjava线程调度坑
from()/just()中不能执行耗时操作，易发生ANR。
因为just()，from()这类能够创建Observable的操作符在创建之初，就已经存储了对象的值，而不是在被订阅的时候才创建。所以在我们订阅之前，耗时操作就已经在开始执行了，如果有耗时操作，需要把from()/just()换成create()。

### 参考资料
* [Rxjava中just操作符的坑](https://www.jianshu.com/p/d482e929bbe6)
* [Rxjava基础及相关线程调度](https://www.cnblogs.com/zhaoyanjun/p/5624395.html)
* [Rxjava详解](http://gank.io/post/560e15be2dca930e00da1083)