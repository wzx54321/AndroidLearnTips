##  事件角度
###  产生事件
- creat()

```
Observable observable = Observable.create(new Observable.OnSubscribe<String>() {
    @Override
    public void call(Subscriber<? super String> subscriber) {
        subscriber.onNext("Hello");
        subscriber.onNext("Hi");
        subscriber.onNext("Aloha");
        subscriber.onCompleted();
    }
});
```

- Just()---最多10个数据

```
Observable observable = Observable.just("Hello", "Hi", "Aloha");
// 将会依次调用：
// onNext("Hello");
// onNext("Hi");
// onNext("Aloha");
// onCompleted();`
```
- from()---集合

```
 String[] words = {"Hello", "Hi", "Aloha"};
Observable observable = Observable.from(words);
// 将会依次调用：
// onNext("Hello");
// onNext("Hi");
// onNext("Aloha");
// onCompleted();

```
- defer（）

```
public class DeferActivity extends AppCompatActivity {
 
    String i = "10" ;
 
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_defer);
 
        i = "11 " ;
 
        Observable<String> defer = Observable.defer(new Func0<Observable<String>>() {
            @Override
            public Observable<String> call() {
                return Observable.just( i ) ;
            }
        }) ;
 
        Observable test = Observable.just(  i ) ;
 
        i = "12" ;
 
        defer.subscribe(new Action1<String>() {
            @Override
            public void call(String s) {
                Log.v( "rx_defer  " , "" + s ) ;
            }
        }) ;
 
        test.subscribe(new Action1() {
            @Override
            public void call(Object o) {
                Log.v( "rx_just " , "" + o ) ;
            }
        }) ;
    }
}
```
输出结果

```
/rx_defer: 12
/rx_just: 11
```

总结:just和defer都能创建事件,但他们是不同的：
- [x]     just操作符是在创建Observable就进行了赋值操作
- [x] defer是在订阅者订阅时才创建Observable，此时才进行真正的赋值操作 
- [x]  defer好处：在某些情况下，等待直到最后一分钟（就是知道订阅发生时）才生成Observable可以确保Observable包含最新的数据。

---



### 事件加工

>### **转化操作符**

#### map 一对一转换

```
Observable.just(1, 2, 3, 4, 5)
        .map(new Func1<Integer, String>() {
            @Override
            public String call(Integer i) {
                return "This is " + i;
            }
        }).subscribe(new Action1<String>() {
            @Override
            public void call(String s) {
                System.out.println(s);
            }
        });

```
####  flatMap  一对多转换/数据是无序的
flatMap()中返回的是Observable对象（所以多用于网络嵌套）

需求1：打印所有小区下面所有房子的价格


```
常规写法：
Community[] communities = {};
Observable.from(communities)
        .subscribe(new Action1<Community>() {
            @Override
            public void call(Community community) {
                for (House house : community.houses) {
                    System.out.println("House price : " + house.price);
                }
            }
        });


```


```
flatmap写法：
Observable.from(communities)
        .flatMap(new Func1<Community, Observable<House>>() {
            @Override
            public Observable<House> call(Community community) {
                return Observable.from(community.houses);
            }
        })
        .subscribe(new Action1<House>() {
            @Override
            public void call(House house) {
                System.out.println("House price : " + house.price);
            }
        });


```
需求2：请求接口1然后紧接着请求接口2

```
public class MainActivity extends AppCompatActivity {

        private static final String TAG = "Rxjava";

        // 定义Observable接口类型的网络请求对象
        Observable<Translation1> observable1;
        Observable<Translation2> observable2;

        @Override
        protected void onCreate(Bundle savedInstanceState) {
            super.onCreate(savedInstanceState);
            setContentView(R.layout.activity_main);

            // 步骤1：创建Retrofit对象
            Retrofit retrofit = new Retrofit.Builder()
                    .baseUrl("http://fy.iciba.com/") // 设置 网络请求 Url
                    .addConverterFactory(GsonConverterFactory.create()) //设置使用Gson解析(记得加入依赖)
                    .addCallAdapterFactory(RxJava2CallAdapterFactory.create()) // 支持RxJava
                    .build();

            // 步骤2：创建 网络请求接口 的实例
            GetRequest_Interface request = retrofit.create(GetRequest_Interface.class);

            // 步骤3：采用Observable<...>形式 对 2个网络请求 进行封装
            observable1 = request.getCall();
            observable2 = request.getCall_2();


            observable1.subscribeOn(Schedulers.io())               // （初始被观察者）切换到IO线程进行网络请求1
                       .observeOn(AndroidSchedulers.mainThread())  // （新观察者）切换到主线程 处理网络请求1的结果
                       .doOnNext(new Consumer<Translation1>() {
                        @Override
                        public void accept(Translation1 result) throws Exception {
                            Log.d(TAG, "第1次网络请求成功");
                            result.show();
                            // 对第1次网络请求返回的结果进行操作 = 显示翻译结果
                        }
                    })

                    .observeOn(Schedulers.io())                 // （新被观察者，同时也是新观察者）切换到IO线程去发起登录请求
                                                                // 特别注意：因为flatMap是对初始被观察者作变换，所以对于旧被观察者，它是新观察者，所以通过observeOn切换线程
                                                                // 但对于初始观察者，它则是新的被观察者
                    .flatMap(new Function<Translation1, ObservableSource<Translation2>>() { // 作变换，即作嵌套网络请求
                        @Override
                        public ObservableSource<Translation2> apply(Translation1 result) throws Exception {
                            // 将网络请求1转换成网络请求2，即发送网络请求2
                            return observable2;
                        }
                    })

                    .observeOn(AndroidSchedulers.mainThread())  // （初始观察者）切换到主线程 处理网络请求2的结果
                    .subscribe(new Consumer<Translation2>() {
                        @Override
                        public void accept(Translation2 result) throws Exception {
                            Log.d(TAG, "第2次网络请求成功");
                            result.show();
                            // 对第2次网络请求返回的结果进行操作 = 显示翻译结果
                        }
                    }, new Consumer<Throwable>() {
                        @Override
                        public void accept(Throwable throwable) throws Exception {
                            System.out.println("登录失败");
                        }
                    });
    }
}
```


### scan：累加操作符

```
Observable observable = Observable.just( 1 , 2 , 3 , 4 , 5  ) ;
       observable.scan(new Func2<Integer,Integer,Integer>() {
           @Override
           public Integer call(Integer o, Integer o2) {
               return o + o2 ;
           }
       })
         .subscribe(new Action1() {
             @Override
             public void call(Object o) {
                 System.out.println( "scan-- " +  o );
             }
         })   ;
```
//运行结果

```
1,3,6,10,15
```
### concatMap flatMapIterable

- concatMap：concatMap()解决了flatMap()的交叉问题，它能够把发射的值连续在一起
- flatMapIterable：flatMapIterable()和flatMap()几乎是一样的，不同的是flatMapIterable()它转化的多个Observable是使用Iterable作为源数据的。

flatMapIterable代码示例：
```
Observable.from(communities)
        .flatMapIterable(new Func1<Community, Iterable<House>>() {
            @Override
            public Iterable<House> call(Community community) {
                return community.houses;
            }
        })
        .subscribe(new Action1<House>() {

            @Override
            public void call(House house) {

            }
        });


```



### groupBy
注：将原始Observable发射的数据按照key来拆分成一些小的Observable，然后这些小Observable分别发射其所包含的的数据

//示例代码


```
List<House> houses = new ArrayList<>();
houses.add(new House("中粮·海景壹号", "中粮海景壹号新出大平层！总价4500W起"));
houses.add(new House("竹园新村", "满五唯一，黄金地段"));
houses.add(new House("中粮·海景壹号", "毗邻汤臣一品"));
houses.add(new House("竹园新村", "顶层户型，两室一厅"));
houses.add(new House("中粮·海景壹号", "南北通透，豪华五房"));
Observable<GroupedObservable<String, House>> groupByCommunityNameObservable = Observable.from(houses)
        .groupBy(new Func1<House, String>() {

            @Override
            public String call(House house) {
                return house.communityName;
            }
        });
        

```

```
Observable.concat(groupByCommunityNameObservable)
        .subscribe(new Action1<House>() {
            @Override
            public void call(House house) {
                System.out.println("小区:"+house.communityName+"; 房源描述:"+house.desc);
            }
        });


```
//执行结果

```
小区:中粮·海景壹号; 房源描述:中粮海景壹号新出大平层！总价4500W起
小区:中粮·海景壹号; 房源描述:毗邻汤臣一品
小区:中粮·海景壹号; 房源描述:南北通透，豪华五房
小区:竹园新村; 房源描述:满五唯一，黄金地段
小区:竹园新村; 房源描述:顶层户型，两室一厅
```



>### **过滤操作符**
### Filter：
条件过滤操作符

```
Observable observable = Observable.just( 1 , 2 , 3 , 4 , 5 , 6 , 7 ) ;
observable.filter(new Func1<Integer , Boolean>() {
    @Override
    public Boolean call(Integer o) {
        //数据大于4的时候才会被发送
        return o > 4 ;
    }
})
        .subscribe(new Action1() {
            @Override
            public void call(Object o) {
                System.out.println( "filter-- " +  o );
            }
        })   ;
```

```
执行结果： 5，6，7
```



### 消息数量过滤操作符的使用
- take ：取前n个数据
- takeLast：取后n个数据
- first 只发送第一个数据
- last 只发送最后一个数据
- skip() 跳过前n个数据发送后面的数据
- skipLast() 跳过最后n个数据，发送前面的数据


```
//take 发送前3个数据
       Observable observable = Observable.just( 1 , 2 , 3 , 4 , 5 , 6 , 7 ) ;
       observable.take( 3 )
               .subscribe(new Action1() {
                   @Override
                   public void call(Object o) {
                       System.out.println( "take-- " +  o );
                   }
               })   ;
 
       //takeLast 发送最后三个数据
       Observable observable2 = Observable.just( 1 , 2 , 3 , 4 , 5 , 6 , 7 ) ;
       observable2.takeLast( 3 )
               .subscribe(new Action1() {
                   @Override
                   public void call(Object o) {
                       System.out.println( "takeLast-- " +  o );
                   }
               })   ;
 
       //first 只发送第一个数据
       Observable observable3 = Observable.just( 1 , 2 , 3 , 4 , 5 , 6 , 7 ) ;
       observable3.first()
               .subscribe(new Action1() {
                   @Override
                   public void call(Object o) {
                       System.out.println( "first-- " +  o );
                   }
               })   ;
 
       //last 只发送最后一个数据
       Observable observable4 = Observable.just( 1 , 2 , 3 , 4 , 5 , 6 , 7 ) ;
       observable4.last()
               .subscribe(new Action1() {
                   @Override
                   public void call(Object o) {
                       System.out.println( "last-- " +  o );
                   }
               })   ;
 
       //skip() 跳过前2个数据发送后面的数据
       Observable observable5 = Observable.just( 1 , 2 , 3 , 4 , 5 , 6 , 7 ) ;
       observable5.skip( 2 )
               .subscribe(new Action1() {
                   @Override
                   public void call(Object o) {
                       System.out.println( "skip-- " +  o );
                   }
               })   ;
 
       //skipLast() 跳过最后两个数据，发送前面的数据
       Observable observable6 = Observable.just( 1 , 2 , 3 , 4 , 5 , 6 , 7 ) ;
       observable5.skipLast( 2 )
               .subscribe(new Action1() {
                   @Override
                   public void call(Object o) {
                       System.out.println( "skipLast-- " +  o );
                   }
               })   ;
```

//执行结果
![image](https://wx1.sinaimg.cn/mw690/006fqmU7ly1frvi4j7x4cj30kh0f3diy.jpg)


### takeUntil(Observable)
订阅并开始发射原始Observable，同时监视我们提供的第二个Observable。如果第二个Observable发射了一项数据或者发射了一个终止通知，takeUntil()返回的Observable会停止发射原始Observable并终止



```
//示例代码
Observable<Long> observableA = Observable.interval(300, TimeUnit.MILLISECONDS);
Observable<Long> observableB = Observable.interval(800, TimeUnit.MILLISECONDS);

observableA.takeUntil(observableB)
        .subscribe(new Subscriber<Long>() {
            @Override
            public void onCompleted() {
                System.exit(0);
            }
            @Override
            public void onError(Throwable e) {

            }
            @Override
            public void onNext(Long aLong) {
                System.out.println(aLong);
            }
        });

try {
    Thread.sleep(Integer.MAX_VALUE);
} catch (InterruptedException e) {
    e.printStackTrace();
}


```

```
执行结果：
0，1
```




### elementAt 、elementAtOrDefault 


```
//elementAt() 发送数据序列中第n个数据 ，序列号从0开始
        //如果该序号大于数据序列中的最大序列号，则会抛出异常，程序崩溃
        //所以在用elementAt操作符的时候，要注意判断发送的数据序列号是否越界
 
        Observable observable7 = Observable.just( 1 , 2 , 3 , 4 , 5 , 6 , 7 ) ;
        observable7.elementAt( 3 )
                .subscribe(new Action1() {
                    @Override
                    public void call(Object o) {
                        System.out.println( "elementAt-- " +  o );
                    }
                })   ;
 
        //elementAtOrDefault( int n , Object default ) 发送数据序列中第n个数据 ，序列号从0开始。
        //如果序列中没有该序列号，则发送默认值
        Observable observable9 = Observable.just( 1 , 2 , 3 , 4 , 5 ) ;
        observable9.elementAtOrDefault(  8 , 666  )
                .subscribe(new Action1() {
                    @Override
                    public void call(Object o) {
                        System.out.println( "elementAtOrDefault-- " +  o );
                    }
                })   ;
```


```
//执行结果
elementAt--：4
elementAtOrDefault--666

```





### Distinct/distinctUntilChanged()
- Distinct：过滤重复数据
- 
```
List<String> list = new ArrayList<>() ;
        list.add( "1" ) ;
        list.add( "2" ) ;
        list.add( "1" ) ;
        list.add( "3" ) ;
        list.add( "4" ) ;
        list.add( "2" ) ;
        list.add( "1" ) ;
        list.add( "1" ) ;

        Observable.from( list )
                .distinct()
                .subscribe(new Action1<String>() {
                    @Override
                    public void call(String s) {
                        System.out.println( "distinct--" + s );
                    }
                }) ;
```

```
//执行结果
1.2，3，4
```


- distinctUntilChanged()： 过滤连续重复数据

```
List<String> list = new ArrayList<>() ;
        list.add( "1" ) ;
        list.add( "2" ) ;
        list.add( "1" ) ;
        list.add( "3" ) ;
        list.add( "4" ) ;
        list.add( "4" ) ;
        list.add( "2" ) ;
        list.add( "1" ) ;
        list.add( "1" ) ;

        Observable.from( list )
                .distinctUntilChanged()
                .subscribe(new Action1<String>() {
                    @Override
                    public void call(String s) {
                        System.out.println( "distinctUntilChanged--" + s );
                    }
                }) ;
```

```
//执行结果；
1，2，1，3，4，2，1
```


### Debounce
（一段时间内没有变化，就会发送一个数据。）

场景：联想搜索的问题或者
```
// 控件绑定
        EditText ed;
        TextView tv;
        ed = (EditText) findViewById(R.id.ed);
        tv = (TextView) findViewById(R.id.tv);

         /*
         * 说明
         * 1. 此处采用了RxBinding：RxTextView.textChanges(name) = 对对控件数据变更进行监听（功能类似TextWatcher），需要引入依赖：compile 'com.jakewharton.rxbinding2:rxbinding:2.0.0'
         * 2. 传入EditText控件，输入字符时都会发送数据事件（此处不会马上发送，因为使用了debounce（））
         * 3. 采用skip(1)原因：跳过 第1次请求 = 初始输入框的空字符状态
         **/
        RxTextView.textChanges(ed)
                .debounce(1, TimeUnit.SECONDS).skip(1)
                .observeOn(AndroidSchedulers.mainThread())
                .subscribe(new Observer<CharSequence>() {
                    @Override
                    public void onSubscribe(Disposable d) {

                    }
                    @Override
                    public void onNext(CharSequence charSequence) {
                        tv.setText("发送给服务器的字符 = " + charSequence.toString());
                    }

                    @Override
                    public void onError(Throwable e) {
                        Log.d(TAG, "对Error事件作出响应" );

                    }

                    @Override
                    public void onComplete() {
                        Log.d(TAG, "对Complete事件作出响应");
                    }
                });
```


![image](http://upload-images.jianshu.io/upload_images/944365-e691a4550597d7c7.gif?imageMogr2/auto-orient/strip)


























==使用场景==：使用场景：百度关键词联想提示。在输入的过程中是不会从服务器拉数据的。当输入结束后，在400毫秒没有输入就会去获取数据。避免了，多次请求给服务器带来的压力.


>### **组合操作符**

### merge(Observable, Observable)
无序的

将两个Observable发射的事件序列组合并成一个事件序列，就像是一个Observable发射的一样。你可以简单的将它理解为两个Obsrvable合并成了一个Observable，合并后的数据是无序的。
==注意：是无序的==

```
String[] letters = new String[]{"A", "B", "C", "D", "E", "F", "G", "H"};
Observable<String> letterSequence = Observable.interval(300, TimeUnit.MILLISECONDS)
        .map(new Func1<Long, String>() {
            @Override
            public String call(Long position) {
                return letters[position.intValue()];
            }
        }).take(letters.length);

Observable<Long> numberSequence = Observable.interval(500, TimeUnit.MILLISECONDS).take(5);

Observable.merge(letterSequence, numberSequence)
        .subscribe(new Observer<Serializable>() {
            @Override
            public void onCompleted() {
                System.exit(0);
            }

            @Override
            public void onError(Throwable e) {
                System.out.println("Error:" + e.getMessage());
            }

            @Override
            public void onNext(Serializable serializable) {
                System.out.print(serializable.toString()+" ");
            }
        });


```

```
//执行结果
A 0 B C 1 D E 2 F 3 G H 4 
```




### StartWith
开始之前插入数据

1.插入普通数据    2.插入Observable对象

```
//插入普通数据
//startWith 数据序列的开头插入一条指定的项 , 最多插入9条数据
Observable observable = Observable.just( "aa" , "bb" , "cc" ) ;
observable
        .startWith( "11" , "22" )
        .subscribe(new Action1() {
            @Override
            public void call(Object o) {
                System.out.println( "startWith-- " + o );
            }
        }) ;
 
//插入Observable对象
List<String> list = new ArrayList<>() ;
list.add( "ww" ) ;
list.add( "tt" ) ;
observable.startWith( Observable.from( list ))
        .subscribe(new Action1() {
            @Override
            public void call(Object o) {
                System.out.println( "startWith2 -- " + o );
            }
        }) ;

```

```
//执行结果
startWith--11
startWith--22
startWith--aa
startWith--bb
startWith--cc


startWith2--ww
startWith2--tt
startWith2--aa
startWith2--bb
startWith2--cc

```






### Concat
将多个obserbavle发射的的数据进行合并发射（有序的）

==注==：它和merge、startWitch和相似，不同之处在于：merge:合并后发射的数据是无序的；startWitch:只能在源Observable发射的数据前插入数据。


```
String[] letters = new String[]{"A", "B", "C", "D", "E", "F", "G", "H"};
Observable<String> letterSequence = Observable.interval(300, TimeUnit.MILLISECONDS)
        .map(new Func1<Long, String>() {
            @Override
            public String call(Long position) {
                return letters[position.intValue()];
            }
        }).take(letters.length);

Observable<Long> numberSequence = Observable.interval(500, TimeUnit.MILLISECONDS).take(5);

Observable.concat(letterSequence, numberSequence)
        .subscribe(new Observer<Serializable>() {
            @Override
            public void onCompleted() {
                System.exit(0);
            }

            @Override
            public void onError(Throwable e) {
                System.out.println("Error:" + e.getMessage());
            }

            @Override
            public void onNext(Serializable serializable) {
                System.out.print(serializable.toString() + " ");
            }
        });


```

```
//执行结果
A B C D E F G H 0 1 2 3 4 
```


### zip 
合并多个观察对象的数据。Func2（）函数重新发送合并后的数据

==注意：==

- 当其中一个Observable发送数据结束或者出现异常后，另一个Observable也将停在发射数据。
- 合并两个的观察对象数据项应该是相等的；如果出现了数据项不等的情况，合并的数据项以最小数据队列为准


```
String[] letters = new String[]{"A", "B", "C", "D", "E", "F", "G", "H"};
Observable<String> letterSequence = Observable.interval(120, TimeUnit.MILLISECONDS)
        .map(new Func1<Long, String>() {
            @Override
            public String call(Long position) {
                return letters[position.intValue()];
            }
        }).take(letters.length);

Observable<Long> numberSequence = Observable.interval(200, TimeUnit.MILLISECONDS).take(5);

Observable.zip(letterSequence, numberSequence, new Func2<String, Long, String>() {
    @Override
    public String call(String letter, Long number) {
        return letter + number;
    }
}).subscribe(new Observer<String>() {
    @Override
    public void onCompleted() {
        System.exit(0);
    }

    @Override
    public void onError(Throwable e) {
        System.out.println("Error:" + e.getMessage());
    }

    @Override
    public void onNext(String result) {
        System.out.print(result + " ");
    }
});


```

```
//执行结果
A0 B1 C2 D3 E4
```



>### **其他操作符**
### 延时操作符
- delay操作符，延迟数据发送

```
Observable<String> observable = Observable.just( "1" , "2" , "3" , "4" , "5" , "6" , "7" , "8" ) ;
 
       //延迟数据发射的时间，仅仅延时一次，也就是发射第一个数据前延时。发射后面的数据不延时
       observable.delay( 3 , TimeUnit.SECONDS )  //延迟3秒钟
               .subscribe(new Action1() {
                   @Override
                   public void call(Object o) {
                       System.out.println("delay-- " + o);
                   }
               }) ;
```
- Timer  延时操作符的使用

>  使用场景：xx秒后，执行xx      
 
```
//5秒后输出 hello world , 然后显示一张图片
        Observable.timer( 5 , TimeUnit.SECONDS )
                .observeOn(AndroidSchedulers.mainThread() )
                .subscribe(new Action1<Long>() {
                    @Override
                    public void call(Long aLong) {
                        System.out.println( "timer--hello world " +  aLong );
                        findViewById( R.id.image).setVisibility(View.VISIBLE );
                    }
                }) ;
```
==总结==
delay 、timer 总结：　

- [ ]  相同点：delay 、 timer 都是延时操作符。
- [ ]  不同点：delay  延时一次，延时完成后，可以连续发射多个数据。timer延时一次，延时完成后，只发射一次数据。


### 功能防抖操作符：throttleFirst
在一段时间内，只取第一个事件，然后其他事件都丢弃。

==使用场景:==

1、button按钮防抖操作，防连续点击  
2 百度关键词联想，在一段时间内只联想一次，防止频繁请求服务器   

```
 Observable.interval( 1 , TimeUnit.SECONDS)
                .throttleFirst( 3 , TimeUnit.SECONDS )
                .subscribe(new Action1<Long>() {
                    @Override
                    public void call(Long aLong) {
                        System.out.println( "throttleFirst--" + aLong );
                    }
                }) ;
```

```
注：这段代码，是循环发送数据，每秒发送一个。throttleFirst( 3 , TimeUnit.SECONDS )   在3秒内只取第一个事件，其他的事件丢弃。
```


```

//执行结果：
0：第一组前3s[0,1,2] 只取第一个事件0
3
6
9
12
15
.。。。。
```



### 网络错误重试操作符  retrywhen
![网络请求重试](http://upload-images.jianshu.io/upload_images/944365-fd347a0ba6a54876.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



```
public class RxJavafixRetrofit2 extends AppCompatActivity {

    private static final String TAG = "RxJava";

    // 设置变量
    // 可重试次数
    private int maxConnectCount = 10;
    // 当前已重试次数
    private int currentRetryCount = 0;
    // 重试等待时间
    private int waitRetryTime = 0;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        // 步骤1：创建Retrofit对象
        Retrofit retrofit = new Retrofit.Builder()
                .baseUrl("http://fy.iciba.com/") // 设置 网络请求 Url
                .addConverterFactory(GsonConverterFactory.create()) //设置使用Gson解析(记得加入依赖)
                .addCallAdapterFactory(RxJava2CallAdapterFactory.create()) // 支持RxJava
                .build();

        // 步骤2：创建 网络请求接口 的实例
        GetRequest_Interface request = retrofit.create(GetRequest_Interface.class);

        // 步骤3：采用Observable<...>形式 对 网络请求 进行封装
        Observable<Translation> observable = request.getCall();

        // 步骤4：发送网络请求 & 通过retryWhen（）进行重试
        // 注：主要异常才会回调retryWhen（）进行重试
        observable.retryWhen(new Function<Observable<Throwable>, ObservableSource<?>>() {
            @Override
            public ObservableSource<?> apply(@NonNull Observable<Throwable> throwableObservable) throws Exception {
                // 参数Observable<Throwable>中的泛型 = 上游操作符抛出的异常，可通过该条件来判断异常的类型
                return throwableObservable.flatMap(new Function<Throwable, ObservableSource<?>>() {
                    @Override
                    public ObservableSource<?> apply(@NonNull Throwable throwable) throws Exception {

                        // 输出异常信息
                        Log.d(TAG,  "发生异常 = "+ throwable.toString());

                        /**
                         * 需求1：根据异常类型选择是否重试
                         * 即，当发生的异常 = 网络异常 = IO异常 才选择重试
                         */
                        if (throwable instanceof IOException){

                            Log.d(TAG,  "属于IO异常，需重试" );

                            /**
                             * 需求2：限制重试次数
                             * 即，当已重试次数 < 设置的重试次数，才选择重试
                             */
                            if (currentRetryCount < maxConnectCount){

                                // 记录重试次数
                                currentRetryCount++;
                                Log.d(TAG,  "重试次数 = " + currentRetryCount);

                                /**
                                 * 需求2：实现重试
                                 * 通过返回的Observable发送的事件 = Next事件，从而使得retryWhen（）重订阅，最终实现重试功能
                                 *
                                 * 需求3：延迟1段时间再重试
                                 * 采用delay操作符 = 延迟一段时间发送，以实现重试间隔设置
                                 *
                                 * 需求4：遇到的异常越多，时间越长
                                 * 在delay操作符的等待时间内设置 = 每重试1次，增多延迟重试时间1s
                                 */
                                // 设置等待时间
                                waitRetryTime = 1000 + currentRetryCount* 1000;
                                Log.d(TAG,  "等待时间 =" + waitRetryTime);
                                return Observable.just(1).delay(waitRetryTime, TimeUnit.MILLISECONDS);


                            }else{
                                // 若重试次数已 > 设置重试次数，则不重试
                                // 通过发送error来停止重试（可在观察者的onError（）中获取信息）
                                return Observable.error(new Throwable("重试次数已超过设置次数 = " +currentRetryCount  + "，即 不再重试"));

                            }
                        }

                        // 若发生的异常不属于I/O异常，则不重试
                        // 通过返回的Observable发送的事件 = Error事件 实现（可在观察者的onError（）中获取信息）
                        else{
                            return Observable.error(new Throwable("发生了非网络异常（非I/O异常）"));
                        }
                    }
                });
            }
        }).subscribeOn(Schedulers.io())               // 切换到IO线程进行网络请求
                .observeOn(AndroidSchedulers.mainThread())  // 切换回到主线程 处理请求结果
                .subscribe(new Observer<Translation>() {
                    @Override
                    public void onSubscribe(Disposable d) {
                    }

                    @Override
                    public void onNext(Translation result) {
                        // 接收服务器返回的数据
                        Log.d(TAG,  "发送成功");
                        result.show();
                    }

                    @Override
                    public void onError(Throwable e) {
                        // 获取停止重试的信息
                        Log.d(TAG,  e.toString());
                    }

                    @Override
                    public void onComplete() {

                    }
                });

    }
}
```


测试结果
-  一开始先通过 断开网络连接 模拟 网络异常错误，即开始重试；
-  等到第3次重试后恢复网络连接，即无发生网络异常错误，此时重试成功
-  

![测试结果](http://upload-images.jianshu.io/upload_images/944365-b265226c3eb8edfe.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)






### 轮询操作符 （无条件） interval/intervalRange
客户端固定事件主动向服务器获取消息
以前做法：handler/timer  发送延时消息

- 网络请求轮询（无限次轮询）interval
- 要是有限次的就吧 interval换成intervalRange

```
public class RxJavafixRxjava extends AppCompatActivity {

    private static final String TAG = "Rxjava";

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        /*
         * 步骤1：采用interval（）延迟发送
         * 注：此处主要展示无限次轮询，若要实现有限次轮询，仅需将interval（）改成intervalRange（）即可
         **/
        Observable.interval(2,1, TimeUnit.SECONDS)
                // 参数说明：
                // 参数1 = 第1次延迟时间；
                // 参数2 = 间隔时间数字；
                // 参数3 = 时间单位；
                // 该例子发送的事件特点：延迟2s后发送事件，每隔1秒产生1个数字（从0开始递增1，无限个）

                 /*
                  * 步骤2：每次发送数字前发送1次网络请求（doOnNext（）在执行Next事件前调用）
                  * 即每隔1秒产生1个数字前，就发送1次网络请求，从而实现轮询需求
                  **/
                 .doOnNext(new Consumer<Long>() {
            @Override
            public void accept(Long integer) throws Exception {
                Log.d(TAG, "第 " + integer + " 次轮询" );

                 /*
                  * 步骤3：通过Retrofit发送网络请求
                  **/
                  // a. 创建Retrofit对象
                Retrofit retrofit = new Retrofit.Builder()
                        .baseUrl("http://fy.iciba.com/") // 设置 网络请求 Url
                        .addConverterFactory(GsonConverterFactory.create()) //设置使用Gson解析(记得加入依赖)
                        .addCallAdapterFactory(RxJava2CallAdapterFactory.create()) // 支持RxJava
                        .build();

                  // b. 创建 网络请求接口 的实例
                GetRequest_Interface request = retrofit.create(GetRequest_Interface.class);

                  // c. 采用Observable<...>形式 对 网络请求 进行封装
                  Observable<Translation> observable = request.getCall();
                  // d. 通过线程切换发送网络请求
                  observable.subscribeOn(Schedulers.io())               // 切换到IO线程进行网络请求
                        .observeOn(AndroidSchedulers.mainThread())  // 切换回到主线程 处理请求结果
                        .subscribe(new Observer<Translation>() {
                            @Override
                            public void onSubscribe(Disposable d) {
                            }

                            @Override
                            public void onNext(Translation result) {
                                // e.接收服务器返回的数据
                                result.show() ;
                            }

                            @Override
                            public void onError(Throwable e) {
                                Log.d(TAG, "请求失败");
                            }

                            @Override
                            public void onComplete() {

                            }
                        });

            }
        }).subscribe(new Observer<Long>() {
            @Override
            public void onSubscribe(Disposable d) {

            }
            @Override
            public void onNext(Long value) {

            }

            @Override
            public void onError(Throwable e) {
                Log.d(TAG, "对Error事件作出响应");
            }

            @Override
            public void onComplete() {
                Log.d(TAG, "对Complete事件作出响应");
            }
        });


    }
}


```
![无限制轮询结果](https://upload-images.jianshu.io/upload_images/944365-2ebf24e19861a453.gif?imageMogr2/auto-orient/)




### 从磁盘 / 内存缓存中 获取缓存数据
![image](http://upload-images.jianshu.io/upload_images/944365-ac1bc26fa511c836.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


==//注：用到过滤firstElement（）和组合操作符 concat（）==


```
// 该2变量用于模拟内存缓存 & 磁盘缓存中的数据
        String memoryCache = null;
        String diskCache = "从磁盘缓存中获取数据";

        /*
         * 设置第1个Observable：检查内存缓存是否有该数据的缓存
         **/
        Observable<String> memory = Observable.create(new ObservableOnSubscribe<String>() {
            @Override
            public void subscribe(ObservableEmitter<String> emitter) throws Exception {

                // 先判断内存缓存有无数据
                if (memoryCache != null) {
                    // 若有该数据，则发送
                    emitter.onNext(memoryCache);
                } else {
                    // 若无该数据，则直接发送结束事件
                    emitter.onComplete();
                }

            }
        });

        /*
         * 设置第2个Observable：检查磁盘缓存是否有该数据的缓存
         **/
        Observable<String> disk = Observable.create(new ObservableOnSubscribe<String>() {
            @Override
            public void subscribe(ObservableEmitter<String> emitter) throws Exception {

                // 先判断磁盘缓存有无数据
                if (diskCache != null) {
                    // 若有该数据，则发送
                    emitter.onNext(diskCache);
                } else {
                    // 若无该数据，则直接发送结束事件
                    emitter.onComplete();
                }

            }
        });

        /*
         * 设置第3个Observable：通过网络获取数据
         **/
        Observable<String> network = Observable.just("从网络中获取数据");
        // 此处仅作网络请求的模拟


        /*
         * 通过concat（） 和 firstElement（）操作符实现缓存功能
         **/

        // 1. 通过concat（）合并memory、disk、network 3个被观察者的事件（即检查内存缓存、磁盘缓存 & 发送网络请求）
        //    并将它们按顺序串联成队列
        Observable.concat(memory, disk, network)
                // 2. 通过firstElement()，从串联队列中取出并发送第1个有效事件（Next事件），即依次判断检查memory、disk、network
                .firstElement()
                // 即本例的逻辑为：
                // a. firstElement()取出第1个事件 = memory，即先判断内存缓存中有无数据缓存；由于memoryCache = null，即内存缓存中无数据，所以发送结束事件（视为无效事件）
                // b. firstElement()继续取出第2个事件 = disk，即判断磁盘缓存中有无数据缓存：由于diskCache ≠ null，即磁盘缓存中有数据，所以发送Next事件（有效事件）
                // c. 即firstElement()已发出第1个有效事件（disk事件），所以停止判断。

                // 3. 观察者订阅
                .subscribe(new Consumer<String>() {
                    @Override
                    public void accept( String s) throws Exception {
                        Log.d(TAG,"最终获取的数据来源 =  "+ s);
                    }
                });
```

//测试结果
![image](http://upload-images.jianshu.io/upload_images/944365-f745dbb1810ed00b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



### 联合判断操作符--combineLatest

```
  /*
         * 步骤1：设置控件变量 & 绑定
         **/
        EditText name,age,job;
        Button list;

        name = (EditText) findViewById(R.id.name);
        age = (EditText) findViewById(R.id.age);
        job = (EditText) findViewById(R.id.job);
        list = (Button) findViewById(R.id.list);

        /*
         * 步骤2：为每个EditText设置被观察者，用于发送监听事件
         * 说明：
         * 1. 此处采用了RxBinding：RxTextView.textChanges(name) = 对对控件数据变更进行监听（功能类似TextWatcher），需要引入依赖：compile 'com.jakewharton.rxbinding2:rxbinding:2.0.0'
         * 2. 传入EditText控件，点击任1个EditText撰写时，都会发送数据事件 = Function3（）的返回值（下面会详细说明）
         * 3. 采用skip(1)原因：跳过 一开始EditText无任何输入时的空值
         **/
        Observable<CharSequence> nameObservable = RxTextView.textChanges(name).skip(1);
        Observable<CharSequence> ageObservable = RxTextView.textChanges(age).skip(1);
        Observable<CharSequence> jobObservable = RxTextView.textChanges(job).skip(1);

        /*
         * 步骤3：通过combineLatest（）合并事件 & 联合判断
         **/
        Observable.combineLatest(nameObservable,ageObservable,jobObservable,new Function3<CharSequence, CharSequence, CharSequence,Boolean>() {
            @Override
            public Boolean apply(@NonNull CharSequence charSequence, @NonNull CharSequence charSequence2, @NonNull CharSequence charSequence3) throws Exception {

                /*
                 * 步骤4：规定表单信息输入不能为空
                 **/
                // 1. 姓名信息
                boolean isUserNameValid = !TextUtils.isEmpty(name.getText()) ;
                // 除了设置为空，也可设置长度限制
                // boolean isUserNameValid = !TextUtils.isEmpty(name.getText()) && (name.getText().toString().length() > 2 && name.getText().toString().length() < 9);

                // 2. 年龄信息
                boolean isUserAgeValid = !TextUtils.isEmpty(age.getText());
                // 3. 职业信息
                boolean isUserJobValid = !TextUtils.isEmpty(job.getText()) ;

                /*
                 * 步骤5：返回信息 = 联合判断，即3个信息同时已填写，"提交按钮"才可点击
                 **/
                return isUserNameValid && isUserAgeValid && isUserJobValid;
            }

                }).subscribe(new Consumer<Boolean>() {
            @Override
            public void accept(Boolean s) throws Exception {
                /*
                 * 步骤6：返回结果 & 设置按钮可点击样式
                 **/
                Log.e(TAG, "提交按钮是否可点击： "+s);
                list.setEnabled(s);
            }
        });
```
![image](http://upload-images.jianshu.io/upload_images/944365-0aa22c498d5f84f7.gif?imageMogr2/auto-orient/strip)

### 发送事件之前的初始化操作符doOnSubscribe() 

使用场景： 可以在事件发出之前做一些初始化的工作，比如弹出进度条等等

 ==注意：==
1.  doOnSubscribe() 默认运行在事件产生的线程里面，然而事件产生的线程一般都会运行在 io 线程里。那么这个时候做一些，更新UI的操作，是线程不安全的。所以如果事件产生的线程是io线程，但是我们又要在doOnSubscribe() 更新UI ， 这时候就需要线程切换。
2.  如果在 doOnSubscribe() 之后有 subscribeOn() 的话，它将执行在离它最近的 subscribeOn() 所指定的线程。   
3.   subscribeOn() 事件产生的线程 ； observeOn() : 事件消费的线程

```
 Observable.create(onSubscribe)
    .subscribeOn(Schedulers.io())
    .doOnSubscribe(new Action0() {
        @Override
        public void call() {
            progressBar.setVisibility(View.VISIBLE); // 需要在主线程执行
        }
    })
    .subscribeOn(AndroidSchedulers.mainThread()) // 指定主线程
    .observeOn(AndroidSchedulers.mainThread())
    .subscribe(subscriber);
```
### doOnNext() 操作符，在每次 OnNext() 方法被调用前执行

 使用场景：从网络请求数据，在数据被展示前，缓存到本地


```
Observable observable = Observable.just( "1" , "2" , "3" , "4" ) ;<br>
       observable.doOnNext(new Action1() {
           @Override
           public void call(Object o) {
               System.out.println( "doOnNext--缓存数据" + o  );
           }
       })
               .subscribe(new Observer() {
                   @Override
                   public void onCompleted() {
 
                   }
 
                   @Override
                   public void onError(Throwable e) {
 
                   }
 
                   @Override
                   public void onNext(Object o) {
                       System.out.println( "onNext--" + o  );
                   }
               }) ;
```

![image](https://images2015.cnblogs.com/blog/605655/201605/605655-20160523144746131-1044437687.png)


### Buffer 操作符
 使用场景：一个按钮每点击3次，弹出一个toast      
 
```
List<String> list = new ArrayList<>();
      for (int i = 1; i < 10; i++) {
          list.add("" + i);
      }
 
      Observable<String> observable = Observable.from(list);
      observable
              .buffer(2)   //把每两个数据为一组打成一个包，然后发送
              .subscribe(new Action1<List<String>>() {
                  @Override
                  public void call(List<String> strings) {
                      System.out.println( "buffer---------------" );
                      Observable.from( strings ).subscribe(new Action1<String>() {
                          @Override
                          public void call(String s) {
                              System.out.println( "buffer data --" + s  );
                          }
                      }) ;
                  }
              });
```
![image](https://images2015.cnblogs.com/blog/605655/201605/605655-20160523164020538-1136364753.png)

```
//第1、2 个数据打成一个数据包，跳过第三个数据 ； 第4、5个数据打成一个包，跳过第6个数据
       observable.buffer( 2 , 3 )  //把每两个数据为一组打成一个包，然后发送。第三个数据跳过去
               .subscribe(new Action1<List<String>>() {
                   @Override
                   public void call(List<String> strings) {
 
                       System.out.println( "buffer22---------------" );
                       Observable.from( strings ).subscribe(new Action1<String>() {
                           @Override
                           public void call(String s) {
                               System.out.println( "buffer22 data --" + s  );
                           }
                       }) ;
                   }
               }) ;
```
![image](https://images2015.cnblogs.com/blog/605655/201605/605655-20160523165927631-165770077.png)

==注：以上的所有操作符只是个人认为比较重要的一部分。还有很多其他的。不一一概述！==



















































### 消费事件（观察者消费事件）subscribe()
- 创建观察者方式1

```
Observer<String> observer = new Observer<String>() {
    @Override
    public void onNext(String s) {
        Log.d(tag, "Item: " + s);
    }

    @Override
    public void onCompleted() {
        Log.d(tag, "Completed!");
    }

    @Override
    public void onError(Throwable e) {
        Log.d(tag, "Error!");
    }
};
```

- 创建观察者方式2：多一个回调方法onStart（）

```
Subscriber<String> subscriber = new Subscriber<String>() {
    @Override
    public void onNext(String s) {
        Log.d(tag, "Item: " + s);
    }

    @Override
    public void onCompleted() {
        Log.d(tag, "Completed!");
    }

    @Override
    public void onError(Throwable e) {
        Log.d(tag, "Error!");
    }
};

```


 ==注释==：onstart():是在事件发送之前做一些准备工作的。他工作的线程取决于发送事件的线程。所以要做进度条更新的初始化准备工作根本不适合。 要想指定线程来做准备工作，用doOnSubscribe()，具体使用看其他里面的操作符。
 
- 观察者和被观察者联系起来

```
observable.subscribe(observer);// 或者：observable.subscribe(subscriber);

事件就在subscribe()的回调中消费了
```
//subscribe()的核心源码


```
// 注意：这不是 subscribe() 的源码，而是将源码中与性能、兼容性、扩展性有关的代码剔除后的核心代码。
// 如果需要看源码，可以去 RxJava 的 GitHub 仓库下载。
public Subscription subscribe(Subscriber subscriber) {
    subscriber.onStart();
    onSubscribe.call(subscriber);
    return subscriber;
}
```
==//注意：==

由此可见被观察者创建时候并不会立刻发送事件，只有subscribe()执行的时候才会发送事件，执行被观察者的call();在发送事件之前会执行onstart()做准备工作

---


## Rxjava实现组件间通信RxBus
- 没有实现背压的RxBus

```
public class RxBus {

    private final Subject<Object> mBus;

    private RxBus() {
        // toSerialized method made bus thread safe
        mBus = PublishSubject.create().toSerialized();
    }

    public static RxBus get() {
        return Holder.BUS;
    }

    public void post(Object obj) {
        mBus.onNext(obj);
    }

    public <T> Observable<T> toObservable(Class<T> tClass) {
        return mBus.ofType(tClass);
    }

    public Observable<Object> toObservable() {
        return mBus;
    }

    public boolean hasObservers() {
        return mBus.hasObservers();
    }

    private static class Holder {
        private static final RxBus BUS = new RxBus();
    }
}
```
- 有实现背压的RxBus

```
public class RxBus {

    private final FlowableProcessor<Object> mBus;

    private RxBus() {
        // toSerialized method made bus thread safe
        mBus = PublishProcessor.create().toSerialized();
    }

    public static RxBus get() {
        return Holder.BUS;
    }

    public void post(Object obj) {
        mBus.onNext(obj);
    }

    public <T> Flowable<T> toFlowable(Class<T> tClass) {
        return mBus.ofType(tClass);
    }

    public Flowable<Object> toFlowable() {
        return mBus;
    }

    public boolean hasSubscribers() {
        return mBus.hasSubscribers();
    }

    private static class Holder {
        private static final RxBus BUS = new RxBus();
    }
}
```
==//注：==

如果你不了解背压的意思，那就请看最后的部分：Rxjava 2.0背压实现


---

## Rxjava 线程切换
1. 切换的使用套路（会用是根本）

  ==总结出来就这几句话==
- 事件产生：默认运行在当前线程，可以由 subscribeOn()  自定义线程，但是只有第一个subscribeOn()起作用呦！


- 事件加工：默认跟事件产生的线程保持一致, 可以由 observeOn() 自定义线程，离他最近最上边的observeOn()起作用。


- 事件消费：默认运行在当前线程，可以有observeOn() 自定义，，离他最近最上边的observeOn()起作用。

==特别注意==：doOnSubscribe() 之后有 subscribeOn() 的话，它将执行在离它最近的 subscribeOn() 所指定的线程。


2. 线程切换的原理----待补充

---
## Rxjava 控制生命周期
Rxjava生命周期框架+compose操作符

依赖：compile 'com.trello:rxlifecycle-components:0.6.1'

- 让你的activity继承RxActivity,RxAppCompatActivity,RxFragmentActivity


- 让你的fragment继承RxFragment,RxDialogFragment

==//示例：activity和fragmnet 中的用法大同小异==

1. @bindToLifecycle  默认和组件的ondestroy()生命周期方法中解除绑定

```
 public class MainActivity extends RxAppCompatActivity {
        TextView textView ;
        
        @Override
        protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        textView = (TextView) findViewById(R.id.textView);
    
        //循环发送数字
        Observable.interval(0, 1, TimeUnit.SECONDS)
            .subscribeOn( Schedulers.io())
            .compose(this.<Long>bindToLifecycle())   //这个订阅关系跟Activity绑定，Observable 和activity生命周期同步
            .observeOn( AndroidSchedulers.mainThread())
            .subscribe(new Action1<Long>() {
                @Override
                public void call(Long aLong) {
                    System.out.println("lifecycle--" + aLong);
                    textView.setText( "" + aLong );
                }
            });
       }
    }
```


2. @bindUntilEvent( ActivityEvent event)   可以指定和组件的哪个生命周期方法中接触绑定

```
//循环发送数字
     Observable.interval(0, 1, TimeUnit.SECONDS)
             .subscribeOn( Schedulers.io())
             .compose(this.<Long>bindUntilEvent(ActivityEvent.STOP ))   //当Activity执行Onstop()方法是解除订阅关系
             .observeOn( AndroidSchedulers.mainThread())
             .subscribe(new Action1<Long>() {
                 @Override
                 public void call(Long aLong) {
                     System.out.println("lifecycle-stop-" + aLong);
                     textView.setText( "" + aLong );
                 }
             });
```




---
## Rxjava 2.0背压实现
啥是背压？简单一句话：

==背压是因为被观察者的发送事件的速度>观察者处理事件的速度==

直接上我认为很好的学习链接：

[Rxjava 2.0背压实现学习资料1](https://mp.weixin.qq.com/s/JiK4jEZNpq3k_hxFPrx4gw)

[Rxjava 2.0背压实现学习资料2](http://blog.csdn.net/carson_ho/article/details/79081407)




---
友情提供好的学习资源链接：

[Rxjava学习资源1](http://www.cnblogs.com/zhaoyanjun/p/5535651.html)


[Rxjava学习资源2----扔物线](https://blog.csdn.net/u011402187/article/details/50374455)


源码：GItHub

[Rxjava](https://github.com/ReactiveX/RxJava)

[RxAndroid](https://github.com/ReactiveX/RxAndroid)


