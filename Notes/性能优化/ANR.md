---
ANR
---

#### 目录
1. ANR监控
2. 分析步骤
3. 案例分析
#### ANR监控
线上监控ANR的方式
它是通过在一个工作线程中，用主线程的 Handler post 一个消息，然后这个线程直接 sleep 5s，sleep 之后看看标志位是否被修改，如果没被修改就说明主线程卡顿了 5s，即发生了 ANR。
#### 分析步骤
##### 确认是否是系统环境影响
1. 受整机负载影响：<br>
搜索`“ANR in”`关键词
![image](https://github.com/Coopergp/Android/assets/163702335/31ddfdfd-d74a-4826-8cdc-4d736507d056)<br>
(1) ANR in com.journeyui.calculator  ANR进程名<br>
(2) Reason: Input dispatching timed out (ActivityRecord{51e27ca u0 com.journeyui.calculator/.Calculator t1837} does not have a focused window)<br>
(3) Reason后面跟的是本次ANR原因，上面这个例子中通俗的解释是：
事件落到com.tencent.mm/.ui.LauncherUI窗口上，不过该窗口直到超时时间到了仍未响应输入事件，input超时时间系统默认是5s。<br>
(4) Load: 31.7 / 33.43 / 30.98
前 1，前 5，前 15 分钟的负载，我们通常看变化趋势。<br>
(5) 2022-01-29 06:21:47.589 to 2022-01-29 06:24:34.860  统计的时间区域<br>
(6) 85% TOTAL: 42% user + 33% kernel + 0% iowait + 0% softirq
代表了整体的负载情况，85% TOTAL说明整机负载很高。<br>
(7) 图中(4)-(5)之间的片段则是各个进程占据的CPU情况, 发生ANR的进程com.journeyui.calculator的CPU占用只有1.7%

2. 受低内存影响：<br>
内存紧张的时候，kswapd线程会活跃起来进行回收内存
![image](https://github.com/Coopergp/Android/assets/163702335/b01aaeb0-23bc-4a83-aaec-bee98b1f08d6)
##### 分析堆栈

- 看event日志<br>
日志中搜索`“am_anr”`关键词，可以确认时间点和进程号
  ![image](https://github.com/Coopergp/Android/assets/163702335/506c1245-7f67-4acc-b350-636bb0ecbed9)
- 看trace文件<br>
看线程运行状态，分析堆栈信息
 ![image](https://github.com/Coopergp/Android/assets/163702335/803e648e-a023-4a7a-a829-a20934b39261)<br>
(1) 线程运行状态<br>
nice值代表该线程优先级，nice的取值范围为-20到19。<br>
通常来说nice的值越大，进程的优先级就越低，获得CPU调用的机会越少，nice值越小，进程的优先级则越高，获得CPU调用的机会越多。<br>
(2) utm：该线程在用户态所执行的时间(单位是jiffies）<br>
stm：该线程在内核态所执行的时间<br>
线程的cpu耗时是两者相加(utm+stm)，utm,stm 单位换算成时间单位为 1 比 10ms。<br>
(3) core：后面的数字表示的跑在哪个核上，core=7表示打印这段日志时该线程跑在大核CPU7上。<br>
(4) 函数调用堆栈，也是我们最为关心的部分。
- 看mainlog日志<br>
  分析发生ANR时的CPU状态(分析系统环境影响)
#### 案例分析
- ##### 主线程耗时
  最为常见的一种类型，也是最容易分析的类型
  ![image](https://github.com/Coopergp/Android/assets/163702335/c2dba1b4-dfb7-4b15-8d57-7b3fc2a8ea15)<br>
  通常来说，如果打印的堆栈是该进程内部的堆栈，并且经过业务证实这段代码确实可能存在耗时，那么根据callstack位置找到代码修改即可。
- ##### 主线程Blocked/Waiting/Sleeping
  ![image](https://github.com/Coopergp/Android/assets/163702335/f7a989ce-b539-40f5-9894-2e2b5a7948c9)<br>
  从上面图中我们可以看到，主线程当前的状态是(1)Blocked，原因是它在等锁(2) 0x06573567，而0x06573567被(3)线程13所持有。<br>
  此时我们自然想到去看下tid=13线程的CallStack，在该份trace文件中搜索关键字"0x06573567"找到线程13的CallStack。
  ![image](https://github.com/Coopergp/Android/assets/163702335/cc975b60-fafe-4d2a-89b8-e4df28166625)<br>
  主线程陷入Blocked状态的缘由，通常是由于持锁线程在做繁琐的事务，或者是主线程和其它线程发生了死锁。<br>
  当然除了主线程陷入Blocked状态这种常见的情况之外，还有一些比较少见的情况。
  
  1) 主线程处于waiting，说明其正在等待其他线程来notify它，堆栈同样是等锁，分析思路一样。<br>
  2) 主线程处于Sleeping状态，说明当前线程主动调用sleep，其堆栈通常是sleeping on <锁ID>。
- ##### Binder阻塞等待

  Binder阻塞等待指的是什么呢？

  比如进程A在其主线程中向进程B发起了Binder请求，进程B因为一些原因比如正在处理耗时消息或网络异常等原因，无法及时响应进程A的Binder请求，造成进程A在主线程上一直阻塞等待Binder的返回结果，最终触发ANR。

  ![image](https://github.com/Coopergp/Android/assets/163702335/8e8fc6ea-b3be-4dd6-b040-20ec6332711f)<br>
  从图中可以看到，Keyguard在做native层的Binder通信，并处于阻塞等待对端结果返回的状态中。

- ##### 无效堆栈

  在我们的实际项目上，经常会遇到这样的情况，日志提供的是完整的，可堆栈看起来却不像"作案"堆栈，即出现的堆栈并不像是真正的凶手。
常见的一种情况:  堆栈落在nativePollOnce上。

  说明应用此时已经处于idle状态

  ![image](https://github.com/Coopergp/Android/assets/163702335/6af3ebbf-fedc-482a-8fe6-559dffd3ada4)<br>
  对于这种情况，说明耗时消息已埋没在历史消息中，历史消息的耗时可能存在下面的几种情况<br>
  (1) 对于这种情况，说明耗时消息已埋没在历史消息中，历史消息的耗时可能存在下面的几种情况<br>
  (2) 当线程等锁时间超出设定阈值，则输出当前的持锁状态。<br>
  (3) 主线程的生命周期回调方法执行时间超出设定阈值，则输出相应信息。<br>
  (4) 对于非异步Binder调用耗时超出设定阈值的时候，输出Binder信息。

- ##### I/O问题

  SharedPreference apply引起的ANR问题

  SP 调用 apply 方法，会创建一个等待锁放到 QueuedWork 中，并将真正数据持久化封装成一个任务放到异步队列中执行，任务执行结束后释放锁。Activity onPause 时执行 QueuedWork.waitToFinish() 等待所有的等待锁释放，等待超时则发生 ANR。其实不管是 commit 还是 apply，都是在主线程进行 IO 操作，那么都是可能会产生 ANR 的。

  那么这个问题如何去解决呢？字节的做法是清空等待锁队列，而 Booster 最初的做法是将 apply() 替换成 commit() 并在子线程中执行，这个方案的的优点是改动很小，风险相对较小，缺点是在调用 commit() 后立即调用 getXxx() 可能会导致 bug，毕竟异步调用 commit() 确实会有一定的概率出现数据不同步。

- ##### 应用内存问题

  我们实际项目中，不时会遇到应用自身内存使用不当导致的ANR。

  这个案例中，我们在案发时间点附近，发现大量的GC片段且很多GC耗时都较长。
  ![image](https://github.com/Coopergp/Android/assets/163702335/afe13b38-0f96-4bc4-81bf-06c6e8f75293)<br>
  Clamp target GC heap from这行日志是在SetIdealFootprint即调整目标堆上限值时会打印

  这种情况下很可能是出现了应用内存泄漏的情况

  关于应用内存使用不当，通常有如下几种情况：<br>
  (1) 频繁的生成临时对象导致堆内存增长迅速，达到下次堆GC触发阈值后便会触发Bg GC，进而导致回收线程跑大核和前台应用争抢CPU。<br>
      另外GC回收阶段会存在一次锁堆，应用的主线程会被pause，这种情况下势必会造成应用使用卡顿甚至ANR。
  (2) 还有一种比较常见的情况是应用发生了较为严重的内存泄漏，导致GC一直无法回收足够的内存。
  (3) 应用申请大内存触发阻塞GC以便能够申请到足够的内存，这种情况通常会引起应用界面的黑屏或者明显的卡顿。
  (4) 我们知道系统低内存时会触发OnTrimMemory回调，如果应用在OnTrimMemory中并且是在主线程中直接调用显式GC接口即System.gc()，也容易引起应用卡顿，对于这个接口的使用需要谨慎。
  

#### 参考

[酷派技术团队：Android ANR|原理解析及常见案例](https://mp.weixin.qq.com/s?__biz=MzkwNjI5MjAxNg==&mid=2247483844&idx=1&sn=a6e043f3e4a494dfabfbc76952b6d936&scene=21#wechat_redirect)
