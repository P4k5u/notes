### 简介
CompleteFuture 结合了 Future 的优点，可以通过回调的方式处理计算结果，并且提供了转换和组合的方式
CompleteFuture 被设计在 Java 中进行异步编程，异步编程意味着在主线程之外创建一个独立的线程，与主线程分隔开，并在上面运行一个非阻塞的任务，然后通知主线程进展，成功或失败
通过这种方式，你的主线程不用为了任务的完成而阻塞/等待，可以用主线程去并行执行其他的任务

### 使用方法

#### 实例化 CompleteFuture
```java
public static <U> CompletableFuture<U> supplyAsync(Supplier<U> supplier);
public static <U> CompletableFuture<U> supplyAsync(Supplier<U> supplier, Executor executor);

public static CompletableFuture<Void> runAsync(Runnable runnable);
public static CompletableFuture<Void> runAsync(Runnable runnable, Executor executor);
```
有两种格式：
- supplyAsync：可以返回异步线程执行之后的结果
- runAsync：不会返回结果，只是执行线程任务
或者通过一个简单的无参构造器
```java
CompletableFuture<String> completableFuture = new CompletableFuture<String>();
```
在实例化方法中，可以传入 Executor 参数，当不传入 Executor 参数时，会默认使用系统级公共线程池 ForkJoinPool.commonPool() ，而这些线程都是守护线程
当主线程结束，JVM 中不存在其余用户线程，那么 CompleteFuture 的守护线程会直接退出，造成任务无法完成

#### 获取结果

##### 同步获取结果
```java
public T get()
public T get(long timeout, TimeUnit unit)
public T getNow(T valueIfAbsent)
public T join()
```
简单使用例子：
```java
CompletableFuture<Integer> future = new CompletableFuture<>();
Integer integer = future.get();
```
get() 方法同样会阻塞到任务完成，主线程会一直阻塞，因为这种方法创建的 future 从未完成
getNow() 方法中的参数 valueIfAbsent 的意思是当计算结果不存在或 Now 时刻没有完成任务，给定一个特点的结果
join() 和 get() 的区别是，join() 返回计算的结果或抛出一个 unchecked 异常，而 get() 返回一个具体的异常

##### 计算完成后续操作

###### complete
```java
public CompletableFuture<T> whenComplete(BiConsumer<? super T,? super Throwable> action)
public CompletableFuture<T> whenCompleteAsync(BiConsumer<? super T,? super Throwable> action)
public CompletableFuture<T> whenCompleteAsync(BiConsumer<? super T,? super Throwable> action, Executor executor)
```
方法名带有 Async 的就是异步处理，同时也可以通过传入 Executor 参数来使用自定义线程池
###### handle
handle 和 complete 的区别就在于返回值，虽然同样返回 CompleteFuture 类型，但是里面的参数类型，handle 方法是可以自定义的
```java
        //开启一个异步方法
        CompletableFuture<List> future = CompletableFuture.supplyAsync(() -> {
            List<String> list = new ArrayList<>();
            list.add("语文");
            list.add("数学");
            // 获取得到今天的所有课程
            return list;
        });
        // 使用handle()方法接收list数据和error异常
        CompletableFuture<Integer> future2 = future.handle((list,error)-> {
            // 如果报错，就打印出异常
            error.printStackTrace();
            // 如果不报错，返回一个包含Integer的全新的CompletableFuture
            return list.size();
             // 注意这里的两个CompletableFuture包含的返回类型不同
        });
```
###### apply
```java
public <U> CompletableFuture<U> thenApply(Function<? super T,? extends U> fn)
public <U> CompletableFuture<U> thenApplyAsync(Function<? super T,? extends U> fn)
public <U> CompletableFuture<U> thenApplyAsync(Function<? super T,? extends U> fn, Executor executor)
```
apply 方法出现异常之后就会直接抛出，交给上一层处理
###### accept
```java
public CompletableFuture<Void> thenAccept(Consumer<? super T> action)
public CompletableFuture<Void> thenAcceptAsync(Consumer<? super T> action)
public CompletableFuture<Void> thenAcceptAsync(Consumer<? super T> action, Executor executor)
```
accept 相关的方法只做最终结果的消费，注意此时返回的 CompleteFuture 是空返回，只消费，无返回
###### exceptionally
```java
public CompletableFuture<T> exceptionally(Function<Throwable, ? extends T> fn)
```
exceptionally() 方法可以捕捉到所有中间过程的异常，方法会给我们一个异常作为参数，我们可以处理这个异常，同时返回一个默认值，跟服务降级类似
使用例子：
```java
//实例化一个CompletableFuture,返回值是Integer
        CompletableFuture<Integer> future = CompletableFuture.supplyAsync(() -> {
            // 返回null
            return null;
        });

        CompletableFuture<String> exceptionally = future.thenApply(result -> {
            // 制造一个空指针异常NPE
            int i = result;
            return i;
        }).thenApply(result -> {
            // 这里不会执行，因为上面出现了异常
            String words = "现在是" + result + "点钟";
            return words;
        }).exceptionally(error -> {
            // 我们选择在这里打印出异常
            error.printStackTrace();
            // 并且当异常发生的时候，我们返回一个默认的文字
            return "出错啊~";
        });

        exceptionally.thenAccept(System.out::println);

    }
```

#### 组合式异步编程

##### 组合 CompleteFuture

###### thenApply()
现在有个场景：
- 根据名字获取学生信息
- 根据学生信息获取课程信息
那么就可以进行以下的链式操，代码如下：
```java
CompletableFuture<List<Lesson>> future = CompletableFuture.supplyAsync(() -> {
            // 根据学生姓名获取学生信息
            return StudentService.getStudent(name);
        }).thenApply(student -> {
            // 再根据学生信息获取今天的课程
            return LessonsService.getLessons(student);
        });
```
将两个异步操作计算合并为一个，这两个异步计算之间相互独立，互不依赖
###### thenCompose()
需要查询美术课和劳技课的工具
使用 thenCompose() 的代码如下：
```java
CompletableFuture<List<String>> total = CompletableFuture.supplyAsync(() -> {
            // 第一个任务获取美术课需要带的东西，返回一个list
            List<String> stuff = new ArrayList<>();
            stuff.add("画笔");
            stuff.add("颜料");
            return stuff;
        }).thenCompose(list -> {
            // 向第二个任务传递参数list(上一个任务美术课所需的东西list)
            CompletableFuture<List<String>> insideFuture = CompletableFuture.supplyAsync(() -> {
                List<String> stuff = new ArrayList<>();
                // 第二个任务获取劳技课所需的工具
                stuff.add("剪刀");
                stuff.add("折纸");
                // 合并两个list，获取课程所需所有工具
                List<String> allStuff = Stream.of(list, stuff).flatMap(Collection::stream).collect(Collectors.toList());
                return allStuff;
            });
            return insideFuture;
        });
        System.out.println(total.join().size());
```
我们通过 **CompletableFuture.supplyAsync(）** 方法创建第一个任务，获得美术课所需的物品list，然后使用**thenCompose()** 接口传递 list 到第二个任务，然后第二个任务获取劳技课所需的物品，整合之后再返回。至此我们完成两个任务的合并。 （说实话，用compose去实现这个业务场景看起来有点别扭，我们看下一个例子）

###### thenCombine()
还是上面的场景，需要查询美术课和劳技课的工具
使用 thenCombine() 的代码如下：
```java
CompletableFuture<List<String>> painting = CompletableFuture.supplyAsync(() -> {
            // 第一个任务获取美术课需要带的东西，返回一个list
            List<String> stuff = new ArrayList<>();
            stuff.add("画笔");
            stuff.add("颜料");
            return stuff;
        });
        
CompletableFuture<List<String>> handWork = CompletableFuture.supplyAsync(() -> {
            // 第二个任务获取劳技课需要带的东西，返回一个list
            List<String> stuff = new ArrayList<>();
            stuff.add("剪刀");
            stuff.add("折纸");
            return stuff;
        });
        
CompletableFuture<List<String>> total = painting
                // 传入handWork列表，然后得到两个CompletableFuture的参数Stuff1和2
                .thenCombine(handWork, (stuff1, stuff2) -> {
                    // 合并成新的list
                    List<String> totalStuff = Stream.of(stuff1, stuff1)
                            .flatMap(Collection::stream)
                            .collect(Collectors.toList());
                    return totalStuff;
                });
        System.out.println(JSONObject.toJSONString(total.join()));
```
然后等待Future集合中的所有任务都完成。
##### 获取所有完成结果 allOf
```java
public static CompletableFuture<Void> allOf(CompletableFuture<?>... cfs)
```
allOf() 方法，当所有给定的任务完成后，返回一个全新的已完成 CompletableFuture
```java
CompletableFuture<Integer> future1 = CompletableFuture.supplyAsync(() -> {
            try {
                //使用sleep()模拟耗时操作
                TimeUnit.SECONDS.sleep(2);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            return 1;
        });

        CompletableFuture<Integer> future2 = CompletableFuture.supplyAsync(() -> {
            return 2;
        });
        CompletableFuture.allOf(future1, future1);
        // 输出3
        System.out.println(future1.join()+future2.join());
```

##### 获取率先完成的任务结果 anyOf
```java
public static CompletableFuture<Object> anyOf(CompletableFuture<?>... cfs)
```
仅等待 Future 集合中最快结束的任务完成（有可能因为他们试图通过不同的方式计算同一个值），并返回它的结果。 **小贴士** ：如果最快完成的任务出现了异常，也会先返回异常，如果害怕出错可以加个**exceptionally()** 去处理一下可能发生的异常并设定默认返回值
```java
CompletableFuture<Integer> future = CompletableFuture.supplyAsync(() -> {
            throw new NullPointerException();
        });

        CompletableFuture<Integer> future2 = CompletableFuture.supplyAsync(() -> {
            try {
                // 睡眠3s模拟延时
                TimeUnit.SECONDS.sleep(3);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            return 1;
        });
        CompletableFuture<Object> anyOf = CompletableFuture
                .anyOf(future, future2)
                .exceptionally(error -> {
                    error.printStackTrace();
                    return 2;
                });
        System.out.println(anyOf.join());
```

### 使用例子

#### 多个方法组合使用
- 通过编程方式完成一个Future任务的执行（即以手工设定异步操作结果的方式）。
- 应对Future的完成时间（即当Future的完成时间完成时会收到通知，并能使用Future的计算结果进行下一步的的操作，不只是简单地阻塞等待操作的结果）
```java
public static void main(String[] args) {
        CompletableFuture.supplyAsync(() -> 1)
                .whenComplete((result, error) -> {
                    System.out.println(result);
                    error.printStackTrace();
                })
                .handle((result, error) -> {
                    error.printStackTrace();
                    return error;
                })
                .thenApply(Object::toString)
                .thenApply(Integer::valueOf)
                .thenAccept((param) -> System.out.println("done"));
    }
```

#### 循环创建并发任务
```java
public static void main(String[] args) {
        long begin = System.currentTimeMillis();
        // 自定义一个线程池
        ExecutorService executorService = Executors.newFixedThreadPool(10);
        // 循环创建10个CompletableFuture
        List<CompletableFuture<Integer>> collect = IntStream.range(1, 10).mapToObj(i -> {
            CompletableFuture<Integer> future = CompletableFuture.supplyAsync(() -> {
                // 在i=5的时候抛出一个NPE
                if (i == 5) {
                    throw new NullPointerException();
                }
                try {
                    // 每个依次睡眠1-9s，模拟线程耗时
                    TimeUnit.SECONDS.sleep(i);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println(i);
                return i;
            }, executorService)
                    // 这里处理一下i=5时出现的NPE
                    // 如果这里不处理异常，那么异常会在所有任务完成后抛出,小伙伴可自行测试
                    .exceptionally(Error -> {
                        try {
                            TimeUnit.SECONDS.sleep(5);
                            System.out.println(100);
                        } catch (InterruptedException e) {
                            e.printStackTrace();
                        }
                        return 100;
                    });
            return future;
        }).collect(Collectors.toList());
        // List列表转成CompletableFuture的Array数组,使其可以作为allOf()的参数
        // 使用join()方法使得主线程阻塞，并等待所有并行线程完成
        CompletableFuture.allOf(collect.toArray(new CompletableFuture[]{})).join();
        System.out.println("最终耗时" + (System.currentTimeMillis() - begin) + "毫秒");
        executorService.shutdown();
    }
```