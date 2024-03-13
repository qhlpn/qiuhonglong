------

此文档记录学习 **服务接口访问控制** 在 alogic 中的实现，包括 **登录控制**，**流量限制**

------

> 开局一张图 ......

<img src="./pictures/051.png" style="zoom: 80%;" />



> 访问控制 **AccessController** 主要有3个切面方法

1. createSessionId() 根据是否登录来控制服务入口
2. accessStart() 采用限流算法来控制流量、获取访问优先级
3. accessEnd() 可用于资源池回收



> debug

``` 
java.lang.RuntimeException: configure debug
        at cn.cdnplus.nest.scout.xscript.DocCache2.configure(DocCache2.java:43)
        at com.alogic.xscript.Block.configure(Block.java:91)
        at com.alogic.xscript.Block.configure(Block.java:81)
        at com.alogic.xscript.Block.configure(Block.java:81)
        at com.alogic.xscript.Script.create(Script.java:88)
        at com.alogic.xscript.Script.create(Script.java:68)
        at com.alogic.together3.TogetherServiceDescription.configure(TogetherServiceDescription.java:160)
        at com.alogic.together3.catalog.FromClasspath.loadServiceDescription(FromClasspath.java:151)
        at com.alogic.together3.catalog.FromClasspath.serviceFound(FromClasspath.java:135)
        at com.alogic.together3.catalog.FromClasspath.scanJar(FromClasspath.java:112)
        at com.alogic.together3.catalog.FromClasspath.scanResource(FromClasspath.java:94)
        at com.alogic.together3.catalog.FromClasspath.createCatalogNode(FromClasspath.java:60)
        at com.logicbus.models.servant.impl.XMLDocumentServantCatalog.getChildren(XMLDocumentServantCatalog.java:209)
        at com.logicbus.models.servant.ServantCatalog$Abstract.scan(ServantCatalog.java:141)
        at com.logicbus.models.servant.ServantCatalog$Abstract.scan(ServantCatalog.java:144)
        at com.logicbus.models.servant.ServantCatalog$Abstract.scan(ServantCatalog.java:131)
        at com.logicbus.backend.ServantRegistry$Abstract.scan(ServantRegistry.java:244)
        at com.logicbus.backend.ServantRegistry$Abstract.configure(ServantRegistry.java:142)
        at com.anysoft.util.Factory.newInstance(Factory.java:104)
        at com.logicbus.backend.ServantFactory$Abstract.configure(ServantFactory.java:113)
        at com.anysoft.util.Factory.newInstance(Factory.java:83)
        at com.anysoft.util.Factory.newInstance(Factory.java:60)
        at com.logicbus.backend.ServantFactory$TheFactory.getServantFactory(ServantFactory.java:228)
        at com.logicbus.backend.ServantFactory$TheFactory.get(ServantFactory.java:216)
        at com.logicbus.backend.server.LogicBusApp.onInit(LogicBusApp.java:187)
        at com.logicbus.backend.server.LogicBusApp.init(LogicBusApp.java:263)
        at com.anysoft.webloader.WebAppContextListener.contextInitialized(WebAppContextListener.java:134)
        at org.eclipse.jetty.server.handler.ContextHandler.callContextInitialized(ContextHandler.java:1068)
        at org.eclipse.jetty.servlet.ServletContextHandler.callContextInitialized(ServletContextHandler.java:572)
        at org.eclipse.jetty.server.handler.ContextHandler.contextInitialized(ContextHandler.java:997)
        at org.eclipse.jetty.servlet.ServletHandler.initialize(ServletHandler.java:746)
        at org.eclipse.jetty.servlet.ServletContextHandler.startContext(ServletContextHandler.java:379)
        at org.eclipse.jetty.webapp.WebAppContext.startWebapp(WebAppContext.java:1449)
        at org.eclipse.jetty.webapp.WebAppContext.startContext(WebAppContext.java:1414)
        at org.eclipse.jetty.server.handler.ContextHandler.doStart(ContextHandler.java:911)
        at org.eclipse.jetty.servlet.ServletContextHandler.doStart(ServletContextHandler.java:288)
        at org.eclipse.jetty.webapp.WebAppContext.doStart(WebAppContext.java:524)
        at org.eclipse.jetty.util.component.AbstractLifeCycle.start(AbstractLifeCycle.java:73)
        at org.eclipse.jetty.util.component.ContainerLifeCycle.start(ContainerLifeCycle.java:169)
        at org.eclipse.jetty.util.component.ContainerLifeCycle.doStart(ContainerLifeCycle.java:117)
        at org.eclipse.jetty.server.handler.AbstractHandler.doStart(AbstractHandler.java:97)
        at org.eclipse.jetty.util.component.AbstractLifeCycle.start(AbstractLifeCycle.java:73)
        at org.eclipse.jetty.util.component.ContainerLifeCycle.start(ContainerLifeCycle.java:169)
        at org.eclipse.jetty.server.Server.start(Server.java:423)
        at org.eclipse.jetty.util.component.ContainerLifeCycle.doStart(ContainerLifeCycle.java:110)
        at org.eclipse.jetty.server.handler.AbstractHandler.doStart(AbstractHandler.java:97)
        at org.eclipse.jetty.server.Server.doStart(Server.java:387)
        at org.eclipse.jetty.util.component.AbstractLifeCycle.start(AbstractLifeCycle.java:73)
        at com.ketty.main.ConfigurableServer.start(ConfigurableServer.java:245)
        at com.ketty.main.ConfigurableServerWithLogback.main(ConfigurableServerWithLogback.java:74)
        
        
        
                 
java.lang.RuntimeException: onExecute debug
        at cn.cdnplus.nest.scout.xscript.DocCache2.onExecute(DocCache2.java:54)
        at com.alogic.xscript.Block.execute(Block.java:104)
        at com.alogic.xscript.plugins.Segment.onExecute(Segment.java:32)
        at com.logicbus.dbcp.xscript.DBConn.onExecute(DBConn.java:83)
        at com.alogic.xscript.Block.execute(Block.java:104)
        at com.alogic.xscript.plugins.Segment.onExecute(Segment.java:32)
        at com.alogic.xscript.Block.execute(Block.java:104)
        at com.alogic.together3.service.TogetherServant.onJson(TogetherServant.java:56)
        at com.logicbus.backend.AbstractServant.actionProcess(AbstractServant.java:54)
        at com.logicbus.backend.server.MessageRouter.doExecute(MessageRouter.java:236)
        at com.logicbus.backend.server.MessageRouter.action(MessageRouter.java:174)
        at com.logicbus.backend.server.MessageRouter.action(MessageRouter.java:119)
        at com.logicbus.backend.server.http.MessageRouterServletHandler.doService(MessageRouterServletHandler.java:265)
        at com.anysoft.webloader.ServletAgent.doGet(ServletAgent.java:72)
        at javax.servlet.http.HttpServlet.service(HttpServlet.java:687)
        at javax.servlet.http.HttpServlet.service(HttpServlet.java:790)
        at org.eclipse.jetty.servlet.ServletHolder$NotAsync.service(ServletHolder.java:1443)
        at org.eclipse.jetty.servlet.ServletHolder.handle(ServletHolder.java:791)
        at org.eclipse.jetty.servlet.ServletHandler.doHandle(ServletHandler.java:550)
        at org.eclipse.jetty.server.handler.ScopedHandler.handle(ScopedHandler.java:143)
        at org.eclipse.jetty.security.SecurityHandler.handle(SecurityHandler.java:602)
        at org.eclipse.jetty.server.handler.HandlerWrapper.handle(HandlerWrapper.java:127)
        at org.eclipse.jetty.server.handler.ScopedHandler.nextHandle(ScopedHandler.java:235)
        at org.eclipse.jetty.server.session.SessionHandler.doHandle(SessionHandler.java:1624)
        at org.eclipse.jetty.server.handler.ScopedHandler.nextHandle(ScopedHandler.java:233)
        at org.eclipse.jetty.server.handler.ContextHandler.doHandle(ContextHandler.java:1435)
        at org.eclipse.jetty.server.handler.ScopedHandler.nextScope(ScopedHandler.java:188)
        at org.eclipse.jetty.servlet.ServletHandler.doScope(ServletHandler.java:501)
        at org.eclipse.jetty.server.session.SessionHandler.doScope(SessionHandler.java:1594)
        at org.eclipse.jetty.server.handler.ScopedHandler.nextScope(ScopedHandler.java:186)
        at org.eclipse.jetty.server.handler.ContextHandler.doScope(ContextHandler.java:1350)
        at org.eclipse.jetty.server.handler.ScopedHandler.handle(ScopedHandler.java:141)
        at org.eclipse.jetty.server.handler.HandlerCollection.handle(HandlerCollection.java:146)
        at org.eclipse.jetty.server.handler.HandlerWrapper.handle(HandlerWrapper.java:127)
        at org.eclipse.jetty.server.Server.handle(Server.java:516)
        at org.eclipse.jetty.server.HttpChannel.lambda$handle$1(HttpChannel.java:388)
        at org.eclipse.jetty.server.HttpChannel.dispatch(HttpChannel.java:633)
        at org.eclipse.jetty.server.HttpChannel.handle(HttpChannel.java:380)
        at org.eclipse.jetty.server.HttpConnection.onFillable(HttpConnection.java:279)
        at org.eclipse.jetty.io.AbstractConnection$ReadCallback.succeeded(AbstractConnection.java:311)
        at org.eclipse.jetty.io.FillInterest.fillable(FillInterest.java:105)
        at org.eclipse.jetty.io.ChannelEndPoint$1.run(ChannelEndPoint.java:104)
        at org.eclipse.jetty.util.thread.strategy.EatWhatYouKill.runTask(EatWhatYouKill.java:336)
        at org.eclipse.jetty.util.thread.strategy.EatWhatYouKill.doProduce(EatWhatYouKill.java:313)
        at org.eclipse.jetty.util.thread.strategy.EatWhatYouKill.tryProduce(EatWhatYouKill.java:171)
        at org.eclipse.jetty.util.thread.strategy.EatWhatYouKill.run(EatWhatYouKill.java:129)
        at org.eclipse.jetty.util.thread.ReservedThreadExecutor$ReservedThread.run(ReservedThreadExecutor.java:383)
        at org.eclipse.jetty.util.thread.QueuedThreadPool.runJob(QueuedThreadPool.java:882)
        at org.eclipse.jetty.util.thread.QueuedThreadPool$Runner.run(QueuedThreadPool.java:1036)
        at java.base/java.lang.Thread.run(Thread.java:833)
```





#### 限流算法

1. **纯计数器限流**

   - 特点：简单粗暴，单机在 Java 中可用 Atomic 等原子类、分布式就 Redis incr
   - 缺点：一般的限流都是为了限制在时间间隔内的访问量，纯计数器没有时间单位

   ``` java
   synchronized boolean synchronize tryAcquire() {
       if (counter < threshold) {
           counter++;
           return true;
       } 	
       return false;
   }
   ```

2. **固定时间窗口计数器**

   - 缺点：窗口临界处流量双峰突发问题

   ![](./pictures/052.png)

   ``` java
   synchronized boolean tryAcquire() {
       long now = currentTimeMillis();
       if(now - lastAcquireTime > TimeWindow) {  // 是否过了时间窗口
           counter = 0;
           lastAcquireTime = now;
       }
       if (counter < threshold) {
           counter++;
           return true;
       }
       return false;
   }
   ```

3. **滑动时间窗口计数器**

   - 缺点：仍存在服务线程突增（流量突增）情况，**不够平滑**。

   ![](./pictures/053.png)

   ``` java
   synchronized boolean tryAcquire() {
       long now = currentTimeMillis();
   	long counter = getCounterInTimeWindow(now); // 当前时间为基准，往前获取窗口内的计数
       if (counter < threshold) {
           addToTimeWindow(now) // 记录
           return true;
       }
       return false;
   }
   ```

4. **漏斗算法**

   - 特点：**先减后加**，流出速率恒定，流入速率随意，满就溢出，**平滑处理**。

   ``` java
   int lastTimeStamp;
   int capacity; // 桶的容量
   int rate; // 水漏出的速度
   long water; // 当前水量(当前累积请求数)
   synchronized boolean tryAcquire()() {
       long now = currentTimeMillis();
       water = Math.max(0L, water - (now - lastTimeStamp) * rate); // 先流出水，计算剩余水量
       lastTimeStamp = now;
       if ((water + 1) < capacity) {
           water += 1;  
           return true;
       }
       return false; 
   }
   ```

5. **令牌算法**

   - 特点：**先加后减**，添加令牌速率恒定，消费速率随意，空就等待，**平滑处理**。
   - 比起漏斗算法，令牌算法**允许流量的突发**（起初桶里有令牌），后面再平滑处理。令牌算法对**用户更友好**，**业界采用最多**。

   ``` java
   int lastTimeStamp;
   int capacity; // 桶的容量
   int rate; // 令牌放入速度
   long tokens; // 当前令牌数量
   synchronized boolean tryAcquire()() {
       long now = currentTimeMillis();
       tokens = Math.min(capacity, tokens + (now - timeStamp) * rate);  // 先添加令牌，计算令牌数
       lastTimeStamp = now;
       if (tokens >= 1) {
           tokens -= 1;
           return true;
       }
       return false;
   }
   ```

   