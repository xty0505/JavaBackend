# Spring

## Spring 框架模块

Spring 是一个轻量级开发框架，旨在提高开发人员的开发效率和系统的可维护性。

一般说的 Spring 即 Spring Framework，它是很多模块的集合。

- 核心容器：所有其他组件的核心，实现了 IoC 依赖注入
- 数据访问/集成
- Web
- AOP
- 消息
- 测试

![image-20210228193728692](C:\Users\aasus\AppData\Roaming\Typora\typora-user-images\image-20210228193728692.png)

##  @RestController vs @Controller

@Controller 一般使用在前后端不分离的情况， 即返回一个 View， 适用于传统的 Spring MVC 应用。

![image-20210228194248390](C:\Users\aasus\AppData\Roaming\Typora\typora-user-images\image-20210228194248390.png)

@RestController 一般使用在前后端分离的情况，即返回对象， 对象数据以 JSON 或 XML 的形式写入 HttpResponseBody 中，适用于 RESful Web 服务。

![image-20210301160327631](C:\Users\aasus\AppData\Roaming\Typora\typora-user-images\image-20210301160327631.png)

## Spring IoC

IoC（Inverse of Control:控制反转）是一种**设计思想**，就是 **将原本在程序中手动创建对象的控制权，交由Spring框架来管理。** IoC 在其他语言中也有应用，并非 Spring 特有。 IoC 容器是 Spring 用来实现 IoC 的载体， IoC 容器实际上就是个**Map**，Map 中存放的是各种对象。

![image-20210228194842244](C:\Users\aasus\AppData\Roaming\Typora\typora-user-images\image-20210228194842244.png)

好处在于

- 将对象的实例化交给 IoC 容器，开发人员只需要写好配置文件/注解即可。
- 增加项目的可维护性，降低了开发难度。

## Spring AOP

AOP (Aspect-Oriented Programming:面向切面编程)能够将那些与业务无关，却为业务模块所**共同调用的逻辑或责任**（例如事务处理、日志管理、权限控制等）封装起来，便于减少系统的重复代码，降低模块间的耦合度，并有利于未来的可拓展性和可维护性。

Spring AOP 基于**动态代理**，

- 代理的对象实现了接口，JDK Proxy 创建代理对象
- 代理对象未实现接口，Cglib 生成一个被代理对象的子类作为代理

![image-20210228195222970](C:\Users\aasus\AppData\Roaming\Typora\typora-user-images\image-20210228195222970.png)

Spring AOP 与 AspectJ AOP 的区别

- Spring AOP 是运行时增加，而 AspectJ 是编译时增强
- Spring AOP 基于代理，AspectJ 基于字节码
- Spring AOP 使用简单，AspectJ 功能更强大

### JDK Proxy

要求代理类和增强类必须实现同一个接口。

使用 JDK  Proxy 实现代理过程

- 实现一个 InvocationHandler，对原类方法调用都会转发到该类的 invoke() 方法。该类持有一个原类对象，在 invoke 中实现增加逻辑并执行原类方法
- Proxy.newProxyInstance

JDK Proxy 存在的问题

- 获取的代理实例无法强制转换为原类

### Cglib

底层采用 ASM 字节码生码生成框架， 直接对需要代理的类的字节码进行操作，生成该类的一个子类，并重写了类中所有可以重写的方法。在重写时将我们定义的额外逻辑织入方法中，从而实现了增强。

使用 Cglib 代理实现过程

- 实现一个 MethodInterceptor，对原类所有**非 final 方法的调用**会被转发到该类的 intercept() 方法
- 在需要使用原类的时候，通过 Cglib 动态代理获取代理对象

## Spring Bean

### Bean 的作用域

- singleton : 唯一 bean 实例，Spring 中的 bean 默认都是单例的。
- prototype : 每次请求都会创建一个新的 bean 实例。
- request : 每一次 HTTP 请求都会产生一个新的 bean，该 bean 仅在当前 HTTP request 内有效。
- session : 每一次 HTTP 请求都会产生一个新的 bean，该 bean 仅在当前 HTTP session 内有效。
- global-session： 全局 session 作用域，仅仅在基于 portlet 的 web 应用中才有意义，Spring5 已经没有了。Portlet 是能够生成语义代码(例如：HTML )片段的小型 Java Web 插件。它们基于 portlet 容器，可以像 servlet 一样处理 HTTP 请求。但是，与 servlet 不同，每个 portlet 都有不同的会话

### 单例 Bean 的线程安全问题

单例 Bean 存在线程安全问题，当多个线程操作同一个 bean 时，对这个变量的写操作会存在线程安全问题。

但是一般情况下， 常用的 `Controller`、`Service`、`Dao` 是无状态的，不能保存数据，因此是线程安全的。对于其他单例 Bean 可以采用以下方式

- 在单例 Bean 中定义一个 `ThreadLocal` 成员变量
- 修改为 prototype 

### Bean 生命周期

- 注册 BeanDefinition：Bean 容器找到配置文件中 Spring Bean 的定义并注册。
- instantiateBean：Bean 容器利用 Java Reflection API 创建一个Bean的实例。
- populateBean：填充属性值。
- intializeBean：初始化 Bean 
  - invokeAwareMethods：如果实现了 `*.Aware`接口，就调用相应的方法。
  - applyBeanPostProcessorsBeforeInitialization：如果有和加载这个 Bean 的 Spring 容器相关的 `BeanPostProcessor` 对象，执行`postProcessBeforeInitialization()` 方法
  - 如果Bean实现了`InitializingBean`接口，执行`afterPropertiesSet()`方法。
  - 如果 Bean 在配置文件中的定义包含 init-method 属性，执行指定的方法。
  - applyBeanPostProcessorsAfterInitialization：如果有和加载这个 Bean的 Spring 容器相关的 `BeanPostProcessor` 对象，执行`postProcessAfterInitialization()` 方法
- 当要销毁 Bean 的时候，如果 Bean 实现了 `DisposableBean` 接口，执行 `destroy()` 方法。
- 当要销毁 Bean 的时候，如果 Bean 在配置文件中的定义包含 destroy-method 属性，执行指定的方法。

![image-20210301154737137](C:\Users\aasus\AppData\Roaming\Typora\typora-user-images\image-20210301154737137.png)

## Spring MVC

![image-20210301155247816](C:\Users\aasus\AppData\Roaming\Typora\typora-user-images\image-20210301155247816.png)

工作流程

- 客户端发送请求，请求到达 DispatcherServlet
- DispatcherServlet 根据请求去 HandlerMapping 查询 Handler
- 解析到对应的 Handler (Controller) 后，通过 HandlerAdapter 适配并执行
- Handler 执行完成后返回 ModelAndView 对象
- ViewResolver 根据逻辑 View 查询实际 View
- DispatcherServlet 把 Model 数据填充到 View（或直接填充到 response 域中）
- 返回 View

## Spring 框架中的设计模式

1.工厂模式

Spring 通过 BeanFactory 或 ApplicationContext 创建 Bean 对象

2.单例模式

Spring 中 Bean 默认都是单例的，好处在

- 对于频繁使用的对象，节省每次创建对象花费的时间
- new 操作次数减少，减轻内存的使用频率，缩短 GC 停顿时间

3.代理模式

Spring AOP 就是通过代理来实现的

4.模板方法

Spring 中 jdbcTemplate、hibernateTemplate 等以 Template 结尾的类

5.适配器模式

Spring AOP 的增强和通知（Advice）使用了适配器模式，Spring MVC 中 HandlerAdapter 也用到了适配器模式

## Spring 事务

### 事务传播行为

支持当前事务的情况：

- **TransactionDefinition.PROPAGATION_REQUIRED：** 如果当前存在事务，则加入该事务；如果当前没有事务，则创建一个新的事务。
- **TransactionDefinition.PROPAGATION_SUPPORTS：** 如果当前存在事务，则加入该事务；如果当前没有事务，则以非事务的方式继续运行。
- **TransactionDefinition.PROPAGATION_MANDATORY：** 如果当前存在事务，则加入该事务；如果当前没有事务，则抛出异常。（mandatory：强制性）

不支持当前事务：

- **TransactionDefinition.PROPAGATION_REQUIRES_NEW：** 创建一个新的事务，如果当前存在事务，则把当前事务挂起。
- **TransactionDefinition.PROPAGATION_NOT_SUPPORTED：** 以非事务方式运行，如果当前存在事务，则把当前事务挂起。
- **TransactionDefinition.PROPAGATION_NEVER：** 以非事务方式运行，如果当前存在事务，则抛出异常。

其他情况：

- **TransactionDefinition.PROPAGATION_NESTED：** 如果当前存在事务，则创建一个事务作为当前事务的嵌套事务来运行；如果当前没有事务，则该取值等价于TransactionDefinition.PROPAGATION_REQUIRED。

### @Transaction

#### 步骤

通过 @Transaction 注解实现事务的步骤

- 在 xml 配置事务信息

```xml
<tx:annotation-driven />
<bean id="transactionManager"
class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
<property name="dataSource" ref="dataSource" />
</bean>
```

- 将 @Transaction 添加到方法或者类上。添加到类上表示该类的所有 public 方法都配置相同的事务属性信息。



#### 机制

在应用系统调用声明 @Transactional 的目标方法时，Spring Framework 默认使用 AOP 代理。

在代码运行时生成一个代理对象，根据 @Transactional 的属性配置信息，这个代理对象决定该声明 @Transactional 的目标方法是否由拦截器 TransactionInterceptor 来使用拦截。

在 TransactionInterceptor 拦截时，会在在目标方法开始执行之前创建并加入事务，并执行目标方法的逻辑, 最后根据执行情况是否出现异常，利用抽象事务管理器 AbstractPlatformTransactionManager 操作数据源 DataSource 提交或回滚事务。

抽象事务管理器根据底层的数据库事务实现，例如 DataSourceTracManager 管理 JDBC 的连接。



