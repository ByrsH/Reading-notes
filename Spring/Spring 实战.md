#### 1.1.2  依赖注入

1.构造器注入：在类的构造器中不自行创建新的对象（任务），而是把对象作为参数传入。这个参数类型可以是一个接口，这样只要某个类实现了该接口，则其类的实例对象可以作为参数传入构造器。这就实现了松耦合，该类不需要知道具体是哪个实现类。

#### 1.1.3 应用切面。在XML中配置bean，然后再配置aop。

#### 1.1.4 使用模版消除样板式代码

### 1.2 容纳你的Bean

1. Spring自带多个容器实现，可以归为两种不同的类型：bean工厂（beanFactory，提供基本的DI支持）、应用上下文（ApplicationContext）基于BeanFactory构建，提供应用框架级别的服务，例如从属性文件解析文本信息以及发布应用事件给感兴趣的事件监听者。

#### 1.2.1 使用应用上下文
 
#### 1.2.2 bean的生命周期


### 1.3 俯瞰Spring风景线
 
## 2、装配bean
           
创建应用对象之间协作关系的行为通常称为装配（wiring），这也是依赖注入（DI）的本质。

### 2.1  主要的装配机制：


1. 在XML中进行显示配置。
2. 在java中进行显示配置。
3. 隐式的bean发现机制和自动装配。
       
### 2.2 自动化装配bean
     
Spring 从两个角度来实现自动化装配：

- 组件扫描（ component scanning ）： Spring 会自动发现应用上下文中所创建的 bean 。
- 自动装配（ autowiring ）： Spring 自动满足 bean 之间的依赖。

#### 2.2.1 

### 2.3 通过java代码装配bean

显示装配的方式：java和XML

#### 2.3.1 创建配置类

添加 @Configuration 注解。

#### 2.3.2 声明简单的bean

@Bean注解

#### 2.3.3 借助JavaConfig实现注入

默认情况下，Spring中的bean都是单例的。
                
#### 2.4 通过XML装配bean

#### 2.4.2 声明一个简单的<bean>

#### 2.4.3 借助构造器注入初始化bean
     
### 2.5 导入和混合配置

#### 2.5.1 在JavaConfig中引用XML配置
               
#### 2.5.2 在XML配置中引用JavaConfig


## 3 高级装配

### 3.1 环境与profile

#### 3.1.1 配置profile bean

@Profile注解。（@Profile("dev") 、@Profile("prod")   只有在dev profile 、prod profile分别激活时，才会创建改配置类中的bean）
                    
#### 3.1.2 激活profile

#### 3.2  条件化的bean
 
@Conditional注解
                   
### 3.3  处理自动装配的歧义性

- 标示首选bean：   @Primary 能够与 @Component 组合用在组件扫描的 bean 上，也可以与 @Bean 组合用在 Java 配置的 bean 声明中。使用 XML 配置 bean 的话，<bean> 元素有一个 primary 属性用来指定首选的 bean 

- 限定自动装配的bean:    @Qualifier 注解是使用限定符的主要方式。它可以与 @Autowired 和 @Inject 协同使用。也可以自定义限定符。


### 3.4  bean的作用域

单例（Singleton）：在整个应用中，只创建bean的一个实例。

原型（Prototype）：每次注入或者通过Spring应用上下文获取的时候，都会创建一个新的bean实例。

会话（Session）：在Web应用中，为每个会话创建一个bean实例。

请求（Rquest）: 在Web应用中，为每个请求创建一个bean实例。

使用@Scope注解设置bean的作用域。

![](https://img-blog.csdnimg.cn/2019022814323811.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)

### 3.5 运行时值注入
                    
属性占位符（Property placeholder）：注入外部的值， 使用@PropertySource注解和Environment。    “${...}”占位符， @Value注解

Spring表达式语言（SpEL）：

- 使用 bean 的 ID 来引用 bean ；
- 调用方法和访问对象的属性；
- 对值进行算术、关系和逻辑运算；
- 正则表达式匹配；
- 集合操作。
 
## 4 面向切面的Spring

### 4.1 什么是面向切面编程

散布于应用中多出的功能被称为横切关注点（cross-cutting concern），面向切面编程（AOP）所要解决的问题是把这些横切关注点与业务逻辑相分离。切面能够帮助我们模块化横切关注点。横切关注点可以被模块化为特殊的类，这些类被称为切面（aspect）。

#### 4.1.1 定义AOP术语

通知（Advice）：
   切面的工作被称为通知。通知定义了切面是什么以及何时使用。
            
连接点（Join point）：
   应用可能也有数以千计的时机应用通知。这些时机被称为连接点。连接点是在应用执行过程中能够插入切面的一个点。

切点（Poincut）：
   切点就定义了 “ 何处 ” 。切点的定义会匹配通知所要织入的一个或多个连接点。

切面（Aspect）：
   切面是通知和切点的结合。通知和切点共同定义了切面的全部内容 —— 它是什么，在何时和何处完成其功能。

引入（Introduction）：
   引入允许我们向现有的类添加新方法或属性。

织入（Weaving）：
   织入是把切面应用到目标对象并创建新的代理对象的过程。


#### 4.1.2 Spring对AOP的支持

Spring提供了4中类型的AOP支持：

- 基于代理的经典Spring AOP；
- 纯POJO切面；
- @AspectJ注解驱动的切面；
- 注入式AspectJ切面（适用于Spring各版本）。

![](https://img-blog.csdnimg.cn/20190228144413794.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)

Spring 只支持方法级别的连接点。               


### 4.2 通过切点来选择连接点

#### 4.2.1 编写切点

#### 4.2.2 在切点中选择bean

bean()指示器

### 4.3 使用注解创建切面
     
#### 4.3.1 定义切面
               
@AspectJ注解
          
![](https://img-blog.csdnimg.cn/20190228144709390.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)

@Pointcut注解能够在一个@AspectJ切面内定义可重用的切点。

@EnableAspectJAutoProxy注解启用自动代理功能。


#### 4.3.2 创建环绕通知
     
@Around注解创建环绕通知

#### 4.3.3 处理通知中的参数

#### 4.3.4 通过注解引入新功能

@DeclareParents注解将新接口引入到bean中。
               
### 4.5 注入AspectJ切面


# 第2部分  Web中的Spring


## 第5章 构建Spring Web应用程序

### 5.1 Spring ＭＶＣ　起步

#### 5.1.1 跟踪Spring MVC的请求

请求的第一站是前端控制器 DispatcherServlet。DispatcherServlet 的任务是将请求发送给 Spring MVC 控制器（ controller ）。

#### 5.1.2 搭建Spring MVC

配置DispatcherServlet
               
继承扩展AbstractAnnotationConfigDispatcherServletInitializer 类。


### 5.2 编写基本的控制器

在Spring MVC中，控制器只是方法上添加了@ReruestMapping注解的类，注解声明了它们所要处理的请求。

#### 5.2.1 测试控制器

使用mock Spring MVC 对controller进行测试。

#### 5.2.2 定义类级别的请求处理

将@RequestMapping写在类级别上， @RequestMapping可以映射多个路径：  @RequestMapping({"/", "/homepage"})

#### 5.2.3 传递模型数据到视图中

### 5.3 接受请求的输入

Spring MVC允许以多种方式将客户端中的数据传送到控制器的处理方法中：

- 查询参数（Query parameter）
- 表单参数（Form parameter）
- 路径变量（Path Variable）

#### 5.3.1 处理参数查询

#### 5.3.2 通过路径参数接受输入

面向资源请求的处理。

在@RequestMapping路径中添加占位符（{ spittleId}）， 参数与它对应： @PathVariable{"spittleId}
               

### 5.4 处理表单

表单中输入框的name与方法参数类型的成员变量名相同。
          
#### 5.4.2 表单验证
![](https://img-blog.csdnimg.cn/20190228145344687.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)  

@Valid注解，告知Spring，需要确保这个对象满足校验限制。Errors参数中对象中是校验的错误信息。



## 第六章 渲染web视图

### 6.1 理解视图解析


## 第七章 Spring MVC的高级技术


### 7.2 处理multipart形式的数据

1. 使用@Bean注解配置解析器，MultipartResolver
2. 页面表单设置form属性： enctype="multipart"， input属性
3. 编写controller中对应的方法：
	1. 添加byte[] 数组参数，并为其添加@RequestPart注解
	2. 使用MultipartFile作为参数，需要配置MultipartResolver
	3. 使用Part作为参数，不用配置MultipartResolver。

### 7.3 异常处理     

1. 使用@ResponseStatus注解，将异常映射到特定的状态码
2. @ExceptionHandler注册方法，用于某个控制器的异常映射处理。


### 7.4 为控制器添加通知

1、 @ControllerAdvice注解一个类，里面包含@ExceptionHandler注解的方法，那么所有的控制器抛出该异常是都会触发该方法。

     
### 7.5 跨重定向请求传递数据


- 使用 URL 模板以路径变量和 / 或查询参数的形式传递数据
	- URL+String
	- 添加到Model中


- 通过 flash 属性发送数据
	- Model通过addFlashAttribute方法，把复杂的数据类型添加的Flash属性中，重定向前Flash属性会复制到会话中。重定向后，再从会话中读取Flash属性。


## 8 Spring Web Flow

跳过


## 9 保护Web应用

Spring Security是一种基于Spring AOP和Servlet规范中的Filter实现的安全框架。


### 9.1 Spring Security简介

Spring Security是为基于Spring的应用程序提供声明式安全保护的安全性框架。它能够在Web请求级别和方法调用级别处理身份认证和授权。它充分利用了依赖注入（dependency injection， DI）和面向切面的技术。

![](https://img-blog.csdnimg.cn/201902281559553.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)


#### 9.1.2 过滤Web请求

Spring Security借助一系列Servlet Filter来提供各种安全性功能。
       
#### 9.2.2 基于数据库表进行认证

#### 9.2.3 基于LDAP进行认证

### 9.3 拦截请求

- 扩展WebSecurityConfigurerAdapter类，覆盖configure(HttpSecurity http) 方法。

![](https://img-blog.csdnimg.cn/20190228161225350.png)


- .antMatchers("/spitters/**").authenticated();    设定路径支持Ant风格的通配符
- .regexMatchers("/spitters/.*").authenticated();   支持正则表达式方式
- .authenticated()  要有在执行该请求时，必须已经登录了应用
- .permitAll()  方法允许请求没有任何的安全限制

![](https://img-blog.csdnimg.cn/20190228161416910.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)

- 具备 ROLE_SPITTER 权限 

![](https://img-blog.csdnimg.cn/20190228161511858.png)

或：

![](https://img-blog.csdnimg.cn/20190228161554334.png)

hasRole()方法中的角色，登录成功时，创建一个Authentication，并添加到SecurityContextHolder上下文环境中：

    public Authentication createAuthentication(User user) {
    return new UsernamePasswordAuthenticationToken(user.getMobile(),
    user.getPassword(),
    Arrays.asList(new SimpleGrantedAuthority("ROLE_USER"))); //TODO: user.getRoleName())));
    }
    
    
    Authentication authentication = authenticationService.auth(mobile, password);
    if (authentication == null
    || authentication instanceof AnonymousAuthenticationToken
    || authentication instanceof UsernamePasswordAuthenticationToken) {
    
    SecurityContextHolder.getContext().setAuthentication(authentication);
    Optional<User> user = userService.getUserByMobile(mobile);
    MyWebUtils.saveInfoAfterAuthed(request, response, user.get().getId(), mobile);
    //发放答题有礼活动的购车券
    couponService.grantAnswerCoupon(user.get().getId(), WebUtils.getSessionId(request));
    }


#### 9.3.1 使用Spring表达式进行保护

- 可以使用hasRole("") （Spring Security 会为角色字符串加上前缀：ROLE_）限制某个特定角色，但不能在相同的路径上同时使用hasIpAddress()限制特定的IP地址。
- 使用Spring表达式（SpEL）  来限定角色：   .antMatchers("/spitter/me").access("hasRole('ROLE_SPITTER')")
- 可以通过 .antMatchers("/spitter/me").access("hasRole('ROLE_SPITTER') and hasIpAddress('192.168.1.2')")  同时限定角色和IP地址。 

![](https://img-blog.csdnimg.cn/20190228161813531.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)  

#### 9.3.2 强制通道的安全性

requiresChannel()方法为选定的URL强制使用HTTPS
![](https://img-blog.csdnimg.cn/20190228161900583.png)

不论何时，只要是对“/spitter/form”的请求，Spring Security都视为需要安全通道（通过调用requiresChannel()的确定）并自动将请求重定向到HTTPS上。
     
使用requiresInsecure()代替requiresSecure()方法，会重定向到不安全的HTTP通道上。      

    .antMatchers("/").requiresInecure();


#### 9.3.3 防止跨站请求伪造

跨站请求伪造（cross-site request forgery, CSRF）：如果一个站点欺骗用户提交请求到其他服务器的话，就会发生CSRF攻击。

Spring Security通过一个同步token的方式来实现CSRF防护的功能。

可以在配置中通过调用 csrf().disable() 来禁用Spring Security的CSRF防护功能，如

![](https://img-blog.csdnimg.cn/20190228162030206.png)


### 9.4 认证用户

#### 9.4.2 启用HTTP Basic认证

在configure() 方法所传入的HttpSecurity对象上调用httpBasic() 即可。还可以通过 realmName() 方法指定域。当在web浏览器中使用时，将会向用户弹出一个简单的模态对话框。本质是HTTP 401响应。

![](https://img-blog.csdnimg.cn/20190228162143530.png)


#### 9.4.3 启用Remember-me功能     

在HttpSecurity对象上调用rememberMe() 即可。
    
![](https://img-blog.csdnimg.cn/20190228162224704.png)

默认情况下，这个功能是通过在cookie中存储一个token完成的，这个token最多两周有效。也可以设定有效时间， .key("")为私钥，默认为SpringSecured，也可以设置为其他的。存储在 cookie 中的 token 包含用户名、密码、过期时间和一个私钥 —— 在写入 cookie 前都进行了 MD5 哈希。

     
#### 9.4.4 退出

默认情况下，退出功能是通过Servlet容器中的Filter实现的，这个Filter会拦截 "/logout" 的请求。用户退出应用，会清除Remember-me token。退出完成后，会重定向到 "/login?logout" 重新登录。

![](https://img-blog.csdnimg.cn/2019022816232211.png)


### 9.5 保护视图

#### 9.5.1 使用Spring Security的JSP标签库

![](https://img-blog.csdnimg.cn/2019022816240317.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)
  
#### 9.5.2 　使用 Thymeleaf 的 Spring Security 方言   

![](https://img-blog.csdnimg.cn/20190228162440396.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)


# 第三部分 后端中的Spring


## 第10章 通过Spring和JDBC征服数据库

数据访问对象（data access object, DAO）

面向接口编程，使数据访问层与应用程序的其他部分隔离开来。

SQLException是JDBC处理数据访问所有问题的通用异常。Spring提供了多个数据访问异常，分别描述了它们抛出时所对应的问题。Spring的异常体系比JDBC简单的SQLException丰富的多，但它并没有与特定的持久化方式相关联。
   
![](https://img-blog.csdnimg.cn/2019022816261473.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)

#### 10.1.2 数据访问模板化     

模板方法将过程中与特定实现相关的部分委托给接口，而这个接口的不同实现定义了过程中的具体行为。

Spring 将数据访问过程中固定的和可变的部分明确划分为两个不同的类：模板（ template ）和回调（ callback ）。模板管理过程中固定的部分， 而回调处理自定义的数据访问代码。
          
![](https://img-blog.csdnimg.cn/20190228162704364.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)

![](https://img-blog.csdnimg.cn/2019022816273683.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)
        

### 10.2 配置数据源

在Spring上下文中配置数据源bean的多种方式：

- 通过JDBC驱动程序定义的数据源
- 通过JNDI (Java Naming and Directory Interface,Java命名和目录接口) 查找的数据源
- 连接池的数据源   


#### 10.2.1 使用JNDI数据源

XML配置：
![](https://img-blog.csdnimg.cn/20190228162900585.png) 

java配置：
![](https://img-blog.csdnimg.cn/20190228162949811.png) 


#### 10.2.2 使用数据源连接池

使用开源的实现：

- Apache Commons DBCP (http://jakarta.apache.org/commons/dbcp) ；
- c3p0 (http://sourceforge.net/projects/c3p0/)  ；
- BoneCP (http://jolbox.com/)。

配置DBCP basicDataSource:
     
![](https://img-blog.csdnimg.cn/201902281651330.png)
    
![](https://img-blog.csdnimg.cn/20190228165211717.png)


#### 10.2.3 基于JDBC驱动的数据源

Spring 提供了三个这样的数据源类：

- DriverManagerDataSource ：在每个连接请求时都会返回一个新建的连接。与 DBCP 的 BasicDataSource 不同， 由 DriverManagerDataSource 提供的连接并没有进行池化管理；
- SimpleDriverDataSource ：与 DriverManagerDataSource 的工作方式类似，但是它直接使用 JDBC 驱动，来解决在特定环境下 的类加载问题，这样的环境包括 OSGi 容器；
- SingleConnectionDataSource ：在每个连接请求时都会返回同一个的连接。尽管 SingleConnectionDataSource 不是严格意 义上的连接池数据源，但是你可以将其视为只有一个连接的池。

java配置DriverManagerDataSource:
![](https://img-blog.csdnimg.cn/2019022816531352.png)

XML配置方法：
![](https://img-blog.csdnimg.cn/20190228165348637.png)


#### 10.2.4 使用嵌入式的数据源

#### 10.2.5 使用profile选择数据源


### 10.3 在Spring中使用JDBC

10.3.2 使用JDBC模版

Spring 的 JDBC 框架承担了资源管理和异常处理的工作，从而简化了 JDBC 代码，让我们只需编写从数据库读写数据的必需代码。

Spring 为 JDBC 提供了三个模板类供选择：

- JdbcTemplate ：最基本的 Spring JDBC 模板，这个模板支持简单的 JDBC 数据库访问功能以及基于索引参数的查询；
- NamedParameterJdbcTemplate ：使用该模板类执行查询时可以将值以命名参数的形式绑定到 SQL 中，而不是使用简单的索引参 数；
- SimpleJdbcTemplate ：该模板类利用 Java 5 的一些特性如自动装箱、泛型以及可变参数列表来简化 JDBC 模板的使用。

使用 JdbcTemplate 来插入数据：
     
![](https://img-blog.csdnimg.cn/20190228165516737.png)


## 第11章 使用对象-关系映射持久化数据

**延迟加载**（ Lazy loading ）：随着我们的对象关系变得越来越复杂，有时候我们并不希望立即获取完整的对象间关系。举一个典型的例 子，假设我们在查询一组 PurchaseOrder 对象，而每个对象中都包含一个 LineItem 对象集合。如果我们只关心 PurchaseOrder 的 属性，那查询出 LineItem 的数据就毫无意义。而且这可能是开销很大的操作。延迟加载允许我们只在需要的时候获取数据。

**预先抓取**（ Eager fetching ）：这与延迟加载是相对的。借助于预先抓取，我们可以使用一个查询获取完整的关联对象。如果我们需 要 PurchaseOrder 及其关联的 LineItem 对象，预先抓取的功能可以在一个操作中将它们全部从数据库中取出来，节省了多次查询的 成本。

**级联**（ Cascading ）：有时，更改数据库中的表会同时修改其他表。回到我们订购单的例子中，当删除 Order 对象时，我们希望同时在 数据库中删除关联的 LineItem 。

Spring 对ORM的支持：

- 支持集成 Spring 声明式事务；
- 透明的异常处理；
- 线程安全的、轻量级的模板类；
- DAO 支持类；
- 资源管理。


### 11.1 在Spring中集成Hibernate

Hibernate提供了缓存、延迟加载、预先抓取和分布式缓存等复杂功能

#### 11.1.1 声明Hibernate的Session工厂



## 第12章 使用NoSQL数据库

### 12.1 使用MongoDB持久化文档数据

有一些数据的最佳表现形式是文档（document）。也就是说，不要把这些数据分散到多个表、节点或实体中，将这些信息收集到一个非规范化 （也就是文档）的结构中会更有意义。尽管两个或两个以上的文档有可能会彼此产生关联，但是通常来讲，文档是独立的实体。能够按照这种 方式优化并处理文档的数据库，我们称之为文档数据库。

当要存储的数据它们之间有丰富的关联关系，那么使用文档数据库存储是不合适的。

Spring Data MongoDB提供了三种方式在Spring应用中使用MongoDB： 

- 通过注解实现对象-文档映射
- 使用MongoTemplate实现基于模板的数据库访问
- 自动化的运行时Repository生成功能     
      

#### 12.1.1 启用MongoDB

在Spring中配置必要的bean，配置MongoClient，以便于访问MongoDB数据库。配置mongoTemplate bean，实现基于模板的数据库访问。启用Spring Data MongoDB的自动化Repository生成功能。

还可以让配置类扩展AbstractMongo-Configuration并重载getDatabaseName()和mongo()方法。

     
#### 12.1.2 为模型添加注解，实现MongoDB持久化

![](https://img-blog.csdnimg.cn/20190228165908798.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)


#### 12.1.3 使用MongoTemplate访问MongoDB

使用注解使用MongoTemplate，MongoOperations是MongoTemplate实现的一个接口：

    
    @Autowired
    MongoOperations mongo


#### 12.1.4 编写MongoDB Repository     

通过@EnableMongoRepositories注解启用Spring Data MongoDB的Repository功能，然后创建一个接口，使其扩展MongoRepository。

MongoRepository 接口有两个参数，第一个是带有 @Document 注解的对象类型，也就是该 Repository 要处理的类型。第二个参数是带 有 @Id 注解的属性类型。

![](https://img-blog.csdnimg.cn/20190228170023138.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)  

     
添加自定义的查询方法

****ByCustomer(String c)   By之前随意字母，之后是文档的某个属性，参数是表示查询与该属性相等值的文档。

指定查询

@Query注解，接受一个JSON格式查询。
![](https://img-blog.csdnimg.cn/20190228170146827.png)

混合自定义的功能


### 12.2 使用Neo4j操作图数据库

文档型数据库是将数据存储到粗粒度的文档中，而图数据库会将数据存储到多个细粒度的节点中，这些节点之间通过关系建立连接。图数据库 中的一个节点通常会对应数据库中的一个概念（ concept ），它会具备描述节点状态的属性。连接两个节点的关联关系可能也会带有属性。

Spring Data Neo4j 提供了将 Java 对象 映射到节点和关联关系的注解、面向模板的 Neo4j 访问方式以及 Repository 实现的自动化生成功能。


#### 12.2.1 配置Spring Data Neo4j

声明GraphDatabaseService bean和启用Neo4j Repository自动生成功能。

#### 12.2.2 使用注解标注图实体

Neo4j 定义了两种类型的实体：节点（ node ）和关联关系（ relationship ）。一般来讲，节点反映了应用中的事物，而关联关系定义了这些事物 是如何联系在一起的。

![](https://img-blog.csdnimg.cn/20190228170430398.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)

不管是节点实体还是关联关系实体，都必须要有一个图 ID ，而且其类型必须为 Long 。


#### 12.2.3 使用Neo4jTemplate

使用 @Autowired   private Neo4jOperations neo4j  注入你任意想使用的地方。

#### 12.2.4 创建自动化的Neo4j Repository
![](https://img-blog.csdnimg.cn/20190228170508250.png)
![](https://img-blog.csdnimg.cn/20190228170555235.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)
![](https://img-blog.csdnimg.cn/20190228170631512.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)

自定义查询


### 12.3 使用Redis操作key-value数据

#### 12.3.1 连接到Redis    

Spring Data Redis 为四种 Redis 客户端实现提供了连接工厂：

- JedisConnectionFactory
- JredisConnectionFactory
- LettuceConnectionFactory
- SrpConnectionFactory

![](https://img-blog.csdnimg.cn/20190228170736670.png)

     
#### 12.3.2 使用Redis Template

Spring Data Redis 提供了两个模 板：

- RedisTemplate
- StringRedisTemplate

![](https://img-blog.csdnimg.cn/20190228170837846.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)

     
     
## 第13章 缓存数据

缓存（Caching）可以存储经常会用到的信息，这样每次需要的时候，这些信息都是立即可用的。

### 13.1 启用对缓存的支持

Spring对缓存的支持有两种方式：

- 注解驱动的缓存
- XML声明的缓存

启用Spring对注解驱动缓存的支持：  @EnableCaching
![](https://img-blog.csdnimg.cn/20190228171039771.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)

XML方式配置：

![](https://img-blog.csdnimg.cn/20190228171110150.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)   

缓存管理器（cache manager）是spring缓存抽象的核心，它能够与多个流行的缓存实现进行集成。

     
#### 13.1.1 配置缓存管理器

Spring提供的缓存管理器实现：

- SimpleCacheManager
- NoOpCacheManager
- ConcurrentMapCacheManager
- CompositeCacheManager
- EhCacheCacheManager

- RedisCacheManager （来自于 Spring Data Redis 项目）
- GemfireCacheManager （来自于 Spring Data GemFire 项目）


使用Ehcache缓存

Ehcache的缓存管理器是EhCacheCacheManager。

![](https://img-blog.csdnimg.cn/20190228171248910.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)
     
EhCache配置声明一个名为spittleCache的缓存，最大的堆存储为50MB，存活时间为100秒。 参考 EhCache 的文档以了解调优 EhCache 配置的细节，地址是 http://ehcache.org/documentation/configuration 。
     
![](https://img-blog.csdnimg.cn/20190228171331506.png)


使用Redis缓存     

Redis 可以用来为 Spring 缓存抽象机制存储缓存条目， Spring Data Redis 提供了 RedisCacheManager ，这是 CacheManager 的一个实现。 RedisCacheManager 会与一个 Redis 服务器协作，并通过 RedisTemplate 将缓存条目存储到 Redis 中。

![](https://img-blog.csdnimg.cn/20190228171402338.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)
     

使用多个缓存管理器

多个缓存管理器：  CompositeCacheManager

![](https://img-blog.csdnimg.cn/20190228171444340.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)
     
当查找缓存条目时， CompositeCacheManager 首先会从 JCacheCacheManager 开始检查 JCache 实现，然后通 过 EhCacheCacheManager 检查 Ehcache ，最后会使用 RedisCacheManager 来检查 Redis ，完成缓存条目的查找。


### 13.2 为方法添加注解以支持缓存     

表 13.1 中的所有注解都能运用在方法或类上。当将其放在单个方法上时，注解所描述的缓存行为只会运用到这个方法上。如果注解放在类级别 的话，那么缓存行为就会应用到这个类的所有方法上。

![](https://img-blog.csdnimg.cn/2019022817155870.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)
     
#### 13.2.1 填充缓存
  
![](https://img-blog.csdnimg.cn/20190228171626316.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)  
     
![](https://img-blog.csdnimg.cn/20190228171708264.png)
![](https://img-blog.csdnimg.cn/20190228171741838.png)

缓存的key默认是方法参数id（我们可以自定义key），value是查询的结果。在调用方法findOne()之前会到缓存中查询是否有存储，如果有该key，那么返回对应的value，不再调用findOne()方法。如果没有则调用方法，并将返回值存储到缓存中。

当为接口方法添加注解后， @Cacheable 注解会被 SpittleRepository 的所有实现继承，这些实现类都会应用相同的缓存规则。

**将值放到缓存之中**

带有 @CachePut 注解的方法始终都会被调用，而且它的返回值也会放到缓存中。这提供一种很便利的机制，能够让我们在请求 之前预先加载缓存。
     
     
**自定义缓存key**

@Cacheable 和 @CachePut 都有一个名为 key 属性，这个属性能够替换默认的 key ，它是通过一个 SpEL 表达式计算得到的。

![](https://img-blog.csdnimg.cn/20190228171838497.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)

按照这种方式配置 @CachePut ，缓存不会去干涉 save() 方法的执行，但是返回的 Spittle 将会保存在缓存中，并且缓存的 key 与 Spittle 的 id 属性相同。
     
![](https://img-blog.csdnimg.cn/20190228171909468.png)


**多条件化缓存**     

@Cacheable 和 @CachePut 提供了两个属性用以实现条件化缓存： unless 和 condition ，这两个属性都接受一个 SpEL 表达式。如 果 unless 属性的 SpEL 表达式计算结果为 true ，那么缓存方法返回的数据就不会放到缓存中。与之类似，如果 condition 属性的 SpEL 表达 式计算结果为 false ，那么对于这个方法缓存就会被禁用掉。

表面上来看， unless 和 condition 属性做的是相同的事情。但是，这里有一点细微的差别。 unless 属性只能阻止将对象放进缓存，但是在 这个方法调用的时候，依然会去缓存中进行查找，如果找到了匹配的值，就会返回找到的值。与之不同，如果 condition 的表达式计算结果 为 false ，那么在这个方法调用的过程中，缓存是被禁用的。就是说，不会去缓存进行查找，同时返回值也不会放进缓存中。

![](https://img-blog.csdnimg.cn/2019022817195841.png)
     
为 unless 设置的 SpEL 表达式会检查返回的 Spittle 对象（在表达式中通过 #result 来识别）的 message 属性。如果它包含 “NoCache” 文本 内容，那么这个表达式的计算值为 true ，这个 Spittle 对象不会放进缓存中。否则的话，表达式的计算结果为 false ，无法满足 unless 的 条件，这个 Spittle 对象会被缓存。

![](https://img-blog.csdnimg.cn/20190228172107903.png)
     
如果 findOne() 调用时，参数值小于 10 ，那么将不会在缓存中进行查找，返回的 Spittle 也不会放进缓存中，就像这个方法没有添 加 @Cacheable 注解一样。

如样例所示， unless 属性的表达式能够通过 #result 引用返回值。这是很有用的，这么做之所以可行是因为 unless 属性只有在缓存方法有 返回值时才开始发挥作用。而 condition 肩负着在方法上禁用缓存的任务，因此它不能等到方法返回时再确定是否该关闭缓存。这意味着它 的表达式必须要在进入方法时进行计算，所以我们不能通过 #result 引用返回值。


#### 13.2.2 移除缓存条目

如果带有 @CacheEvict 注解的方法被调用的话，那么会有一个或更多的条目会在缓存 中移除。当缓存中的值过时或不合法时，应该使用@CacheEvict把它删除。

![](https://img-blog.csdnimg.cn/20190228172151239.png)
     
注意：与 @Cacheable 和 @CachePut 不同， @CacheEvict 能够应用在返回值为 void 的方法上，而 @Cacheable 和 @CachePut 需要非 void 的返回值，它将会作为放在缓存中的条目。因 为 @CacheEvict 只是将条目从缓存中移除，因此它可以放在任意的方法上，甚至 void 方法。
     
![](https://img-blog.csdnimg.cn/20190228172239666.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)
     
     

### 13.3 使用XML声明缓存

使用XML声明缓存的原因：

- 不希望在源码中添加Spring的注解
- 需要在没有源码的bean上应用缓存功能

![](https://img-blog.csdnimg.cn/20190228172334912.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)

    
     
## 第14章 保护方法应用

### 14.1 使用注解保护方法

Spring Security提供了三种不同的安全注解：

- Spring Security自带的@Secured注解
- JSR-250的@RolesAllowed注解
- 表达式驱动的注解，包括 @PreAuthorize 、 @PostAuthorize 、 @PreFilter 和 @PostFilter 。


#### 14.1.1 使用@Secured注解限制方法调用     

在配置类上使用@EnableGlobalMethodSecurity，并且扩展类GlobalMethodSecurityConfiguration。

![](https://img-blog.csdnimg.cn/2019022817245166.png)

#### 14.1.2 在Spring Security中使用JSR-250的@RolesAllowed注解     

@RolesAllowed与@Secured的区别在于， @RolesAllowed是JSR-250定义的Java标准注解，@Secured是Spring特定的注解。

![](https://img-blog.csdnimg.cn/20190228172541450.png)
     

### 14.2 使用表达式实现方法级别的安全性     

![](https://img-blog.csdnimg.cn/20190228172622276.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70)

#### 14.2.1 表述方法访问规则     

@PreAuthorize 和 @PostAuthorize ，它们能够基于表达式的计算结果来限制方法的访问。

@PreAuthorize 的表达式会在方法调用之前执行，如果表 达式的计算结果不为 true 的话，将会阻止方法执行。与之相反， @PostAuthorize 的表达式直到方法返回才会执行，然后决定是否抛出安全 性的异常。

**在方法调用前验证权限**：     

![](https://img-blog.csdnimg.cn/20190228172715154.png)
     

表达式中的 #spittle 部分直接引用了方法中的同名参数。

**在方法调用之后验证权限    ** 

![](https://img-blog.csdnimg.cn/20190228172758499.png)
     

#### 14.2.2  过滤方法的输入和输出

**事后对方法的返回值进行过滤 **    

@PostFilter 也使用一个 SpEL 作为值参数。但是，这个表达式不是用来限制方法访问 的， @PostFilter 会使用这个表达式计算该方法所返回集合的每个成员，将计算结果为 false 的成员移除掉。

![](https://img-blog.csdnimg.cn/20190228172842703.png)
     
**事先对方法的参数进行过滤**
     
![](https://img-blog.csdnimg.cn/20190228172931630.png)

定义许可计算器  



# 第4部分 Spring集成


## 第15章 使用远程服务

远程调用技术:

1. 远程方法调用（ Remote Method Invocation ， RMI ）；
2. Caucho 的 Hessian 和 Burlap ；
3. Spring 基于 HTTP 的远程服务；
4. 使用 JAX-RPC 和 JAX-WS 的 Web Service



## 第16章 使用Spring MVC创建REST API

RestTemplate让我们在使用RESTful资源时免于编写样板代码。

除了 TRACE 以外， RestTemplate 涵盖了所有的 HTTP 动作。除此之外， execute() 和 exchange() 提供了较低层次的通用方法来使用任意 的 HTTP 方法。

![](https://img-blog.csdnimg.cn/20190228173109742.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA2NTcwOTQ=,size_16,color_FFFFFF,t_70) 

     

## 第17章 Spring消息














































 















