### 源码阅读感悟

#### Segment设计思想
1. **为啥不用自增id方案** ：利用mysql主键原子自增：随业务增多，DB压力过大，且会有单点故障。mysql先插id再取，有两次io次数。性能差，不可取。那能不能自己定义一个id呢？   
2. **分段id+缓存池：** 不用mysql自增id，自己定义一个id ，比如定义一个step字段（0-1000，1000-2000...）。应用服务器一次从mysql拿1000个id过来，应用服务器定义一个id缓存池来存放这边id，id池用完了再从数据库中取，避免了频繁访问数据库。当此方案中的id池刚好用完了，此时如果有很多线程打在了应用服务器上，会导致挤压很多请求，极端情况会导致服务不可用，这种情况怎么办？
3. **双缓冲池：** 应用服务器搞两个id池，当id池1用到了10%的时候（`可以根据消费速度做动态自适应调整池的大小和池的阈值，Leaf没有做这一步`），则去初始化id池2。当id池1用完了，切到id池2，再去初始化id池1，这样可以防止线程阻塞。
4. **高并发优化：** **异步+线程池**：起一个单线程对id池进行异步填充。  **无锁化**：ThreadLocal对业务线程做优化，每个业务线程的ThreadLocal从id池缓存出100个id自己消费，去id池拿的时候会涉及到资源竞争使用CAS去拿。**资源锁细粒度化**：mysql id表中定义一个业务标识字段，每一个争用位给一个业务标识字段，根据业务标识字段去分拆，这样上锁只会锁这个业务字段的。  

 Segment设计的缺点：Id是递增且是可计算的（`64位：时间戳 + node id + area id + 自增`），不适合订单的id生成场景，容易被竟对公司推测出一天的订单量，比如竞对在两天中午 12 点分别下单，就可以猜出一天的订单量。

#### 开源软件的一个重要原则：去依赖
* 依赖太多会导致易用性不高，比如依赖了spring，那么没有spring就跑不起来，局限性很高

#### 见贤思齐焉，见不贤而内自省也
* Leaf的设计思路值得学习，但代码中也有很多不规范的地方。解决方案还有继续优化的余地。
todo
#### 总结
* 再梳理一遍脉络
* 知识串联， 双buffer的设计思想类似于 redis rehash,快到达阈值刷新buffer类似mysql脏页刷新策略、redis触发aof重写时机。 
* Leaf 高可用机制 todo
* https://tech.meituan.com/2017/04/21/mt-leaf.html
* [微信消息序列号实现方案](https://mp.weixin.qq.com/s/JqIJupVKUNuQYIDDxRtfqA)

[中文文档](./README_CN.md) 
