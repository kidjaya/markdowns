# Spring 底层原理自学手册（面向 5 年经验开发者）

## 一、先定调：官方文档的定位

| 资源 | 定位 | 什么时候用 |
|---|---|---|
| Spring Reference Doc | **字典**，讲 what 和 how to use | 确认某个配置的确切语义、边界行为 |
| Spring Boot Reference Doc | **字典 + 部分科普** | 写 starter、调自动装配时必查 |
| 源码 + 断点 | **教材**，也是唯一的事实来源 | 理解 why 和 how it works |
| 手写 mini 轮子 | **考试**，把"看过"变成"懂了" | 每个阶段收尾时必做 |

一句话：**文档负责不让你错，源码负责让你懂，手写负责让你忘不掉。**

### 官方文档里真正值得精读的部分（不用全看）

- Core Technologies → **The IoC Container** 整章，尤其：
  - Container Overview
  - **Container Extension Points**（BeanPostProcessor / BeanFactoryPostProcessor，原理浓度最高）
  - Customizing the Nature of a Bean（生命周期回调顺序）
- Core Technologies → **Aspect Oriented Programming with Spring**
- Data Access → **Transaction Management**（声明式事务那节）
- Spring Boot Reference → **Creating Your Own Auto-configuration**
- Spring Boot Reference → Appendices: **Application Properties / Properties Migrator**

### 隐藏宝藏（很多人不知道）

- `AbstractApplicationContext.refresh()` 方法的 **javadoc 注释本身就是一张流程图**，几乎逐行解释了每步干什么。读源码前先读这段注释。
- `spring-boot-autoconfigure` jar 里，每个 `XxxAutoConfiguration` 类的类级 javadoc 写得极其详细，比任何书都准。
- 启动时加 `--debug`，Spring Boot 会打印完整的 **ConditionEvaluationReport**——哪些自动配置命中、哪些被否定、为什么。这是理解自动装配最快的工具。
- Actuator 的 `/actuator/conditions` 端点可以实时看装配决策。

---

## 二、版本建议：先啃 5.3.x / Boot 2.7.x

理由：**包名还是 `javax.*`，市面上 90% 的资料、书籍、视频能直接对上。**
Spring Framework 6 / Boot 3 全面迁移到 `jakarta.*`，看似只改了包名，实际会让每份参考资料都产生"为什么对不上"的干扰，白白消耗注意力。
等你把主线走通，再花半天看 Jakarta 迁移清单即可。

---

## 三、六个模块的源码入口清单（断点清单）

### 模块 1：IoC 容器启动主线

| 追问 | 断点 / 入口位置 |
|---|---|
| 容器到底启动了几步 | `AbstractApplicationContext.refresh()` —— 12 个方法调用，逐个单步走 |
| 配置类是怎么被解析成 Bean 定义的 | `ConfigurationClassPostProcessor.processConfigBeanDefinitions()` |
| @ComponentScan 扫了什么 | `ClassPathBeanDefinitionScanner.doScan()` |
| @Bean 方法什么时候被调用 | `ConfigurationClassBeanDefinitionReader.loadBeanDefinitionsForBeanMethod()` |
| Bean 在哪里真正被 new 出来 | `AbstractAutowireCapableBeanFactory.createBean()` → `doCreateBean()` |
| 三级缓存在哪 | `DefaultSingletonBeanRegistry.getSingleton()` / `addSingletonFactory()` |
| @Autowired 是谁填的 | `AutowiredAnnotationBeanPostProcessor.postProcessProperties()` |

**三个必须能说清的问题：**
1. `BeanFactory` 和 `ApplicationContext` 除了继承关系，行为差异在哪（国际化、事件、资源加载、自动后处理器注册）
2. 循环依赖为什么**三级**缓存而不是两级？—— 因为要处理 AOP 代理对象的提前引用，`singletonFactories` 里放的是 `ObjectFactory`，只有真正被循环引用时才执行 `getEarlyBeanReference` 生成代理
3. 为什么构造器循环依赖解不了、prototype 循环依赖解不了

### 模块 2：Bean 生命周期与扩展点

完整顺序（要能讲出逻辑，而不是背流程）：

```
实例化 createBeanInstance()
  → populateBean() 属性注入
  → setBeanName / setBeanFactory 等 Aware
  → BeanPostProcessor.postProcessBeforeInitialization()
  → InitializingBean.afterPropertiesSet() / @PostConstruct / init-method
  → BeanPostProcessor.postProcessAfterInitialization()   ← AOP 代理在这一步生成
  → 就绪可用
  → DisposableBean.destroy() / @PreDestroy / destroy-method
```

**关键认知：Spring 的所有高级特性都是 BeanPostProcessor 的子孙。**
`AutowiredAnnotationBeanPostProcessor`（注入）、`AnnotationAwareAspectJAutoProxyCreator`（AOP）、`AsyncAnnotationBeanPostProcessor`（异步）、`ScheduledAnnotationBeanPostProcessor`（定时任务）。
理解 BeanPostProcessor = 理解 Spring 一半的可扩展性。

### 模块 3：AOP 代理生成

| 追问 | 断点位置 |
|---|---|
| 代理在哪个扩展点被创建 | `AbstractAutoProxyCreator.postProcessAfterInitialization()` → `wrapIfNecessary()` |
| JDK Proxy 还是 CGLIB | `DefaultAopProxyFactory.createAopProxy()` —— 决策逻辑就这一段 |
| @EnableAspectJAutoProxy 干了什么 | `AspectJAutoProxyRegistrar` 注册了 `AnnotationAwareAspectJAutoProxyCreator` 的 BeanDefinition |
| 切面如何变成通知 | `ReflectiveAspectJAdvisorFactory.getAdvisor()` |
| 多个切面的执行顺序 | `ReflectiveMethodInvocation.proceed()` —— 递归 + 索引递增，这是责任链的核心 |

注意：Spring 5.2 起 `proxyTargetClass` 默认 true，即**默认走 CGLIB**。JDK Proxy 要求有接口且 `proxyTargetClass=false`。

### 模块 4：声明式事务

| 追问 | 断点位置 |
|---|---|
| @Transactional 怎么变成切面 | `ProxyTransactionManagementConfiguration` → `BeanFactoryTransactionAttributeSourceAdvisor` + `TransactionInterceptor` |
| 事务开启逻辑 | `TransactionAspectSupport.invokeWithinTransaction()` |
| Connection 怎么绑线程 | `TransactionSynchronizationManager`（一堆 ThreadLocal） |
| 传播行为怎么实现 | `AbstractPlatformTransactionManager.getTransaction()` / `handleExistingTransaction()` |
| REQUIRES_NEW 为何挂得起 | `suspend()` 把旧资源摘下来存进 `SuspendedResourcesHolder`，新的绑到 ThreadLocal |

**这一模块的达标标准：能不查资料就解释以下 8 种失效场景的根因**
同类内部调用、方法非 public、final/private/static 方法、异常被 catch 吞掉、rollbackFor 不对导致异常类型不匹配、数据库引擎不支持（MyISAM）、传播行为配错（如 SUPPORTS）、Bean 未被 Spring 管理。

### 模块 5：MVC 请求分发链路

| 追问 | 断点位置 |
|---|---|
| 请求总入口 | `DispatcherServlet.doService()` → `doDispatch()` |
| 怎么找到处理方法 | `AbstractHandlerMapping.getHandler()` → `RequestMappingHandlerMapping` |
| 为什么要 HandlerAdapter | `RequestMappingHandlerAdapter.handleInternal()` —— 因为 handler 形态多样（Controller Bean、方法、函数式），需要统一适配 |
| 参数从哪来的 | `HandlerMethodArgumentResolver` 家族 |
| @RequestBody 怎么转对象 | `RequestResponseBodyMethodProcessor` → `HttpMessageConverter` 链 |
| @ResponseBody 怎么转 JSON | `RequestResponseBodyMethodProcessor.handleReturnValue()` |
| 异常怎么被处理 | `HandlerExceptionResolver` 链 → `@ControllerAdvice` 由 `ExceptionHandlerExceptionResolver` 执行 |

### 模块 6：Boot 自动装配

| 追问 | 断点位置 |
|---|---|
| @SpringBootApplication 是什么 | 三个注解合成：`@SpringBootConfiguration` + `@EnableAutoConfiguration` + `@ComponentScan` |
| 自动配置类从哪读出来 | `AutoConfigurationImportSelector.getCandidateConfigurations()` —— 2.7 读 `META-INF/spring.factories`，3.x 读 `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` |
| 条件怎么判的 | `OnClassCondition` / `OnBeanCondition` / `OnPropertyCondition`，都实现 `Condition`；`(value, metadata)` 返回布尔 |
| Tomcat 什么时候起来 | `ServletWebServerApplicationContext.onRefresh()` → `createWebServer()` |
| run() 里做了什么 | `SpringApplication.run()` —— 准备环境 / 创建上下文 / refresh / 事件广播（`ApplicationListener` 由 spring.factories 加载） |

**实操建议：** 起一个最小 Boot 应用，加 `--debug` 启动，读 ConditionEvaluationReport 里的 "Positive matches" / "Negative matches"，再对照源码，半天就能彻底搞明白。

---

## 四、关联框架：用同一个套路快速击破

不要逐个通读源码，**只找「框架与 Spring 的衔接点」和「框架自己的核心抽象」**。

### MyBatis

| 关注点 | 位置 |
|---|---|
| 怎么接进 Spring | `SqlSessionFactoryBean.buildSqlSessionFactory()` —— 一个 FactoryBean |
| Mapper 接口为什么能注入 | `MapperScannerConfigurer` + `MapperFactoryBean`，每个 Mapper 注册成一个 FactoryBean |
| 接口没有实现类怎么执行 | `MapperProxy`（JDK 动态代理）→ `MapperMethod.execute()` |
| SqlSession 线程安全怎么保证 | `SqlSessionTemplate` + `SqlSessionInterceptor`（代理，每次调用从 ThreadLocal 取/建 session） |
| 一级缓存 | `BaseExecutor.localCache`（SqlSession 级别），Spring 整合后基本失效——因为每次操作后 session 就关了 |
| 二级缓存 | `CachingExecutor`，namespace 级别，TransactionalCacheManager 负责提交时才刷入 |

### Spring Cloud / 微服务组件

- **OpenFeign**：本质是 JDK Proxy + `MethodHandler`，核心入口 `feign.ReflectiveFeign`；负载均衡交给 `LoadBalancerClient`，替换点就在 `Client` 接口实现。
- **Nacos/注册发现**：核心是 `ServiceRegistry` 抽象 + 心跳任务；理解"客户端拉/推 + 本地缓存"即可，不必深挖一致性协议。

### 通用方法论
> 任何框架只需回答三个问题：**谁创建的？谁调用的？谁销毁的？**
> 答得出来，这个框架就算打通了。

---

## 五、推荐资源（按性价比排序）

**书**
1. 《Spring 揭秘》王福强 —— 讲**设计意图**最好的一本，已绝版，找 PDF 或二手。IoC 和 AOP 部分到现在都没过时。
2. 《Spring 源码深度解析》郝佳 —— 按 5.x 源码逐模块讲解，适合作案头的工具书，按需查。
3. 《Spring Boot 编程思想》马昕曦（小马哥）—— Boot 讲得最深，适合模块 6。

**视频（免费，B 站可搜）**
- 雷丰阳《Spring 注解驱动开发》—— 容器、AOP、MVC 讲得最清晰，入门源码第一选。
- 尚硅谷《Spring5 框架》—— 雷神同源，讲得更细，适合补充。

**项目**
- `code4craft/tiny-spring` —— 黄亿华写的迷你 Spring，IoC + AOP 一共不到 2000 行，照着他抄一遍，胜过看三本书。
- 自己写：一个支持 `@Component` / `@Autowired` / `@Transactional` / AOP 的容器，约 500 行。

---

## 六、执行节奏

1. **每天只追一个问题**，追到具体的那一行代码实现为止，不要泛读。
2. 建一个仅含 `spring-context` + `spring-boot-starter-web` 的最小工程，专门用来下断点，别在生产项目里 debug。
3. 每学完一个模块，**写一篇短文或给同事讲一遍**，讲不清楚的地方就是还没懂的地方。
4. 每周末回头看自己写的笔记，把"记下来的结论"改写成"为什么必须这样"。

最后一句：**五年经验最忌讳的是"再看一遍源码"这种通读式学习。** 你的优势是见过足够多的问题现场——带着那些踩过的坑（事务为什么没回滚、Bean 为什么没注入、配置为什么不生效）去反查源码，效率是通读的十倍。
