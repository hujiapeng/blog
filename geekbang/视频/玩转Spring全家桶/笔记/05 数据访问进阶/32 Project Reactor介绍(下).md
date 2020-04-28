1. 一个简单的生产、消费模型如下代码：
```
   Flux.range(1, 6)
           .subscribe(i -> log.info("Subscribe {}: {}", Thread.currentThread(), i)
                   , e -> log.error("error {}", e.toString())
                   , () -> log.info("Subscriber COMPLETE")
                   //,s -> s.request(4)
   );
```
 其中subscribe方法第一个参数是对该条数据的消费，第二个参数是消费过程中如果出现异常，参数就是该异常对象，第三个参数是对消费完所有数据后的处理，如果有第四个参数，这个是限制消费数量的。
2. 添加了各种操作的示例代码，如下
```
    @Override
    public void run(ApplicationArguments args) throws Exception {
        //注意，流式处理虽然有多线程，但是也有顺序，如注册事件或动作的先后顺序有可能会影响结果
        Flux.range(1, 6)
                //将数据在一个线程池的缓存线程上进行发布，Schedulers.elastic()这个缓存的线程时效默认为60s
                .publishOn(Schedulers.elastic())
                .doOnRequest(n -> log.info("Request {} number", n))
                //当生产结束时执行
                .doOnComplete(() -> log.info("Publisher COMPLETE"))
                //对生产的数据进行处理
                .map(i -> {
                    log.info("Publish {}, {}", Thread.currentThread(), i);
//                    return 10 / (i - 3);
                    return i;
                })
                //如果生产数据中有异常，进行如下处理
                .onErrorResume(e -> {
                    log.error("Exception {}", e.toString());
                    return Mono.just(-1);
                })
                //对生产的异常数据直接返回
                .onErrorReturn(-1)
                //使用线程池的一个线程进行消费
                .subscribeOn(Schedulers.single())
                //消费生产出的数据
                .subscribe(i -> log.info("Subscribe {}: {}", Thread.currentThread(), i)
                        , e -> log.error("error {}", e.toString())
                        , () -> log.info("Subscriber COMPLETE")
//                        , s -> s.request(3)
                );
        //防止子线程还没有结束，主线程提前退出
        Thread.sleep(2000);
    }
```
3. 通过查看源码得知，在subscribe之前，所有的动作都是在用onAssembly进行包装，等到调用subscribe的时候才执行绑定的一些操作