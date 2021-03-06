# Spring的Cache抽象
## 1. 概述
  Spring 3.0之后，Spring Framework提供了显示添加缓存到Spring Application的支持，和事务支持一样，缓存抽象用最少的代码嵌入方式实现各种缓存方案的一致实现。  
  Spring4.1 开始，随着 [JSR-107注解](https://docs.spring.io/spring/docs/5.0.13.RELEASE/spring-fsramework-reference/integration.html#cache-jsr-107) 和越来越多的自定义选项，缓存抽象更进一步得到显著改进。
## 2.理解缓存抽象
>缓存与缓冲（Cache vs Buffer)
术语中"buffer"与"cache"经常被交替使用，但请注意两者表示的意识是不同的。buffer通常用来表示快慢实体间的中间临时存储。由于一端不得不等待另一端导致慢的影响性能，而buffer通过影响一次余东整个数据块而不是一小块一小块移动从而减缓了这个影响。数据只从buffer写入和读取一次。此外，缓冲器对读取他的对象是可见的。 
另外一方面，根据定义，cache是隐藏的，而且双方都不知道缓存发生。他同样优化了性能，但是是通过快速多次读同样的数据来实现。  
>更多关于`buffer&cache`说明，请查阅维基百科的 [The_difference_between_buffer_and_cache](https://en.wikipedia.org/wiki/Cache_(computing)#The_difference_between_buffer_and_cache)

 在其核心，抽象将缓存用于Java方法，从而基于缓存中的可用信息减少方法的执行次数。也就是说，每次目标方法被执行，抽象都将执行缓存行为，来检查该方法是否有被给定的一样参数执行过。如果有，则直接返回缓存结果而不再去执行实际方法；如果没有，则执行方法，并建结果缓存后返回，下次执行一样的方法是，可以直接返回改缓存结果。这种方式，对于高消耗（不管CPU或IO方面）的方法对一组给定参数只需执行一次，执行结果可复用，而不必再次重复运行该方法。缓存的逻辑是透明的并不会对调用方有任何影响。

 > 显然，这个方式只适用于那些给定特定的输入（或者参数）不管执行多少次返回输出（结果）是一致的方法。

其它缓存的操作也是有该抽象提供，比如更新缓存内容或者删除一列数据中的一项，这些对于应用运行过程中可能改变的数据来说是非常有用。

和Spring Framework的其它服务一样，缓存服务也是一个抽象（不是一个缓存实现）且需要使用实际存储去存储缓存数据-也就是说，抽象使开发者可以不必去编写缓存逻辑但也不提供实际存储。抽象通过org.springframework.cache.Cache和org.springframework.cache.CacheManager 实现。

这有些开箱即用的实现方案：JDK基于缓存的java.util.concurrent.ConcurrentMap/Ehchace2.x/Gemfire cache/Caffeine 以及符合JSP-107规范的缓存（eg.Ehcache3.x）.可通过查看 [Plugging-in different back-end caches](https://docs.spring.io/spring/docs/5.0.13.RELEASE/spring-framework-reference/integration.html#cache-plug) 获取更多关于其他缓存仓库/服务如何集成（插入）的更多信息。

> 缓存抽象对于多线程与多进程环境并无特殊处理，所以这些特性多是有缓存实现去处理。

如果你有个多进程环境（比如：应用部署在不通的服务节点上），你将需要去配置相应的缓存服务。根据你的用例场景，多个节点上建相同数据副本可能就满足，但如果应用运行中数据需要变更，则可能需要启动其它的传播机制。

缓存的特定功能项与编程实现的典型`get-if-not-found-then-proceed-and-put-eventually`代码块是一致的：没有应用锁，几个线程可能尝试并发加载同一个项。同样的适用于`eviction`:如果几个线程尝试去同时更新和删除数据，你可能使用陈旧的数据。有些缓存服务该领域上提供了高级功能，更多细节可以去查看你正在使用缓存的相应文档。

使用缓存抽象，开发者需要关注两个方面：
+ **caching declaration**: 标识需要缓存的方法及其相应需要的策略
+ **cache configuration**: 配置存储和读取数据的缓存

## 2.3 基于声明注解的缓存
为了缓存声明，抽象提供了一组Java注解
+ `@Cacheable` 开启缓存填充
+ `@CacheEvict` 触发缓存清楚
+ `@CachePut` 不干扰方法执行时更新缓存
+ `@Caching` 对于要用于该方法的多个缓存重新分组
+ `@CacheConfig` 类级别上共享一些与缓存相关的公共设置

让我们更进一步了解以下各个注释：  
### 2.3.1 @Cacheable 注解
顾名思义，`@Cacheable` 用于声明该方法可缓存-也就是，该方法会将结果存储进缓存，以此类推，随后的调用（具有相同的参数）时，缓存中的值直接返回不再去执行该方法。最简单的形式，就是注解声明需要与带注解的方法相关联的缓存的名称。
```java
@Cacheable("books")
public Book findBook(ISDN isbn){...}
```
上面的片段中，`findBook`方法关联了名称是`books`的缓存。每次该方法被调用时，都会检查缓存，是否该方法已经执行过不需要再次重复。大部分情况下，只声明一个缓存，但注解允许指定多个名称，以便使用多个缓存。在这个种情况下，执行方法前将检查每个缓存-如果至少命中一个缓存，那个关联值将直接返回：
> 即使缓存方法并没有实际执行，所有其他不包含该值的其它缓存也将同时被更新（上面的指定多个缓存值情况，不通缓存间默认会再被调用到时被同步）
```java
@Cacheable({"books","isbns"})
public Book findBook(ISDN isbn){...}
```
#### 默认key生成器
因为缓存本质上是一个key-value存储仓库，每次缓存方法调用是需要转换为适合访问缓存的key值。开箱即用的情况下，缓存抽象使用了基于以下算法的KeyGenerator(key生成器)。

+ 没有参数值，直接返回`SimpleKey.EMPTY`
+ 如果只有一个参数，返回该参数
+ 如果超过一个参数，返回一个包含所有参数的`SimpleKey`

这种方法适合于大多数场景，只要参数有自然值且实现了`hashCode()`与`equals`方法。如果不是则需要去改变相应策略。

要提供不同的默认key生成器，需要去实现一个`org.springframework.cache.interceptor.KeyGenerator`接口。

> 随着Spring4.0的发布，默认可以生成策略发生了改变，早期版本Spring使用的生成策略，在多个key参数情况下，只考虑了参数的`hashCode()`而不是`equals()`;这样就有可能引起意外的key冲突（背景信息可以查看[SPR-10237](https://github.com/spring-projects/spring-framework/issues/14870)）。新的`SimplekeyGenerator` 为这样场景使用了一个复合键。
>
>如果你想要继续使用早期的key策略，你可以配置过期的`org.springframework.cache.interceptor.DefaultKeyGenerator`类或者创建一个自定义基于hash的‘KeyGenerator’实现。

#### 自定义key生成声明
由于缓存是通用的，因此很可能目标方法有各种签名，而这些并不能简单的映射到缓存结构的顶部。当目标方法有多个参数且只有其中部分适合缓存（其它参数可能只是方法的逻辑）时，这一点就变成很明显。例如：
```java
@Cacheable("books")
public Book findBook(ISBN isbn, boolean checkWarehouse, boolean includeUsed)
```
匆匆一看，虽然后面这两个boolean参数会影响书的查找方式，但对于缓存是没用处的。更进一步如果两个中一个有用，一个没用呢？

对于这种场景，`@Cacheable`注解支持使用者去定制key如果通过它的key属性值来生成。开发者可以使用[SpEL](https://docs.spring.io/spring/docs/5.0.13.RELEASE/spring-framework-reference/core.html#expressions)表达式去提取有效参数（或者其他关联属性）、执行操作或者甚至不编码或实现任何接口下直接调用方法。这是默认生成器最推荐的处理方式，因为随着代码库的增长，方法上再签名越趋于不同；虽然默认策略能对些方法有效，但很难对全部方法有效。  
下面是一些SpEL的声明例子-如果你不熟悉这块，可以先抽出点时间去熟悉阅读下[Spring Expression Language](https://docs.spring.io/spring/docs/5.0.13.RELEASE/spring-framework-reference/core.html#expressions):

```java
@Cacheable(cacheNames="books",key="#isbn")
public Book findBook(ISBN isbn,boolean checkWarehouse,boolean includeUsed)

@Cacheable(cacheNames="books",key="#isbn.rawNumber")
public Book findBook(ISBN isbn,boolean checkWarehouse,boolean includeUsed)

@Cacheable(cacheNames="books",key="#T(someType).hash(#isbn)")
public Book findBook(ISBN isbn,boolean checkWarehouse,boolean includeUsed)

```
上面的片段展示了，怎样简单的去选择一个确切的参数、或者其中的属性，甚至执行一个静态方法。

如果生成key的算法太定制化或者需要共享，你可以再操作中自定义个`keyGenerator`。然后，指定要使用的KeyCenerator bean的实现的名称：
```java
@Cacheable(cacheNames="books",keyGenerator="myKeyGenerator")
public Book findBook(ISBN isbn,boolean checkWarehouse,boolean includeUsed)
```
> 注意：上面key和keyGenerator参数是互斥的，如果同时指定两个将会导致异常。

#### 默认缓存解析器
开箱即用，缓存抽象使用一个简单的`CacheResolver`，它从操作级别上使用`CacheManager`配置定义的缓存对象获取。  
要提供不同的默认缓存解析器，就需要去实现`org.springframework.cache.interceptor.CacheResolver`接口。 

#### 自定义缓存解析器
默认的缓存解析器能很好适应于以单个`CacheManager`与不复杂的缓存解析需求的应用程序。  
为了以多个缓存管理器工作的程序，则可能需要给每个操作单独设置`cacheManager`。
```java
@Cacheable(cacheNames="books",cacheManager="anotherCacheManager")
public Book findBook(ISBN isbn){...}
```
同样也可以key生成器完全一样的方式的替换`CacheResolver`。解析对于每个缓存操作是必须的，提供一个机会，实现基于运行时参数去解析在用的实际缓存。
```java
@Cacheable(cacheResolver="runtimeCacheResolver")
public Book findBook(ISBN isbn){...}
```
> Spring4.1之后，缓存注解的value属性不再是强制性的，因为这份信息可通过`CacheResolver`来提供，不管注解的内容是什么。  
与key与keyCenerator类似，cacheManager与cacheResolver参数也是互斥的，如果一个操作两个都指定则会抛出异常，就像自定义了`CacheManager`将会被`CacheResolver`所忽略。这个可能不是你所预期的。

#### 缓存同步
再多线程环境中，有些操作可能被相同参数同时并发执行（尤其在启动是）。默认下，缓存抽象并不加任何锁，同样的值可能被计算多次，这样缓存的目的就无效了。

对于这种场景，`sync`属性可以用来指示对应的缓冲提供程序给在值运算时对缓存入口添加一个锁。这样，就只有一个线程在忙于值的计算，而其它线程阻塞，直到这个值被更新进缓存。
```java
@Cacheable(cacheNames="foos",sync=true)
public Foo executeExpensiveOperation(String id){...}
```
> 注意：这个可选项特性，你最中意的的缓存库可能不支持他。所有核心框架提供的`CacheManager`实现都支持他。可以去查阅你所选的缓存服务提供程序的文档，了解更多详情。

#### 条件式缓存：
有些时候，一个方法并适合与每次多缓存数据（比如，可能依赖于给定的参数）。缓存注解支持这样的功能，通过给`condition`参数设置一个能计算出true/false的SpEL表达式。如果true，这个方法这缓存-如果不是，则不缓存，这样不管返回值是什么或者使用参数是什么，每次都会执行。一个快速例子：下面方法只有在参数`name`长度短于32位才可缓存
```java
@Cacheable(cacheNames="book",condition="#name.length()<32")
public Book findBook(String name)
```
除了condition还有一个参数，`unless`参数用于禁止值添加到缓存中，不同于`condition`,`unless`表达式是在方法被执行之后才计算的，扩散上面的例子-我们希望只添加paperback books到缓存中（paperback 平装书，hardback 精装书）
```java
@Cacheable(cacheNames="book",condition="#name.length()<32",unless="#result.hardback")
public Book findBook(String name)
```
缓存抽象支持`java.util.Optional`,使用它时只有当其内容存在时才当作缓存使用。`#result`是关联也业务实体且不是在一个支持包装器（wrapper）上，因此先前例子可以按以下重写：
```java
@Cacheable(cacheNames="book",condition="#name.length<32",unless="#result?.hardback")
public Optional<Book> findBook(String name)
```
注意`result`一直引用的还是`Book`而不是`Optional`。由于他可能是`null`,所以我们需要使用安全索引操作符（safe navigation operator）。

#### 可用的缓存SpEL运算上下文
每个`SpEL`表达式运算都需要一个专门的`context`。除了构建参数外，框架还提供了缓存相关的元数据，比如参数名。下面表格罗列的项都是可用的，因此我们可以使用他们来计算key和conditional等。

| 名称        | 位置         | 描述              | 例子
| :---       | :---         | :---              | :--- 
| methodname | root object  |被执行方法名称  |#root.methodName
| method     | root object  |被执行的方法    |#root.method.name
| target     | root object  | 正在调用的目标对象 | #root.target
| targetClass| root object  | 正在调用的目标类 | #root.targetClass
| args | root object | 用于调用目标的参数（数组）| #root.args[0]
| caches | root object | 用于执行当前方法的缓存集合 | #root.caches[0].name
| argument name| evaluation context | 任何方法参数的名称，如果由于一些原因名称不可用（比如，没调试信息），参数名称一样在#a<#args>下可用，其中args代表参数索引（从0开始） | #iban 或者#a0 (还可以使用#p0或#p<args>符号作废别名)
| result | evaluation context | 方法调用结果（要缓存的值）。只有再unless表达式，缓存添加表达式（去计算key时），缓存失效表达式（判断为失效时）情况下可用。此外对于支持的包装器，比如`Optional`,`#result` 引用的是实际对象，而不是包装类 | #result

### 2.3.2 @CachePut 注解
对于缓存需要更新，并且不能影响方法执行情况时，就可以使用`@CachePut`注解。也就是说，执行方法，并将结果放入到缓存中（根据`@CachePut`选项）。他支持和`@Cacheable`一样的选项，且应该用于缓存填充，而不是方法流程的优化。
```java
@CachePut(cacheNames="book",key="#isbn")
public Book updateBook(ISBN isbn,BookDescriptor descriptor)
```
>注意：在同一个方法中使用@CachePut和@Cacheable中注解是强烈不推荐的，因为他们是不同的行为。后者通过使用缓存跳过方法的执行，而前者则强制执行以更新缓存。这会导致异常发生意外行为，且除了特定的案例（例如，具有将他们彼此排斥的条件注解）外，应该避免这样额声明。还有需要注意一点，这样的条件不依赖与result对象，因为这些条件是预校验以确认排除。

### 2.3.3 @CacheEvict 注解
缓存抽象不仅仅支持往缓存服务填充，同样也支持失效清理。这对于程序移除旧的或者无用的数据很有用。与`@Cahceable`相反，`@CacheEvict`声明的方法用于缓存的清理，也就是该方法扮演这一个从缓存删除数据的触发器。如其他注解一样，`@CacheEvict`需要指定一个或多个他应该影响的缓存， 允许一个自定义缓存与key解析器或者定制的condition,除此之外，还有额外的参数`allEntries`,用于指定是否缓存访问内的失效，而非基于某个key。
```java
@CacheEvict(cacheNames="books",allEntries=true)
public void loadBookds(InputStream batch)
```
这个选项非常方便，在一个缓存区域需要清空的时候-不用一个个去遍历失效（这将花很长时间，因此效率很低），所有的项在入上面展示的一个操作就全部移除的。请注意，在这种场景中框架忽略了任何key的指定，因为本身也是没用的（因为失效并不基于某个entry）。  

你也可以通过`beforeInvocation`指定失效操作发生在方法执行后（默认）还是方法执行前。先前再rest注解提供过类似场景-一旦方法执行成功，缓存上的动作就会执行。如果方法不执行（因为缓存命中）或者过程异常了，失效操作将不会触发。而后者（`beforeinvocation=true`）则会一直在方法被执行前触发-在失效不关联方法执行结果的场景下这个非常有用。

有一个重要点需要注意，使用`@CacheEvict`的方法可能是void-因为该方法作用是触发器，返回结果忽略了（因为他们并不与缓存交互）-这和`@Cacheable`场景中需要添加或变更缓存中的值而需要返回结果不同。

### 2.3.4 Caching 注解
一些情况下需要制定相同类型的多个注解，比如`@CacheEvict`与`@CachePut`，例如，因为不同的缓存可能条件或者key表达式是不同的。`@Caching`允许多个嵌套使用`@Cacheable`,`@CachePut`,`@CacheEvict`在同一方法上。
```java
@Caching(evict={@CacheEvict("primary"),@CacheEvict(cacheNames="secondary",key="#p0")})
public Book importBook(String deposit,Date date);
```
### 2.3.5 @CacheConfig 注解
目前为止，我们看到的缓存操作提供了很多自定义选项，且他们可以操作基础上来设置。但是，有些自定义选项适用于类的所有操作时，可能需要反复重复配置。为此，在类中一个个缓存操作去指定配置的属性可以直接在类级别上定义。这就是`@CahceConfig`发挥作用的地方。
```java
@CacheConfig("books")
public class BookRepositoryImpl implements BookRepository {

    @Cacheable
    public Book findBook(ISBN isbn) {...}
}
```
`@CacheConfig`是类级别省的注解，应许共享缓存名称、定制的`KeyGenerator`、定制的`CacheManager`以及定制化的`CacheResolver`。在类级别上添加这个注解，就不需要每个缓存操作去开启了。

操作级别的定制化建覆盖`@CacheConfig`的定制化，所以，每个定制化缓存操作有三个级别：
* 全局配置： 适用`CacheManager`,`KeyGenerator`
* 类级别 ：适用`@CacheConfig`
* 操作级别

### 2.3.6 启动缓存注解
需要重点注意的，虽然生命的缓存的注解，但是并不会自动触发他们生效-和Spring里面很多东西一样，该功能必须以生命方式启用(也就是说，如果你有时候怀疑缓存是问题所在，你可能只需要移除一个启动配置，而不是你全部的缓存注解)

启用缓存注解，直接添加`@EnableCaching` 到你的一个`@Configuration`类就可：
```java
@Configuration
@EnableCaching
public class AppConfig {
}
```
或者Xml配置使用`cache:annotation-driven`
```xml
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:cache="http://www.springframework.org/schema/cache"
    xsi:schemaLocation="
        http://www.springframework.org/schema/beans https://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/cache https://www.springframework.org/schema/cache/spring-cache.xsd">

        <cache:annotation-driven/>
</beans>
```
`cache:annotation-driven`与`@EnableCaching`注解允许指定各种选项，这些选项影响通过AOP将相应缓存行为添加到应用中，这种配置其实是有意`@Transactional`配置一致的。

> 注意：默认的缓存注解的advice模式是代理，代理只允许通过代理的方式调用才能拦截；本地在同一个类内部方法直接调用并不能拦截。为更高的拦截机制，可以考虑切换成‘aspectj’,再编译和加载区间嵌入。

> `<cache:annotation-driven/>`只能在同一个定义的应用上下文中寻找`@Cacheable/@CachePut/@CacheEvict/@Caching`。也意味着，如果你是在`WebApplicationContext`为`DispatcherServlet`添加`<cahce:annotation-driver/>`。他只检查在控制器（Controllers）中的beans而非服务。更多信息请查看MVC章节。

> 


官方文档地址：<https://docs.spring.io/spring/docs/5.0.13.RELEASE/spring-framework-reference/integration.html#cache>
