# 设计模式 · 核心知识清单（结合框架源码）

> 面向：5 年经验 Java 后端工程师，从"会用"走向"懂原理"
> 用法：当作自检清单，答不出来的条目就是下一步要补的地方
>
> 说明：本文不罗列 23 种模式的教科书定义。每一种只回答三件事：**框架在哪里用了它**、**什么时候该你用**、**最小实现长什么样**。

## 一、核心心智模型

1. **设计模式的本质是"给反复出现的解耦问题起名字"。** 不是为了炫技，是为了在代码评审时说"这里用个策略模式"，对方立刻懂你的意图。模式的价值在于**沟通效率**，不在于代码本身。
2. **Spring 框架本身就是一本设计模式教科书。** IoC = 工厂 + 单例 + 模板方法；AOP = 代理 + 责任链；MVC = 前端控制器 + 适配器 + 策略。读懂 Spring 源码，23 种模式自然就懂了大半。
3. **容易混淆的模式，区分点在"意图"而不是"结构"。** 代理和装饰器的 UML 长得几乎一样，区别只在于：代理**控制访问**（我可以不让你调），装饰器**增强功能**（我一定让你调，只是加点东西）。分不清时问意图。
4. **大部分业务代码只需要 7 种模式。** 工厂/策略/模板方法/代理/责任链/观察者/建造者，这七种覆盖了 80% 的场景。剩下的多数是框架作者的事。
5. **警惕"为了模式而模式"。** 一个类就够了的地方硬套抽象工厂，只会增加阅读成本。判断标准很简单：**这个位置未来会不会有多种实现？** 不会就别抽象。

## 二、必懂机制

### 2.1 创建型

#### 2.1.1 单例模式

**意图**：保证一个类只有一个实例，并提供全局访问点。

**框架中的位置**：
- Spring 的 `singleton` scope：`DefaultSingletonBeanRegistry.singletonObjects`（一个 `ConcurrentHashMap`）
- **关键点：Spring 的单例是"容器级"的，不是 JVM 级的。** 同一个 `SingletonBeanRegistry` 内唯一，但多个 `ApplicationContext`（或父子容器）里同一个 BeanDefinition 会有多个实例。而 GOF 单例是 ClassLoader 级的。
- JDK 里的 `Runtime.getRuntime()`、`System` 的 `SecurityManager`、`LogManager`

**源码位置**：
```java
// org.springframework.beans.factory.support.DefaultSingletonBeanRegistry
private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256);
// getSingleton(String beanName, ObjectFactory<?> singletonFactory) —— 双重检查 + synchronized(singletonObjects)
```

**五种写法与各自的坑**：

| 写法 | 线程安全 | 懒加载 | 防反射/反序列化 | 推荐度 |
|---|---|---|---|---|
| 饿汉（static final 字段） | ✅ | ❌ | ❌ | 简单场景可用 |
| DCL + `volatile` | ✅ | ✅ | ❌ | 经典面试答案，**注意 volatile 不能少** |
| 静态内部类 | ✅ | ✅ | ❌ | **推荐**，JVM 类加载机制保证线程安全，写法最优雅 |
| 枚举 | ✅ | ❌（类加载即实例化） | ✅ | **最安全**，《Effective Java》推荐，**唯一能防反射攻击** |
| `Spring @Component` | ✅（容器保证） | 看 `lazy-init` | — | **业务代码首选**，交给容器管生命周期 |

```java
// 静态内部类（推荐）
public class Singleton {
    private Singleton() {}
    private static class Holder {
        private static final Singleton INSTANCE = new Singleton();
    }
    public static Singleton getInstance() { return Holder.INSTANCE; }
}

// 枚举（最安全，能防反射 newInstance 和反序列化创建新实例）
public enum Singleton {
    INSTANCE;
    public void doSomething() { }
}
```

**什么时候该你用**：
- 实际业务里**绝大多数情况交给 Spring 容器管理**，不要手写
- 手写场景通常是：框架的入口类、全局配置持有者、连接池管理者
- **反模式**：用单例持有可变状态（尤其请求的上下文）——这是典型的并发 bug 来源

**与其他模式的关系**：Spring 的单例 + `FactoryBean` + 三级缓存，合起来实现了 IoC 容器，也顺带解决了循环依赖。

---

#### 2.1.2 工厂模式（简单工厂 / 工厂方法 / 抽象工厂）

**意图**：把对象创建过程封装起来，让调用方不关心具体类型。

**框架中的位置（这是最重要的一组）**：

| 位置 | 说明 |
|---|---|
| `BeanFactory` | **工厂方法**。最顶层抽象，定义了 `getBean()` |
| `DefaultListableBeanFactory` | 具体工厂实现，负责创建全部 Bean |
| **`FactoryBean<T>`** | **最容易被搞混的一个** |
| `SqlSessionFactory` / `SqlSessionFactoryBuilder` | MyBatis 的工厂 + 建造者组合 |
| `LoggerFactory`（slf4j） | 静态工厂，屏蔽底层的日志实现 |
| `DriverManager` | JDBC 的工厂 + 桥接 |

**`BeanFactory` vs `FactoryBean` —— 面试必考，务必讲清**：

```java
public interface FactoryBean<T> {
    T getObject() throws Exception;      // 返回真正的对象
    Class<?> getObjectType();
    default boolean isSingleton() { return true; }
}
```

- `BeanFactory`：**容器**，是"生产 Bean 的工厂"。`context.getBean()` 主体就是它
- `FactoryBean`：**一个特殊的 Bean**，它本身是 Bean，但它 `getObject()` 返回的东西才是真正要注册的对象
- **取对象时的差异**：
  ```java
  context.getBean("myFactoryBean")     // 拿到 getObject() 的结果
  context.getBean("&myFactoryBean")    // 加 & 前缀，拿到 FactoryBean 本身
  ```
- **典型应用**：MyBatis-Spring 的 `SqlSessionFactoryBean`（本身是 FactoryBean，产出 `SqlSessionFactory`）、`MapperFactoryBean`（产出 Mapper 接口的代理对象）、Dubbo 的 `ReferenceBean`（产出远程服务代理）
- **为什么需要它**：当对象的创建逻辑很复杂（需要读配置、建连接、生成代理），没法靠反射 new 出来时，用 FactoryBean 接管创建过程

**源码入口**：
```
org.springframework.beans.factory.support.AbstractBeanFactory.doGetBean()
  → getSingleton(beanName)                       // 先查缓存
  → getObjectForBeanInstance(...)                // 关键：如果是 FactoryBean，调用 getObject()
     → FactoryBeanRegistrySupport.getObjectFromFactoryBean()
```

**什么时候该你用**：
- 创建对象需要复杂的前置逻辑（读配置、建连接、生成代理）→ `FactoryBean`
- 根据类型码/参数返回不同实现 → 简单工厂或工厂方法
- 有一整套相关的产品要统一创建 → 抽象工厂（罕见）

**手写最小实现**：
```java
// 工厂方法：让子类决定创建什么
public abstract class Parser {
    public Result parse(String file) {           // 模板方法 + 工厂方法
        byte[] data = read(file);
        return doParse(data);                    // 子类实现具体解析
    }
    protected abstract Result doParse(byte[] data);
}
public class JsonParser extends Parser { protected Result doParse(byte[] d) { ... } }
```

---

#### 2.1.3 建造者模式

**意图**：分步构造复杂对象，尤其是有大量可选参数的场景。

**框架中的位置**：
- `SqlSessionFactoryBuilder`（MyBatis）
- `BeanDefinitionBuilder`（Spring）
- `RestTemplateBuilder` / `WebClient.Builder`（Spring）
- JDK 的 `StringBuilder`、`Stream.Builder`、`HttpRequest.newBuilder()`（Java 11）
- Lombok 的 `@Builder`

**什么时候该你用**：
1. **构造方法参数超过 4 个，且其中多个可选** —— 这是最直接的判断信号
2. 对象构造有多个阶段（如先设 A 再设 B，最后 build）
3. 需要构造不可变对象（只有全参构造 + builder，没有 setter）

```java
public final class HttpRequest {
    private final String url;
    private final String method;
    private final Map<String, String> headers;
    private final Duration timeout;
    private final String body;

    private HttpRequest(Builder b) { ... }   // 私有构造

    public static final class Builder {
        private String url; private String method = "GET";
        private Map<String, String> headers = new HashMap<>();
        private Duration timeout = Duration.ofSeconds(3);
        private String body;
        public Builder url(String url) { this.url = url; return this; }
        public Builder header(String k, String v) { headers.put(k, v); return this; }
        public Builder timeout(Duration d) { this.timeout = d; return this; }
        public HttpRequest build() {
            if (url == null) throw new IllegalStateException("url required");  // 校验在 build 里
            return new HttpRequest(this);
        }
    }
}
// 用法：HttpRequest.builder().url("...").header("a","b").timeout(Duration.ofSeconds(5)).build();
```

**与工厂的区别**：建造者关注**怎么一步步组装**，工厂关注**生产哪个产品**。Builder 通常最后才产出对象，且产出后可以链式调用。

---

#### 2.1.4 原型模式

**意图**：通过复制现有对象来创建新对象，跳过昂贵的初始化。

**框架中的位置**：
- Spring 的 `@Scope("prototype")` —— **注意**：这只是"每次都新建"，与 GOF 的"通过 clone 复制"不是一回事，名字容易误导
- `Object.clone()` / `Cloneable`
- MyBatis 的 `Configuration` 里 ShardingSphere 等框架复制 SQL 模板对象

**关键坑：深拷贝 vs 浅拷贝**
```java
class Order implements Cloneable {
    private List<Item> items;
    @Override protected Order clone() {
        try {
            Order o = (Order) super.clone();       // 浅拷贝：items 仍指向同一个 List
            o.items = new ArrayList<>(this.items); // 必须手动深拷贝引用类型
            return o;
        } catch (CloneNotSupportedException e) { throw new AssertionError(); }
    }
}
```
**生产建议**：几乎不要用 `Cloneable`（接口设计有缺陷，没有 clone 方法；且深拷贝要手写）。用拷贝构造方法或序列化/工具类。

---

### 2.2 结构型

#### 2.2.1 代理模式（最核心）

**意图**：为目标对象提供替身，**控制对它的访问**。

**框架中的位置 —— 这是 Java 后端最高频的模式**：

| 位置 | 代理类型 | 说明 |
|---|---|---|
| **Spring AOP** | JDK 动态代理 / CGLIB | `@Transactional`、`@Cacheable`、`@Async` 全靠它 |
| **MyBatis Mapper** | JDK 动态代理 | **Mapper 接口没有实现类，代理对象执行 SQL** |
| Dubbo Consumer | JDK 动态代理 / javassist | `ReferenceConfig.get()` 生成远程服务代理 |
| Feign | JDK 动态代理 | `@FeignClient` 接口 → HTTP 调用 |
| Retofit / JPA Repository | 动态代理 | 同上思路 |
| RPC 客户端 stub | 动态代理 | 屏蔽网络细节 |

**JDK 动态代理 vs CGLIB 的完整对比**：

| | JDK 动态代理 | CGLIB |
|---|---|---|
| 实现方式 | 运行时生成实现接口的代理类 | 继承目标类，生成子类（ASM 操作字节码） |
| 要求 | **目标必须有接口** | 目标可无接口，但**不能是 final 类/方法** |
| JDK 8 性能 | 反射调用，较慢 | **快**（FastClass 索引调用） |
| JDK 9+ 性能 | 对反射做了优化，**已接近甚至超过 CGLIB** | 相当 |
| 生成开销 | 小 | 大（要操作字节码） |
| 依赖 | JDK 内置 | 需要 `spring-core` 内联的 cglib 包 |

**Spring 的选择逻辑**（`DefaultAopProxyFactory.createAopProxy`）：
```java
// org.springframework.aop.framework.DefaultAopProxyFactory（简化后）
if (config.isOptimize() || config.isProxyTargetClass() || hasNoUserSuppliedProxyInterfaces(config)) {
    Class<?> targetClass = config.getTargetClass();
    if (targetClass == null) throw new AopConfigException("TargetSource cannot determine target class");
    // 即使开了 proxyTargetClass，目标本身是接口或已是 JDK 代理，仍走 JDK 代理
    if (targetClass.isInterface() || Proxy.isProxyClass(targetClass)) {
        return new JdkDynamicAopProxy(config);
    }
    return new ObjenesisCglibAopProxy(config);       // CGLIB
}
else {
    return new JdkDynamicAopProxy(config);           // 默认 JDK 动态代理
}
```
实际规则（按判定顺序）：
1. 目标没有实现任何接口 → **CGLIB**（JDK 代理没法下手）
2. `spring.aop.proxy-target-class=true`（**Spring Boot 2.x 起默认为 true**）→ CGLIB
3. 其余情况（实现了接口且未强制 CGLIB）→ **JDK 动态代理**
4. 目标类本身是接口，或已经是代理类 → JDK 动态代理

**为什么 Boot 改成默认 CGLIB**：解决了"注入实现类而非接口时，JDK 代理转型失败"的经典报错（`The bean 'xxx' could not be injected because it is a JDK dynamic proxy`）。

**手写最小实现（JDK 动态代理）**：
```java
public class LogInvocationHandler implements InvocationHandler {
    private final Object target;
    public LogInvocationHandler(Object target) { this.target = target; }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        long start = System.nanoTime();
        try {
            return method.invoke(target, args);        // 反射调用真实方法
        } finally {
            log.info("{} cost {}ms", method.getName(), (System.nanoTime() - start) / 1_000_000);
        }
    }
}
OrderService proxy = (OrderService) Proxy.newProxyInstance(
    OrderService.class.getClassLoader(),
    new Class[]{OrderService.class},
    new LogInvocationHandler(new OrderServiceImpl()));
```

**MyBatis Mapper 代理的精妙之处**（面试高频）：
```java
// org.apache.ibatis.binding.MapperProxy.invoke()
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    try {
        if (Object.class.equals(method.getDeclaringClass())) {
            return method.invoke(this, args);          // toString/hashCode/equals 直接本地处理
        }
        return cachedInvoker(method).invoke(proxy, method, args, sqlSession);
    } catch (Throwable t) { throw ExceptionUtil.unwrapThrowable(t); }
}
// cachedInvoker 最终调 MapperMethod.execute(sqlSession, args)
//   → 根据 SQL 类型（SELECT/INSERT/UPDATE/DELETE）走 sqlSession.selectOne/insert/...
```
**三层结构**：`MapperProxy`（InvocationHandler）→ `MapperMethod`（封装 SQL 与签名）→ `SqlSession`（执行）。
**这就是为什么 Mapper 接口不需要实现类** —— 方法调用被代理拦截后，转成了一次 SQL 执行。

**代理的经典坑 —— 自调用导致事务失效**：
```java
@Service
public class OrderServiceImpl {
    @Transactional
    public void methodA() { this.methodB(); }        // ❌ this 是原始对象，不是代理，methodB 的事务不生效

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void methodB() { }
}
```
**原因**：事务是通过代理对象织入的，`this` 指向的是被代理的原始对象，绕过了代理。
**解法**（三种，按推荐度）：
1. 拆到另一个 Service（最干净）
2. `AopContext.currentProxy()` 拿到当前代理：`((OrderServiceImpl) AopContext.currentProxy()).methodB()`（需开 `exposeProxy=true`）
3. 注入自己：`@Autowired @Lazy private OrderServiceImpl self;`（有循环依赖风险）

---

#### 2.2.2 装饰器模式

**意图**：**动态地给对象添加职责**，不改变其接口。

**与代理的结构几乎相同，区别在意图**：

| | 代理 | 装饰器 |
|---|---|---|
| 意图 | **控制访问**（可以拦截、可以不让你调） | **增强功能**（一定让你调，只是包一层） |
| 关注点 | 把 client 和被代理对象**隔离开** | 一层套一层，**递归组合** |
| 创建时机 | 通常在编译时/启动时确定 | 运行时动态层层叠加 |

**框架中的位置**：
- **JDK IO 流**：`new DataInputStream(new BufferedInputStream(new FileInputStream("a.txt")))` —— 最经典的装饰器
- **Dubbo 的 Wrapper 机制**（特别典型）：
  ```java
  // Protocol 接口有三个包装类，层层包裹
  ProtocolFilterWrapper → ProtocolListenerWrapper → QosProtocolWrapper → DubboProtocol
  // ExtensionLoader 检测到构造函数是 (Protocol)，自动识别为包装类并嵌套组装
  ```
  源码：`ExtensionLoader.loadExtensionClasses()` → `cachedWrapperClasses`
- Spring Security 的 `HttpServletRequestWrapper`
- MyBatis 的 `Executor` 被 `CachingExecutor` 包装（二级缓存）

**手写最小实现**：
```java
public interface DataSource { Connection getConnection(); }
public abstract class DataSourceDecorator implements DataSource {
    protected DataSource delegate;
    public DataSourceDecorator(DataSource delegate) { this.delegate = delegate; }
    public Connection getConnection() { return delegate.getConnection(); }
}
public class LoggingDataSource extends DataSourceDecorator {
    public LoggingDataSource(DataSource d) { super(d); }
    @Override public Connection getConnection() {
        log.info("acquire connection");
        return delegate.getConnection();       // 增强后仍交给被装饰者
    }
}
// 可无限叠加：new MetricsDataSource(new LoggingDataSource(realDataSource))
```

**什么时候该你用**：需要在不改原有类的前提下，横向添加多种可选增强（日志、限流、加密、缓存），且这些增强可以自由组合。

---

#### 2.2.3 适配器模式

**意图**：把一个接口转换成客户端期望的另一个接口，让原本不兼容的类可以协作。

**框架中的位置**：

| 位置 | 适配什么 |
|---|---|
| **Spring MVC `HandlerAdapter`** | **最典型**。把各种形态的处理器（`@RequestMapping` 方法、`Controller` 实现类、`HttpRequestHandler`、静态资源）统一适配成可被 `DispatcherServlet` 调用的形式 |
| `DispatcherServlet.doDispatch()` | `HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler()); ... ha.handle(req, res, handler)` |
| slf4j |  `slf4j-log4j12` / `logback-classic` 把各种日志框架适配成统一的 `Logger` 接口 |
| Spring `AdvisorAdapter` | 把各种 Advice（`MethodBeforeAdvice` / `AfterReturningAdvice` / `ThrowsAdvice`）适配成统一的 `MethodInterceptor` |
| `Callable` → `Runnable` | `Executors.callable(runnable)` |
| `Arrays.asList()` | 数组 → List |

**Spring MVC 的 HandlerAdapter 详解**（面试高频）：
```java
public interface HandlerAdapter {
    boolean supports(Object handler);                       // 我能不能处理这个 handler
    ModelAndView handle(HttpServletRequest req, HttpServletResponse res, Object handler) throws Exception;
    long getLastModified(HttpServletRequest req, Object handler);
}
```
内置四个实现：
- `RequestMappingHandlerAdapter` —— **处理 `@RequestMapping` 注解方法（现在最常用）**
- `HandlerFunctionAdapter` —— WebFlux 函数式端
- `HttpRequestHandlerAdapter` —— 处理实现了 `HttpRequestHandler` 的 Bean（如静态资源）
- `SimpleControllerHandlerAdapter` —— 处理实现了 `Controller` 接口的 Bean（老式）

**这解决什么问题**：Controller 的形态千差万别，`DispatcherServlet` 不可能为每种形态写 if-else。加个适配器层，新形态只要注册一个新 Adapter，Servlet 核心逻辑不用动。

**什么时候该你用**：
- 对接第三方 SDK，它的接口和你系统的接口对不上
- 重构老代码，老接口不能删但要兼容新实现
- 统一多个相似库的差异（如同时支持多个短信厂商）

**注意**：发现自己在写大量的 `xxxToYyyAdapter` 时，先确认是不是真的不能改接口——能改接口就直接改，适配器是"改不动"时的无奈之举。

---

#### 2.2.4 门面模式（Facade）

**意图**：为复杂的子系统提供一个简化的统一入口。

**框架中的位置**：
- **slf4j 是双重身份**：对外是门面（统一 API），对内是适配器
- `DispatcherServlet` —— 对客户端屏蔽了 HandlerMapping/HandlerAdapter/ViewResolver 等一整套组件
- Spring Cloud OpenFeign 对 Ribbon/LoadBalancer/编码器/解码器的封装
- `JdbcTemplate` —— 对 JDBC 的 `Connection/Statement/ResultSet` + 异常转换 + 连接管理做了封装

**与适配器的区别**：适配器是**转换接口**让两者能协作；门面是**提供一个新接口**让复杂子系统变简单。

**什么时候该你用**：一个业务操作需要调 5 个服务的 8 个方法时，对外只暴露一个 `createOrder()`。

---

#### 2.2.5 桥接模式

**意图**：把抽象和实现解耦，让两者可以独立演进（避免类爆炸）。

**框架中的位置**：
- **JDBC 是最经典的例子**：`DriverManager` + `java.sql.Driver` 接口。Java 只定义接口（抽象），各数据库厂商提供实现（`mysql-connector-j`、`ojdbc`）。**换数据库只换驱动 jar，代码不动**
- slf4j 的 `Logger` 接口与具体实现的绑定
- 抽象维度的例子：`消息`（普通/加急/特急）× `发送方式`（短信/邮件/推送）—— 用继承要 9 个类，用桥接只要 3+3

```java
// 不用桥接：UrgentSmsMessage / UrgentEmailMessage / NormalSmsMessage ... 类爆炸
// 用桥接：抽象层持有一个实现层的引用
abstract class Message {
    protected MessageSender sender;      // 桥接的关键：组合而非继承
    public Message(MessageSender sender) { this.sender = sender; }
    public abstract void send(String content);
}
class UrgentMessage extends Message {
    public void send(String c) { sender.send("[加急] " + c); }
}
new UrgentMessage(new SmsSender());
new UrgentMessage(new EmailSender());    // 两个维度独立扩展
```

**什么时候该你用**：发现类有两个以上独立变化的维度，且用继承实现会导致类数量相乘。

---

#### 2.2.6 享元模式

**意图**：共享对象，减少内存占用（分离内部状态与外部状态）。

**框架/语言中的位置**：
- **String 常量池**：字符串字面量复用（这是为什么 String 必须不可变）
- **Integer 缓存**：`-128~127` 由 `IntegerCache` 缓存，`Integer.valueOf(127) == Integer.valueOf(127)` 为 true
- **数据库连接池 / 线程池**：线程对象本身被复用，任务状态是外部传入的
- `Boolean.TRUE` / `Boolean.FALSE` 两个常量实例

**什么时候该你用**：系统里有大量高度相似的对象，且其中大部分状态可以外提。典型如：游戏中的粒子、文本编辑器里的字符样式、缓存中的元数据。

---

#### 2.2.7 组合模式

**意图**：把对象组合成树形结构，让调用方对单个对象和组合对象一视同仁。

**框架中的位置 —— MyBatis 的 SqlNode 是最漂亮的例子**：
```java
// org.apache.ibatis.scripting.xmltags
public interface SqlNode {
    boolean apply(DynamicContext context);    // 每个节点往 context 里拼 SQL 片段
}
// 叶子节点
class TextSqlNode implements SqlNode { ... }        // 静态文本
class IfSqlNode implements SqlNode {                // <if test="...">
    public boolean apply(DynamicContext ctx) {
        if (evaluator.evaluateBoolean(test, ctx.getBindings())) {
            contents.apply(ctx);                    // 递归调用子节点
            return true;
        }
        return false;
    }
}
// 组合节点 —— 它自己也是 SqlNode，可以包含其他 SqlNode
class MixedSqlNode implements SqlNode {
    private final List<SqlNode> contents;
    public boolean apply(DynamicContext ctx) {
        contents.forEach(node -> node.apply(ctx));
        return true;
    }
}
```
`<where>`、`<trim>`、`<foreach>`、`<choose>` 都是不同的 Node 实现，**任意嵌套组合**，最终用统一的 `apply()` 拼出完整 SQL。这就是组合模式的教科书实现。

**其他例子**：Spring 的 `BeanDefinition` 树、JSON 树（`JsonNode`）、文件系统 Tree、菜单树、规则引擎的 AND/OR 组合

**手写最小实现**：
```java
interface Rule { boolean match(Request req); }
class AndRule implements Rule {
    private final List<Rule> rules;
    public boolean match(Request req) { return rules.stream().allMatch(r -> r.match(req)); }
}
class OrRule implements Rule { ... }
// 使用：new AndRule(List.of(new IpRule("192.*"), new NotRule(new PathRule("/admin"))))
```

---

### 2.3 行为型

#### 2.3.1 模板方法（业务代码中最好用的一个）

**意图**：定义算法骨架，把某些步骤延迟到子类实现。

**框架中的位置（到处都是）**：

| 位置 | 模板结构 |
|---|---|
| **`AbstractApplicationContext.refresh()`** | **最经典**。12 个步骤固定顺序，其中 `onRefresh()` / `postProcessBeanFactory()` 留给子类 |
| **`JdbcTemplate.execute()`** | 固定流程：获取连接 → 创建 Statement → 执行 → 处理结果 → 关闭连接 → 异常转换。只留 `PreparedStatementCallback.doInPreparedStatement()` 给调用方 |
| `RestTemplate` / `RabbitTemplate` / `RedisTemplate` | 都是 Template 命名，同构的思路 |
| **`AQS`** | `acquire()` 定义了完整的排队/阻塞/唤醒流程，只留 `tryAcquire()` 给子类 —— 所以叫 AbstractQueuedSynchronizer |
| MyBatis `BaseExecutor` | 定义了 query/update 的骨架，把 `doQuery()` / `doUpdate()` 留给 `SimpleExecutor` / `ReuseExecutor` / `BatchExecutor` |
| Servlet 的 `HttpServlet.service()` | 分发到 doGet/doPost |

**AQS 的模板方法细节**（面试必考）：
```java
// AbstractQueuedSynchronizer
public final void acquire(int arg) {                   // final：骨架不可改
    if (!tryAcquire(arg) &&                            // 钩子：子类实现抢锁逻辑
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg)) // 骨架：入队、挂起、被唤醒
        selfInterrupt();
}
protected boolean tryAcquire(int arg) { throw new UnsupportedOperationException(); }
// ReentrantLock.Sync.nonfairTryAcquire() 实现 tryAcquire
// Semaphore / CountDownLatch / ReentrantReadWriteLock 各自实现自己的 tryAcquireShared
```

**为什么 Spring 的 Template 类这么好用**：它把"必须做对的重复劳动"（资源管理、异常转换、事务边界）固化在父类里，调用方只写真正有差异的那几行。这是 DRY 的最佳实践。

**手写最小实现**：
```java
public abstract class AbstractImportTask {
    public final void run() {                     // final 防止子类改流程
        List<Row> rows = read();
        validate(rows);
        List<Entity> entities = convert(rows);    // 钩子：各类型不同
        write(entities);
        notify(collectSummary(entities));            // 钩子：可选覆盖
    }
    protected abstract List<Entity> convert(List<Row> rows);
    protected void notify(Summary s) { }          // 钩子方法（空实现，可选覆盖）
}
```

**什么时候该你用**：这段流程有多个变体，**且流程顺序是固定的**。变体多且组合复杂时改用策略模式。

---

#### 2.3.2 策略模式（消除 if-else 最有效的手段）

**意图**：定义一系列可互换的算法，运行时选择。

**框架中的位置**：

| 位置 | 策略接口 → 实现 |
|---|---|
| **Dubbo LoadBalance** | `RandomLoadBalance` / `RoundRobinLoadBalance` / `LeastActiveLoadBalance` / `ShortestResponseLoadBalance` / `ConsistentHashLoadBalance`，由 URL 参数 `loadbalance` 选择 |
| Spring `Resource` | `ClassPathResource` / `FileSystemResource` / `UrlResource` / `ByteArrayResource` |
| MyBatis `ExecutorType` | `SIMPLE` / `REUSE` / `BATCH` |
| Spring Security 认证 | 多种 `AuthenticationProvider` 并存，逐个尝试 |
| JDK `Comparator` | Lambda 就是最轻量的策略传递 |
| Spring `InstantiationStrategy` | `SimpleInstantiationStrategy` / `CglibSubclassingInstantiationStrategy` |

**与模板方法的区别**（重要）：
- **模板方法**：流程骨架固定，只改其中几步（**继承**实现，编译期确定）
- **策略**：整个算法都可替换（**组合**实现，运行时切换）

**Spring 中最优雅的用法 —— 把策略实现类全部注册成 Bean，用 Map 自动注入**：
```java
public interface PaymentStrategy {
    String getType();                  // 策略标识
    PayResult pay(Order order);
}

@Component class AliPayStrategy implements PaymentStrategy {
    public String getType() { return "ALIPAY"; }
    public PayResult pay(Order o) { ... }
}
@Component class WechatPayStrategy implements PaymentStrategy {
    public String getType() { return "WECHAT"; }
    public PayResult pay(Order o) { ... }
}

@Service
public class PaymentService {
    // Spring 自动按 Bean 名 → Bean 实例注入 Map，key 是 Bean 名
    private final Map<String, PaymentStrategy> strategies;
    public PaymentService(Map<String, PaymentStrategy> strategies) { this.strategies = strategies; }

    public PayResult pay(String type, Order order) {
        PaymentStrategy s = Optional.ofNullable(strategies.get(type))
            .orElseThrow(() -> new UnsupportedOperationException("不支持的支付方式: " + type));
        return s.pay(order);
    }
}
```
**这个写法的价值**：新增支付方式只需加一个 `@Component` 类，`PaymentService` 一行代码都不用改。彻底干掉 `switch/if-else`。

**什么时候该你用**：出现 `if (type.equals("A")) {...} else if (type.equals("B")) {...}` 且未来还会加 C、D、E 的时候。**这是最容易识别的重构信号。**

---

#### 2.3.3 责任链模式

**意图**：把请求沿着处理者链传递，每个处理者决定处理还是转交下一个。

**框架中的位置（这个模式在中间件里遍地开花）**：

| 位置 | 说明 |
|---|---|
| **Servlet Filter 链** | `FilterChain.doFilter()` 的递归调用，`javax.servlet.Filter` |
| **Spring Security** | `FilterChainProxy` 里几十个 Filter（认证、授权、CSRF、Session 管理...） |
| **Spring MVC** | `HandlerExecutionChain` 中的 `HandlerInterceptor`（preHandle / postHandle / afterCompletion） |
| **Dubbo Filter 链** | `ProtocolFilterWrapper.buildInvokerChain()` 串联所有 `@Activate` Filter |
| **MyBatis** | `InterceptorChain.pluginAll()` 串联分页、SQL 打印等插件 |
| **Netty** | `ChannelPipeline`（变体，双向链表，Inbound/Outbound 两条方向） |
| **Sentinel** | `SlotChain`（NodeSelectorSlot → ClusterBuilderSlot → StatisticSlot → FlowSlot → DegradeSlot） |
| Spring AOP | 拦截器链 `ReflectiveMethodInvocation.proceed()` |

**Servlet Filter 的执行顺序（典型的递归回调）**：
```
xxxFilter.doFilter(req, res, chain) {
    // before 代码（正序执行）
    chain.doFilter(req, res);        // 递归调用下一个 Filter
    // after 代码（倒序执行）
}
```
这就形成了**洋葱模型**：请求进来时正序穿过所有 Filter 的前半段，到达 Servlet，返回时倒序穿过所有 Filter 的后半段。理解这一点，`@Transactional` 写在 Filter 里不生效、异常被外层吞掉这类问题就都解释得通了。

**手写最小实现**：
```java
public abstract class Handler {
    private Handler next;
    public Handler setNext(Handler next) { this.next = next; return next; }
    public final void handle(Context ctx) {
        if (process(ctx)) return;                    // 处理了就终止
        if (next != null) next.handle(ctx);          // 否则转交
    }
    protected abstract boolean process(Context ctx);
}
// 组装：auth.setNext(rateLimit).setNext(bizValidate).setNext(biz);
```

**变体对比**：
- **纯责任链**：有一个处理者处理完就终止（如上面的例子）
- **不纯责任链**：所有处理者都处理一遍（如 Netty Pipeline、AOP 拦截器链）
- **Spring AOP 的 proceed() 典型实现**：
  ```java
  // ReflectiveMethodInvocation.proceed()
  if (this.currentInterceptorIndex == this.interceptorsAndDynamicMethodMatchers.size() - 1) {
      return invokeJoinpoint();                      // 到链尾，调真实方法
  }
  Object interceptor = ++this.currentInterceptorIndex ...;
  return ((MethodInterceptor) interceptor).invoke(this);   // 把自己(this)传进去，形成递归
  ```

**什么时候该你用**：一个请求要经过多个独立的处理步骤，且这些步骤需要**可插拔、可排序、可动态增删**（审批流、风控规则链、参数校验链、HTTP 拦截器）。

---

#### 2.3.4 观察者 / 发布订阅

**意图**：一个对象状态变化时，自动通知所有依赖它的对象。

**框架中的位置**：
- **Spring 事件机制**：`ApplicationEventPublisher` + `ApplicationListener` / `@EventListener`
- **Spring 内置事件**：`ContextRefreshedEvent`、`ContextStartedEvent`、`ContextClosedEvent`、`RequestHandledEvent`
- **Guava EventBus** / **Spring 的 `ApplicationEventMulticaster`**
- ZooKeeper 的 **Watcher** —— 分布式版的观察者
- **MQ 的发布订阅** —— 跨进程版的观察者
- JDK 的 `Observer` / `Observable`（**JDK 9 已废弃**，别用）

```java
// 定义事件（Spring 4.2+ 不需要继承 ApplicationEvent 了）
public record OrderPaidEvent(Long orderId, BigDecimal amount) {}

// 发布
@Service
public class OrderService {
    @Autowired ApplicationEventPublisher publisher;
    public void pay(Long orderId) {
        // 业务逻辑
        publisher.publishEvent(new OrderPaidEvent(orderId, amount));
    }
}

// 订阅（多个监听器互不知晓，完全解耦）
@Component
public class CouponListener {
    @EventListener
    public void onPaid(OrderPaidEvent e) { /* 发券 */ }
}
@Component
public class StatsListener {
    @Async                                        // 异步执行，不阻塞主流程
    @EventListener
    public void onPaid(OrderPaidEvent e) { /* 更新统计 */ }
}
@Component
public class NotifyListener {
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)  // 事务提交后才触发
    @EventListener
    public void onPaid(OrderPaidEvent e) { /* 发通知 */ }
}
```

**三个必须知道的坑**：
1. **默认同步执行**：`@EventListener` 默认在同一个线程里同步跑，监听器里的慢操作会拖慢主流程 → 加 `@Async`
2. **默认同一个事务**：监听器抛异常会导致发布方的业务回滚 → 加 `@TransactionalEventListener(AFTER_COMMIT)` 或 try-catch
3. **ApplicationEventMulticaster 默认没有异常隔离**：某个监听器挂了，后续的监听器不会被调用

**观察者 vs 发布订阅**：
- **观察者**：被观察者直接持有观察者引用，**同步**、紧耦合（Spring 事件就是这样）
- **发布订阅**：中间有个事件总线/消息代理，**异步**、完全解耦（MQ）

**什么时候该你用**：一个动作要触发多个后续处理，且这些处理**不应该影响主流程**（注册后发欢迎邮件、加积分、初始化默认数据、同步搜索引擎）。

---

#### 2.3.5 迭代器模式

**框架中的位置**：JDK 的 `Collection.iterator()`、`Iterator<E>` 及其 `forEachRemaining`。

**必须知道的坑 —— fail-fast**：
```java
for (String s : list) {
    if (s.equals("x")) list.remove(s);       // ❌ ConcurrentModificationException
}
```
**根因**：`ArrayList.Itr` 持有 `expectedModCount`，创建迭代器时等于 ArrayList 的 `modCount`。调用 `list.remove()` 修改了 `modCount` 但没更新迭代器的 `expectedModCount`，下次 `next()` 检查不一致就抛异常。

**正确写法**：
```java
list.removeIf("x"::equals);                  // ✅ 推荐
// 或
Iterator<String> it = list.iterator();
while (it.hasNext()) { if (...) it.remove(); }   // ✅ 用迭代器自己的 remove
// 或 CopyOnWriteArrayList（fail-safe，遍历的是快照）
```

---

#### 2.3.6 状态模式

**意图**：对象行为随内部状态改变而改变，看起来像是修改了类。

**与策略模式的区分（易混）**：**结构完全一样，意图不同**
- **策略**：调用方主动选择用哪个算法（`paymentService.pay("ALIPAY", ...)`）
- **状态**：状态自己驱动切换，调用方不知道状态的存在（`order.next()` 会自动流转）

**什么时候该你用**：有明确的状态机（订单待支付→已支付→已发货→已完成），且不同状态下的行为差异很大。**用状态模式替换一大坨 `switch(status)`**。

```java
interface OrderState { void pay(OrderCtx ctx); void ship(OrderCtx ctx); OrderStateEnum type(); }
@Component class PendingPayState implements OrderState {
    public void pay(OrderCtx ctx) { ctx.setState(PAID); }        // 状态转移内聚在状态类里
    public void ship(OrderCtx ctx) { throw new IllegalStateException("未支付不能发货"); }
}
```

---

#### 2.3.7 命令模式

**意图**：把请求封装成对象，支持排队、撤销、重做、日志。

**框架中的位置**：
- `Runnable` / `Callable` 是最朴素的命令
- **线程池的任务队列** —— 命令排队执行
- **MQ 的消息** —— 分布式版的命令（持久化 + 重投 + 延迟）
- **数据库事务日志 / undo log** —— 可撤销的命令
- `Statement` / `PreparedStatement` —— 把 SQL 封装成命令对象

**什么时候该你用**：需要 undo/redo（文本编辑器、批量操作回滚）、需要把操作排队或异步执行、需要记录操作日志做审计。

---

#### 2.3.8 解释器模式

**框架中的位置**：
- **SpEL（Spring Expression Language）**：`SpelExpressionParser` 解析 `@Value("#{...}")`、`@PreAuthorize("hasRole('ADMIN')")`
- `java.util.regex.Pattern` —— 正则表达式就是解释器的经典实现
- SQL 解析器（ShardingSphere、JSqlParser）
- MyBatis 的 OGNL 表达式（`<if test="name != null and name != ''">`）

**什么时候该你用**：需要定义一门小语言/DSL（规则引擎的条件表达式、查询过滤语法）。注意：复杂语法一定用 ANTLR 这类解析器生成器，别手写。

---

#### 2.3.9 访问者模式

**框架中的位置**：
- ASM / Javassist 的 `ClassVisitor` / `MethodVisitor`（字节码改写）
- MyBatis 的 `SqlNode.apply(DynamicContext)` —— 实际上是访问者的变体
- Hibernate 的 `EventListener`

**特点**：**双分派**，适合"数据结构稳定但操作频繁变化"的场景。使用频率不高，但读 ASM 代码时必须理解它。

---

#### 2.3.10 中介者模式

**框架中的位置**：
- **消息队列** —— 服务间不再直接调用，通过 MQ 中转，是分布式版的中介者
- `ExecutorService` —— 线程之间不直接通信，通过队列
- 注册中心（Nacos）—— 服务之间不直接连接，通过注册中心发现

**什么时候该你用**：多个对象之间形成复杂的网状依赖（M×N 条连线）时，引入中介者把网状变成星形（M+N）。

---

#### 2.3.11 备忘录模式

**意图**：在不破坏封装的前提下保存对象状态，支持回滚。

**什么时候该你用**：需要快照/回滚的场景——编辑器的撤销、游戏存档、**数据库的事务回滚（undo log 就是备忘录的工程实现）**。使用频率低。

---

## 三、高频踩坑

| 现象 | 根因 | 解法 |
|---|---|---|
| **`@Transactional` 自调用不生效** | `this` 是原始对象不是代理，绕过了 AOP | 拆到另一个 Service；或 `AopContext.currentProxy()`（要开 `exposeProxy`）；或注入自己（`@Lazy`） |
| 加了 `@Transactional` 但事务没回滚 | 抛的是受检异常且未指定 `rollbackFor`（默认只对 `RuntimeException` 回滚）；或异常被 catch 吞了 | `@Transactional(rollbackFor = Exception.class)`；catch 后重新抛出或 `TransactionAspectSupport.currentTransactionStatus().setRollbackOnly()` |
| 注入实现类报 `could not be injected because it is a JDK dynamic proxy` | 目标实现接口时默认用 JDK 动态代理，代理对象不能转型为具体实现类 | `spring.aop.proxy-target-class=true`（Boot 2.x 起已默认）；或注入接口而非实现类 |
| 单例 Bean 里持有请求级状态导致数据串号 | 单例被所有请求共享，成员变量被并发覆盖 | 不要在单例里存可变状态；用局部变量、`ThreadLocal`（用完 `remove()`）、或 `@Scope("prototype")/request` |
| DCL 单例偶发拿到半初始化对象 | 缺 `volatile`，指令重排让引用先于构造赋值 | 加 `volatile`；或改用静态内部类 / 枚举单例 |
| 手写单例被反射或反序列化破解 | `Constructor.setAccessible(true)` 能绕开私有构造；反序列化会 `new` 新实例 | 用**枚举单例**（唯一能防住）；或构造里抛异常；加 `readResolve()` |
| `getBean("xxx")` 拿到的不是想要的对象 | `"xxx"` 是 `FactoryBean`，拿到的是它 `getObject()` 的产出 | 加 `&` 前缀：`getBean("&xxx")` 拿 FactoryBean 本身 |
| 迭代器遍历时删除元素抛 `ConcurrentModificationException` | fail-fast 机制：`modCount` 与 `expectedModCount` 不一致 | `removeIf()`；或用 `Iterator.remove()`；或用 `CopyOnWriteArrayList` |
| `@EventListener` 监听器里的慢操作拖慢主流程 | 事件默认**同步**执行 | 加 `@Async`（要开 `@EnableAsync`） |
| 监听器抛异常导致主业务回滚 | 监听器与发布方默认在同一事务 | `@TransactionalEventListener(AFTER_COMMIT)`；或监听器内部 try-catch |
| 某个监听器挂了，后面的监听器都不执行了 | `ApplicationEventMulticaster` 没有异常隔离（默认 `SimpleApplicationEventMulticaster` 无 ErrorHandler） | 监听器内部必须 try-catch；或自定义 `multicaster.setErrorHandler()` |
| 用 `Cloneable` 做深拷贝，改了副本影响了原对象 | `super.clone()` 是**浅拷贝**，引用类型字段仍共享 | 手动深拷贝引用字段；或用序列化 / BeanUtils；干脆别用 `Cloneable` |
| 为了"以后可能扩展"提前抽象，代码反而难读 | 过度设计：抽象点没有第二个实现 | 等真的出现第二个实现再抽象（Rule of Three） |
| 策略类越加越多，新人不知道该用哪个 | 缺少统一的注册/发现机制 | 用 `Map<String, Strategy>` 自动注入 + 枚举定义所有类型 + 启动时校验完整性 |
| 装饰器层层嵌套，出问题时不知道是哪层 | 没有日志/追踪，嵌套层次不可见 | 每层打日志；或用 Tracer / traceId 串联 |
| Filter 里拿不到 `@Transactional` 的事务 | Servlet Filter 在 Interceptor 之外，更在 AOP 之外（洋葱模型） | 事务逻辑放到 Service 层，不要在 Filter 里做业务 |
| 责任链过长导致延迟累积 | 每个节点都有 IO 或锁 | 合并相邻职责节点；并行化无依赖节点；加超时和链路追踪 |
| 用 `@Builder` 后无参构造和 setter 都没了，Jackson 反序列化失败 | Lombok `@Builder` 生成全参构造，覆盖了默认无参构造 | 同时加 `@NoArgsConstructor` + `@AllArgsConstructor` + `@Builder` |
| 继承模板方法时改了流程导致行为异常 | 模板方法没用 `final` 修饰，被子类覆盖 | 骨架方法加 `final`，只暴露 `protected abstract` 钩子 |

## 四、往下钻的入口

**Spring 源码（按模式分组）**：

| 模式 | 源码位置 |
|---|---|
| 单例 | `org.springframework.beans.factory.support.DefaultSingletonBeanRegistry`（`singletonObjects` / `getSingleton` / 三级缓存 `singletonFactories`） |
| 工厂 | `AbstractBeanFactory.doGetBean()`、`FactoryBeanRegistrySupport.getObjectFromFactoryBean()` |
| 代理 | `DefaultAopProxyFactory.createAopProxy()`、`JdkDynamicAopProxy.invoke()`、`CglibAopProxy.getProxy()` |
| 责任链 | `ReflectiveMethodInvocation.proceed()`、`DefaultAopProxyFactory` 的拦截器链构建 |
| 模板方法 | `AbstractApplicationContext.refresh()`（**12 步，必读 javadoc**）、`JdbcTemplate.execute()` |
| 适配器 | `DispatcherServlet.doDispatch()` → `getHandlerAdapter()`、`HandlerAdapter` 的四个实现 |
| 观察者 | `AbstractApplicationContext.publishEvent()`、`SimpleApplicationEventMulticaster.multicastEvent()` |
| 策略 | `InstantiationStrategy`（`SimpleInstantiationStrategy` / `CglibSubclassingInstantiationStrategy`） |
| 建造者 | `BeanDefinitionBuilder`、`AbstractBeanDefinition` |

**MyBatis 源码**：
- 代理：`MapperProxyFactory.newInstance()`、`MapperProxy.invoke()`、`MapperMethod.execute()`
- 工厂：`SqlSessionFactoryBuilder.build()` → `DefaultSqlSessionFactory`
- 组合：`org.apache.ibatis.scripting.xmltags` 包（`MixedSqlNode` / `IfSqlNode` / `WhereSqlNode` / `ForEachSqlNode`）
- 模板方法：`BaseExecutor.query()` → `SimpleExecutor.doQuery()`
- 装饰器：`CachingExecutor` 包装 `SimpleExecutor`（二级缓存）
- 责任链：`InterceptorChain.pluginAll()`
- 建造者：`MappedStatement.Builder`、`XMLConfigBuilder`

**Dubbo 源码**：
- 策略：`LoadBalance` 的五个实现，`ExtensionLoader.getExtension(url.getParameter("loadbalance", "random"))`
- 责任链：`ProtocolFilterWrapper.buildInvokerChain()`
- 装饰器：`ProtocolListenerWrapper`、`QosProtocolWrapper`（Wrapper 自动嵌套）
- 适配器：`Invoker` → `Exporter` 的适配，`JavassistProxyFactory.getProxy()`
- 代理：`InvokerInvocationHandler`
- 模板方法：`AbstractClusterInvoker.invoke()` → `FailoverClusterInvoker.doInvoke()`

**必看书目具体章节**：
- 《设计模式：可复用面向对象软件的基础》（GoF）—— 只看意图、适用性、结构三节即可，代码示例偏 C++
- 《Head First 设计模式》—— 理解意图与结构的最佳入门
- 《Effective Java》第 3 版 —— Item 1（静态工厂）、Item 3（单例用枚举）、Item 13（谨慎 clone）、Item 21（策略接口优于函数对象）
- 《重构》第 2 版 —— 第 6-9 章，看"从 if-else 到策略"、"从 switch 到多态"的重构手法

**实践建议 —— 在代码里发现模式的信号**：

| 看到的坏味道 | 该考虑的模式 |
|---|---|
| 一长串 `if-else if` 按类型分支 | 策略 + Map 自动注册 |
| 多个类的某个方法流程高度相似，只有几步不同 | 模板方法 |
| 创建对象要传很多参数，其中多个可选 | 建造者 |
| 需要在方法前后加日志/鉴权/计时，且已有几十处 | 代理 / 装饰器 |
| 一个操作要触发好几个不相关的后续动作 | 观察者（Spring 事件） |
| 方法签名里有多个同类型参数容易传错 | 参数对象 / 建造者 |
| 一个类有两维度变化导致子类数量相乘 | 桥接 |
| 状态判断的 `switch(status)` 散布在多个方法里 | 状态模式 |
| 请求要经过一串校验，且顺序要能调 | 责任链 |

## 五、达标自检

**基础概念**
1. Spring 的单例和 GOF 的单例有什么区别？为什么 Spring 的单例在多容器下不唯一？
2. 手写 DCL 单例时为什么必须加 `volatile`？不加具体会发生什么？
3. 有哪五种单例写法？哪种能防反射攻击？为什么？
4. **`BeanFactory` 和 `FactoryBean` 有什么区别？** `getBean("x")` 和 `getBean("&x")` 分别拿到什么？
5. JDK 动态代理和 CGLIB 的实现原理分别是什么？各自的限制是什么？Spring Boot 2.x 起默认用哪个，为什么改？
6. **为什么 `@Transactional` 自调用会失效？至少说出两种解法并比较优劣。**
7. 代理模式和装饰器模式的结构几乎一样，本质区别是什么？各举一个框架中的例子。
8. 适配器与门面的区别是什么？Spring MVC 里哪个是适配器？
9. 模板方法和策略模式的区别是什么？什么情况下必须用其中一个？
10. 状态模式和策略模式的 UML 结构一样，怎么区分？

**框架源码**
11. `AbstractApplicationContext.refresh()` 是哪种模式的体现？它的哪几个方法是留给子类的钩子？
12. MyBatis 的 Mapper 接口为什么不需要实现类？请说出从调用 Mapper 方法到执行 SQL 的完整代理链路。
13. AQS 用了哪种设计模式？哪些是模板方法、哪些是钩子方法？除了 `ReentrantLock`，还有哪些 JDK 类基于它？
14. 说出 Spring AOP 拦截器链的实现（哪个类、哪个方法），它如何实现"方法前后都能织入"？
15. Dubbo 的 Wrapper 机制是哪种模式？它是怎么自动识别并层层嵌套的？
16. MyBatis 的 `SqlNode` 用了哪种模式？`<where><if>` 嵌套是怎么被拼成 SQL 的？
17. Spring MVC 里 `HandlerAdapter` 为什么存在？它有哪几个内置实现？
18. Spring 事件机制是观察者还是发布订阅？同步还是异步？
19. 享元模式在 JDK 里有哪些具体例子？String 常量池能共享的前提是什么？
20. 为什么说 JDBC 是桥接模式的经典例子？

**实战应用**
21. 你的代码里有一段按 `type` 分发的 `if-else`，未来还会加类型。请给出重构方案（具体到代码结构）。
22. 一个订单支付成功后要发券、发通知、更新统计、同步 ES，怎么设计让支付主流程不被拖累且互不影响？要考虑事务边界。
23. 要对接三家短信厂商，接口各不相同且可能更换。你怎么设计？
24. 一个接口要实现多级缓存（本地 → Redis → DB），你会怎么组织代码？
25. 有一串风控规则（黑名单、频率限制、金额校验、地域校验）要依次执行且顺序可配置，怎么设计？
26. 你接手了一个方法体超过 300 行、里面有大量 `switch(status)` 的订单类。你的重构步骤是什么？
27. 什么情况下**不应该**引入设计模式？举出你自己的判断标准。
28. 你最近写的代码里，有哪些地方实际用到了本文提到的模式？哪些地方其实该用而没用？
