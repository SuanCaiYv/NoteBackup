哈哈哈哈哈，题目有点猖狂。但是既然你都来了，那就看看吧，毕竟响应式编程随着高并发对于性能的吃紧，越来越重要了。

哦对了，这是一篇Java文章。

废话不多说，直接步入正题。

## 响应式编程核心组件
步入正题之前，我希望你对发布者/订阅者模型有一些了解。

直接看图：
![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/69188e8a9a054e498c4d7cc53b56d497~tplv-k3u1fbpfcp-watermark.image)

Talk is cheap, show you the code!
``` Java
public class Main {

    public static void main(String[] args) {
        Flux<Integer> flux = Flux.range(0, 10);
        flux.subscribe(i -> {
            System.out.println("run1: " + i);
        });
        flux.subscribe(i -> {
            System.out.println("run2: " + i);
        });
    }
}
```

输出：
```
run1: 0
run1: 1
run1: 2
run1: 3
run1: 4
run1: 5
run1: 6
run1: 7
run1: 8
run1: 9
run2: 0
run2: 1
run2: 2
run2: 3
run2: 4
run2: 5
run2: 6
run2: 7
run2: 8
run2: 9

Process finished with exit code 0
```
### Flux
Flux是一个多元素的生产者，言外之意，它可以生产多个元素，组成元素序列，供订阅者使用。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fb9b8c59906c4942ab0e137272fd1fb0~tplv-k3u1fbpfcp-watermark.image)
### Mono
Mono和Flux的区别在于，它只能生产一个元素供生产者订阅，也就是数量的不同。

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9fd08d8a5eac4cf1b9afb0af7267aa6f~tplv-k3u1fbpfcp-watermark.image)

Mono的一个常见的应用就是Mono<ServerResponse\>作为WebFlux的返回值。毕竟每次请求只有一个Response对象，所以Mono刚刚好。
## 快速创建一个Flux/Mono并订阅它

来看一些官方文档演示的方法。

``` Java
Flux<String> seq1 = Flux.just("foo", "bar", "foobar");

List<String> iterable = Arrays.asList("foo", "bar", "foobar");
Flux<String> seq2 = Flux.fromIterable(iterable);

Mono<String> noData = Mono.empty();

Mono<String> data = Mono.just("foo");

Flux<Integer> numbersFromFiveToSeven = Flux.range(5, 3);
```
### subscribe()方法(Lambda形式)

- subscribe()方法默认接受一个Lambda表达式作为订阅者来使用。它有四个变种形式。
- 在这里说明一下subscribe()第四个参数，指出了当订阅信号到达，初次请求的个数，如果是null则全部请求(Long.MAX_VALUE)

``` Java
public class FluxIntegerWithSubscribe {

    public static void main(String[] args) {
        Flux<Integer> integerFlux = Flux.range(0, 10);
        integerFlux.subscribe(i -> {
            System.out.println("run");
            System.out.println(i);
        }, error -> {
            System.out.println("error");
        }, () -> {
            System.out.println("done");
        }, p -> {
            p.request(2);
        });
    }
}
```
如果去掉初次请求，那么会请求最大值：
``` Java
public class FluxIntegerWithSubscribe {

    public static void main(String[] args) {
        Flux<Integer> integerFlux = Flux.range(0, 10);
        // 在这里说明一下subscribe()第四个参数，指出了当订阅信号到达，初次请求的个数，如果是null则全部请求(Long.MAX_VALUE)
        // 其余subscribe()详见源码或文档:https://projectreactor.io/docs/core/release/reference/#flux
        integerFlux.subscribe(i -> {
            System.out.println("run");
            System.out.println(i);
        }, error -> {
            System.out.println("error");
        }, () -> {
            System.out.println("done");
        });
    }
}
```
输出：
```
run
0
run
1
run
2
run
3
run
4
run
5
run
6
run
7
run
8
run
9
done

Process finished with exit code 0
```
### 继承BaseSubscriber(非Lambda形式)

- 这种方式更多像是对于Lambda表达式的一种替换表达。
- 对于基于此方法的订阅，有一些注意事项，比如初次订阅时，要至少请求一次。否则会导致程序无法继续获得新的元素。

``` Java
public class FluxWithBaseSubscriber {

    public static void main(String[] args) {
        Flux<Integer> integerFlux = Flux.range(0, 10);
        integerFlux.subscribe(new MySubscriber());
    }

    /**
     * 一般来说，通过继承BaseSubscriber<T>来实现，而且一般自定义hookOnSubscribe()和hookOnNext()方法
     */
    private static class MySubscriber extends BaseSubscriber<Integer> {

        /**
         * 初次订阅时被调用
         */
        @Override
        protected void hookOnSubscribe(Subscription subscription) {
            System.out.println("开始啦！");
            // 记得至少请求一次，否则不会执行hookOnNext()方法
            request(1);
        }

        /**
         * 每次读取新值调用
         */
        @Override
        protected void hookOnNext(Integer value) {
            System.out.println("开始读取...");
            System.out.println(value);
            // 指出下一次读取多少个
            request(2);
        }

        @Override
        protected void hookOnComplete() {
            System.out.println("结束啦");
        }
    }
}
```
输出：
``` Java
开始啦！
开始读取...
0
开始读取...
1
开始读取...
2
开始读取...
3
开始读取...
4
开始读取...
5
开始读取...
6
开始读取...
7
开始读取...
8
开始读取...
9
结束啦

Process finished with exit code 0
```
### 终止订阅:Disposable

- Disposable是一个订阅时返回的接口，里面包含很多可以操作订阅的方法。
- 比如取消订阅。

在这里使用多线程模拟生产者生产的很快，然后立马取消订阅（虽然立刻取消但是由于生产者实在太快了，所以订阅者还是接收到了一些元素）。

其他的方法，比如Disposables.composite()会得到一个Disposable的集合，调用它的dispose()方法会把集合里的所有Disposable的dispose()方法都调用。

``` Java
public class FluxWithDisposable {

    public static void main(String[] args) {
        Disposable disposable = getDis();
        // 每次打印数量一般不同，因为调用了disposable的dispose()方法进行了取消，不过如果生产者产地太快了，那么可能来不及终止。
        disposable.dispose();
    }

    private static Disposable getDis() {
        class Add implements Runnable {

            private final FluxSink<Integer> fluxSink;

            public Add(FluxSink<Integer> fluxSink) {
                this.fluxSink = fluxSink;
            }

            @Override
            public synchronized void run() {
                fluxSink.next(new Random().nextInt());
            }
        }
        Flux<Integer> integerFlux = Flux.create(integerFluxSink -> {
            Add add = new Add(integerFluxSink);
            new Thread(add).start();
            new Thread(add).start();
            new Thread(add).start();
            new Thread(add).start();
            new Thread(add).start();
            new Thread(add).start();
            new Thread(add).start();
            new Thread(add).start();
            new Thread(add).start();
            new Thread(add).start();
            new Thread(add).start();
        });
        return integerFlux.subscribe(System.out::println);
    }
}
```
输出：
```
这里的输出每次调用可能都会不同，因为订阅之后取消了，所以能打印多少取决于那一瞬间CPU的速度。
```
### 调整发布者发布速率

- 为了缓解订阅者压力，订阅者可以通过负压流回溯进行重塑发布者发布的速率。最典型的用法就是下面这个——通过继承BaseSubscriber来设置自己的请求速率。但是有一点必须明确，就是hookOnSubscribe()方法必须至少请求一次，不然你的发布者可能会“卡住”。

``` Java
public class FluxWithLimitRate1 {

    public static void main(String[] args) {
        Flux<Integer> integerFlux = Flux.range(0, 100);
        integerFlux.subscribe(new MySubscriber());
    }

    private static class MySubscriber extends BaseSubscriber<Integer> {

        @Override
        protected void hookOnSubscribe(Subscription subscription) {
            System.out.println("开始啦！");
            // 记得至少请求一次，否则不会执行hookOnNext()方法
            request(1);
        }

        @Override
        protected void hookOnNext(Integer value) {
            System.out.println("开始读取...");
            System.out.println(value);
            // 指出下一次读取多少个
            request(2);
        }

        @Override
        protected void hookOnComplete() {
            System.out.println("结束啦！");
        }
    }
}
```

- 或者使用limitRate()实例方法进行限制，它返回一个被限制了速率的Flux或Mono。某些上流的操作可以更改下流订阅者的请求速率，有一些操作有一个prefetch整型作为输入，可以获取大于下流订阅者请求的数量的序列元素，这样做是为了处理它们自己的内部序列。这些预获取的操作方法一般默认预获取32个，不过为了优化；每次已经获取了预获取数量的75%的时候，会再获取75%。这叫“补充优化”。

``` Java
public class FluxWithLimitRate2 {

    public static void main(String[] args) {
        Flux<Integer> integerFlux = Flux.range(0, 100);
        // 最后，来看一些Flux提供的预获取方法：
        // 指出预取数量
        integerFlux.limitRate(10);
        // lowTide指出预获取操作的补充优化的值，即修改75%的默认值；highTide指出预获取数量。
        integerFlux.limitRate(10, 15);
        // 哎～最典型的就是，请求无数:request(Long.MAX_VALUE)但是我给你limitRate(2)；那你也只能乖乖每次得到两个哈哈哈哈！
        // 还有一个就是limitRequest(N)，它会把下流总请求限制为N。如果下流请求超过了N，那么只返回N个，否则返回实际数量。然后认为请求完成，向下流发送onComplete信号。
        integerFlux.limitRequest(5).subscribe(new MySubscriber());
        // 上面这个只会输出5个。
    }
}
```
## 程序化地创建一个序列
### 静态同步方法：generate()

现在到了程序化生成Flux/Mono的时候。首先介绍generate()方法，这是一个同步的方法。言外之意就是，它是线程不安全的，且它的接收器只能一次一个的接受输入来生成Flux/Mono。也就是说，它在任意时刻只能被调用一次且只接受一个输入。

或者这么说，它生成的元素序列的顺序，取决于代码编写的方式。

``` Java
public class FluxWithGenerate {

    public static void main(String[] args) {
        // 下面这个是它的变种方法之一：第一个参数是提供初始状态的，第二个参数是一个向接收器写入数据的生成器，入参为state(一般为整数，用来记录状态)，和接收器。
        // 其他变种请看源码
        Flux.generate(() -> 0, (state, sink) -> {
            sink.next(state+"asdf");
            // 加上对于sink.complete()的调用即可终止生成；否则就是无限序列。
            return state+1;
        }).subscribe(System.out::println);
        // generate方法的第三个参数用于结束生成时被调用，消耗state。
        Flux.generate(AtomicInteger::new, (state, sink) -> {
            sink.next(state.getAndIncrement()+"qwer");
            return state;
        }).subscribe(System.out::println);
        // generate()的工作流看起来就像:next()->next()->next()->...
    }
}
```

- 通过上述代码不难看到，每次的接收器接受的值来自于上一次生成方法的返回值，也就是state=上一个迭代的返回值(其实称为上一个流才准确，这么说只是为了方便理解)。

- 不过这个state每次都是一个全新的(每次都+1当然是新的)，那么有没有什么方法可以做到前后两次迭代的state是同一个引用且还可以更新值呢？答案就是原子类型。也就是上面的第二种方式。
### 静态异步多线程方法：create()

说完了同步生成，接下来就是异步生成，还是多线程的！让我们有请:create()闪亮登场！！！

- create()方法对外暴露出一个FluxSink对象，通过它我们可以访问并生成需要的序列。除此之外，它还可以触发回调中的多线程事件。

- create另一特性就是很容易把其他的接口与响应式桥接起来。注意，它是异步多线程并不意味着create可以并行化你写的代码或者异步执行；怎么理解呢？就是，create方法里面的Lambda表达式代码还是单线程阻塞的。如果你在创建序列的地方阻塞了代码，那么可能造成订阅者即使请求了数据，也得不到，因为序列被阻塞了，没法生成新的。

- 其实通过上面的现象可以猜测，默认情况下订阅者使用的线程和create使用的是一个线程，当然阻塞create就会导致订阅者没法运行咯！

- 上述问题可以通过Scheduler解决，后面会提到。

``` Java
public class FluxWithCreate {

    public static void main(String[] args) throws InterruptedException {
        TestProcessor<String> testProcessor = new TestProcessor<>() {

            private TestListener<String> testListener;

            @Override
            public void register(TestListener<String> stringTestListener) {
                this.testListener = stringTestListener;
            }

            @Override
            public TestListener<String> get() {
                return testListener;
            }
        };
        Flux<String> flux = Flux.create(stringFluxSink -> testProcessor.register(new TestListener<String>() {
            @Override
            public void onChunk(List<String> chunk) {
                for (String s : chunk) {
                    stringFluxSink.next(s);
                }
            }

            @Override
            public void onComplete() {
                stringFluxSink.complete();
            }
        }));
        flux.subscribe(System.out::println);
        System.out.println("现在是2020/10/22 22:58；我好困");
        TestListener<String> testListener = testProcessor.get();
        Runnable1<String> runnable1 = new Runnable1<>() {

            private TestListener<String> testListener;

            @Override
            public void set(TestListener<String> testListener) {
                this.testListener = testListener;
            }

            @Override
            public void run() {
                List<String> list = new ArrayList<>(10);
                for (int i = 0; i < 10; ++ i) {
                    list.add(i+"-run1");
                }
                testListener.onChunk(list);
            }
        };
        Runnable1<String> runnable2 = new Runnable1<>() {

            private TestListener<String> testListener;

            @Override
            public void set(TestListener<String> testListener) {
                this.testListener = testListener;
            }

            @Override
            public void run() {
                List<String> list = new ArrayList<>(10);
                for (int i = 0; i < 10; ++ i) {
                    list.add(i+"-run2");
                }
                testListener.onChunk(list);
            }
        };
        Runnable1<String> runnable3 = new Runnable1<>() {

            private TestListener<String> testListener;

            @Override
            public void set(TestListener<String> testListener) {
                this.testListener = testListener;
            }

            @Override
            public void run() {
                List<String> list = new ArrayList<>(10);
                for (int i = 0; i < 10; ++ i) {
                    list.add(i+"-run3");
                }
                testListener.onChunk(list);
            }
        };
        runnable1.set(testListener);
        runnable2.set(testListener);
        runnable3.set(testListener);
        // create所谓的"异步"，"多线程"指的是在多线程中调用sink.next()方法。这一点在下面的push对比中可以看到
        new Thread(runnable1).start();
        new Thread(runnable2).start();
        new Thread(runnable3).start();
        Thread.sleep(1000);
        testListener.onComplete();
        // 另一方面，create的另一个变体可以设置参数来实现负压控制，具体看源码。
    }
    public interface TestListener<T> {

        void onChunk(List<T> chunk);

        void onComplete();
    }

    public interface TestProcessor<T> {

        void register(TestListener<T> tTestListener);

        TestListener<T> get();
    }

    public interface Runnable1<T> extends Runnable {
         void set(TestListener<T> testListener);
    }
}
```
### 静态异步单线程方法：push()

说完了异步多线程，同步的生成方法，接下来就是异步单线程：push()。

其实说到push和create的对比，我个人理解如下：

- create允许多线程环境下调用.next()方法，只管生成元素，元素序列的顺序取决于...算了，随机的，毕竟多线程；

- 但是push只允许一个线程生产元素，所以是有序的，至于异步指的是在新的线程中也可以，而不必非得在当前线程。

- 顺带一提，push和create都支持onCancel()和onDispose()操作。一般来说，onCancel只响应于cancel操作，而onDispose响应于error，cancel，complete等操作。

``` Java
public class FluxWithPush {

    public static void main(String[] args) throws InterruptedException {
        TestProcessor<String> testProcessor = new TestProcessor<>() {

            private TestListener<String> testListener;

            @Override
            public void register(TestListener<String> testListener) {
                this.testListener = testListener;
            }

            @Override
            public TestListener<String> get() {
                return this.testListener;
            }
        };
        Flux<String> flux = Flux.push(stringFluxSink -> testProcessor.register(new TestListener<>() {
            @Override
            public void onChunk(List<String> list) {
                for (String s : list) {
                    stringFluxSink.next(s);
                }
            }

            @Override
            public void onComplete() {
                stringFluxSink.complete();
            }
        }));
        flux.subscribe(System.out::println);
        Runnable1<String> runnable = new Runnable1<>() {

            private TestListener<String> testListener;

            @Override
            public void set(TestListener<String> testListener) {
                this.testListener = testListener;
            }

            @Override
            public void run() {
                List<String> list = new ArrayList<>(10);
                for (int i = 0; i < 10; ++i) {
                    list.add(UUID.randomUUID().toString());
                }
                testListener.onChunk(list);
            }
        };
        TestListener<String> testListener = testProcessor.get();
        runnable.set(testListener);
        new Thread(runnable).start();
        Thread.sleep(15);
        testListener.onComplete();
    }

    public interface TestListener<T> {
        void onChunk(List<T> list);
        void onComplete();
    }

    public interface TestProcessor<T> {
        void register(TestListener<T> testListener);
        TestListener<T> get();
    }

    public interface Runnable1<T> extends Runnable {
        void set(TestListener<T> testListener);
    }
}
```

同create一样，push也支持负压调节。但是我没写出来，我试过的Demo都是直接请求Long.MAX_VALUE，其实就是通过sink.onRequest(LongConsumer)方法调用来实现负压控制的。原理在这，想深究的请自行探索，鄙人不才，花费一下午没实现。
### 实例方法：handle()
在Flux的实例方法里，handle类似filter和map的操作。

``` Java
public class FluxWithHandle {

    public static void main(String[] args) {
        Flux<String> stringFlux = Flux.push(stringFluxSink -> {
            for (int i = 0; i < 10; ++ i) {
                stringFluxSink.next(UUID.randomUUID().toString().substring(0, 5));
            }
        });
        // 获取所有包含'a'的串
        Flux<String> flux = stringFlux.handle((str, sink) -> {
            String s = f(str);
            if (s != null) {
                sink.next(s);
            }
        });
        flux.subscribe(System.out::println);
    }

    private static String f(String str) {
        return str.contains("a") ? str : null;
    }
}
```
## 线程和调度
### Schedulers的那些静态方法
一般来说，响应式框架都不支持并发，P.s. create那个是生产者并发，它本身不是并发的。所以也没有可用的并发库，需要开发者自己实现。

同时，每一个操作一般都是在上一个操作所在的线程里运行，它们不会拥有自己的线程，而最顶的操作则是和subscribe()在同一个线程。比如Flux.create(...).handle(...).subscribe(...)都在主线程运行的。

在响应式框架里，Scheduler决定了操作在哪个线程被怎么执行，它的作用类似于ExecutorService。不过功能稍微多点。如果你想实现一些并发操作，那么可以考虑使用Schedulers提供的静态方法，来看看有哪些可用的：
#### Schedulers.immediate(): 直接在当前线程提交Runnable任务，并立即执行。

``` Java
package com.learn.reactor.flux;

import reactor.core.scheduler.Schedulers;

/**
 * @author Mr.M
 */
public class FluxWithSchedulers {

    public static void main(String[] args) throws InterruptedException {
        // Schedulers.immediate(): 直接在当前线程提交Runnable任务，并立即执行。
        System.out.println("当前线程：" + Thread.currentThread().getName());
        System.out.println("zxcv");
        Schedulers.immediate().schedule(() -> {
            System.out.println("当前线程是：" + Thread.currentThread().getName());
            System.out.println("qwer");
        });
        System.out.println("asdf");
        // 确保异步任务可以打印出来
        Thread.sleep(1000);
    }
}
```
通过上面看得出，immediate()其实就是在执行位置插入需要执行的Runnable来实现的。和直接把代码写在这里没什么区别。
#### Schedulers.newSingle()：保证每次执行的操作都使用的是一个新的线程。
``` Java
package com.learn.reactor.flux;

import reactor.core.scheduler.Schedulers;

/**
 * @author Mr.M
 */
public class FluxWithSchedulers {

    public static void main(String[] args) throws InterruptedException {
        // 如果你想让每次调用都是一个新的线程的话，可以使用Schedulers.newSingle()，它可以保证每次执行的操作都使用的是一个新的线程。
        Schedulers.single().schedule(() -> {
            System.out.println("当前线程是：" + Thread.currentThread().getName());
            System.out.println("bnmp");
        });
        Schedulers.single().schedule(() -> {
            System.out.println("当前线程是：" + Thread.currentThread().getName());
            System.out.println("ghjk");
        });
        Schedulers.newSingle("线程1").schedule(() -> {
            System.out.println("当前线程是：" + Thread.currentThread().getName());
            System.out.println("1234");
        });
        Schedulers.newSingle("线程1").schedule(() -> {
            System.out.println("当前线程是：" + Thread.currentThread().getName());
            System.out.println("5678");
        });
        Schedulers.newSingle("线程2").schedule(() -> {
            System.out.println("当前线程是：" + Thread.currentThread().getName());
            System.out.println("0100");
        });
        Thread.sleep(1000);
    }
}
```
Schedulers.single()，它的作用是为当前操作开辟一个新的线程，但是记住，所有使用这个方法的操作都共用一个线程；

#### Schedulers.elastic()：一个弹性无界线程池。
无界一般意味着不可管理，因为它可能会导致负压问题和过多的线程被创建。所以马上就要提到它的替代方法。

#### Schedulers.bounededElastic()：有界可复用线程池
``` Java
package com.learn.reactor.flux;

import reactor.core.scheduler.Schedulers;

/**
 * @author Mr.M
 */
public class FluxWithSchedulers {

    public static void main(String[] args) throws InterruptedException {
        Schedulers.boundedElastic().schedule(() -> {
            System.out.println("当前线程是：" + Thread.currentThread().getName());
            System.out.println("1478");
        });
        Schedulers.boundedElastic().schedule(() -> {
            System.out.println("当前线程是：" + Thread.currentThread().getName());
            System.out.println("2589");
        });
        Schedulers.boundedElastic().schedule(() -> {
            System.out.println("当前线程是：" + Thread.currentThread().getName());
            System.out.println("0363");
        });
        Thread.sleep(1000);
    }
}
```
Schedulers.boundedElastic()是一个更好的选择，因为它可以在需要的时候创建工作线程池，并复用空闲的池；同时，某些池如果空闲时间超过一个限定的数值就会被抛弃。

同时，它还有一个容量限制，一般10倍于CPU核心数，这是它后备线程池的最大容量。最多提交10万条任务，然后会被装进任务队列，等到有可用时再调度，如果是延时调度，那么延时开始时间是在有线程可用时才开始计算。

由此可见Schedulers.boundedElastic()对于阻塞的I/O操作是一个不错的选择，因为它可以让每一个操作都有自己的线程。但是记得，太多的线程会让系统备受压力。

#### Schedulers.parallel()：提供了系统级并行的能力
``` Java
package com.learn.reactor.flux;

import reactor.core.scheduler.Schedulers;

/**
 * @author Mr.M
 */
public class FluxWithSchedulers {

    public static void main(String[] args) throws InterruptedException {
        Schedulers.parallel().schedule(() -> {
            System.out.println("当前线程是：" + Thread.currentThread().getName());
            System.out.println("6541");
        });
        Schedulers.parallel().schedule(() -> {
            System.out.println("当前线程是：" + Thread.currentThread().getName());
            System.out.println("9874");
        });
        Thread.sleep(1000);
    }
}
```
最后，Schedulers.parallel()提供了并行的能力，它会创建数量等于CPU核心数的线程来实现这一功能。

#### 其他线程操作
顺带一提，还可以通过ExecutorService创建新的Scheduler。当然，Schedulers的一堆newXXX方法也可以。

有一点很重要，就是boundedElastic()方法可以适用于传统阻塞式代码，但是single()和parallel()都不行，如果你非要这么做那就会抛异常。自定义Schedulers可以通过设置ThreadFactory属性来设置接收的线程是否是被NonBlocking接口修饰的Thread实例。

Flux的某些方法会使用默认的Scheduler，比如Flux.interval()方法就默认使用Schedulers.parallel()方法，当然可以通过设置Scheduler来更改这种默认。

在响应式链中，有两种方式可以切换执行上下文，分别是publishOn()和subscribeOn()方法，前者在流式链中的位置很重要。在Reactor中，可以以任意形式添加任意数量的订阅者来满足你的需求，但是，只有在设置了订阅方法后，才能激活这条订阅链上的全部对象。只有这样，请求才会上溯到发布者，进而产生源序列。

### 在订阅链中切换执行上下文

#### publishOn()
publishOn()就和普通操作一样，添加在操作链的中间，它会影响在它下面的所有操作的执行上下文。看个例子：
``` Java
public class FluxWithPublishOnSubscribeOn {

    public static void main(String[] args) throws InterruptedException {
    	// 创建一个并行线程
        Scheduler s = Schedulers.newParallel("parallel-scheduler", 4);
        final Flux<String> flux = Flux
                .range(1, 2)
                // map肯定是跑在T上的。
                .map(i -> 10 + i)
                // 此时的执行上下文被切换到了并行线程
                .publishOn(s)
                // 这个map还是跑在并行线程上的，因为publishOn()的后面的操作都被切换到了另一个执行上下文中。
                .map(i -> "value " + i);
        // 假设这个new出来的线程名为T
        new Thread(() -> flux.subscribe(System.out::println));
        Thread.sleep(1000);
    }
}
```

#### subscribeOn()
``` Java
public class FluxWithPublishOnSubscribeOn {

    public static void main(String[] args) throws InterruptedException {
        // 依旧是创建一个并行线程
        Scheduler ss = Schedulers.newParallel("parallel-scheduler", 4);
        final Flux<String> fluxflux = Flux
                .range(1, 2)
                // 不过这里的map就已经在ss里跑了
                .map(i -> 10 + i)
                // 这里切换，但是切换的是整个链
                .subscribeOn(s)
                // 这里的map也运行在ss上
                .map(i -> "value " + i);
        // 这是一个匿名线程TT
        new Thread(() -> fluxflux.subscribe(System.out::println));
        Thread.sleep(1000);
    }
}
```
subscribeOn()方法会把订阅之后的整个订阅链都切换到新的执行上下文中。无论在subscribeOn()哪里，都可以把最前面的订阅之后的订阅序列进行切换，当然了，如果后面还有publishOn()，publishOn()会进行新的切换。