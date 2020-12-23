---
title: Spring Boot基础知识
date: 2020-12-15 10:49:50
keywords: SpringBoot
categories: 
- Spring Boot
tags:
- Spring Boot
description: Spring Boot基础知识
---
##### 控制反转(Inversion of Control,IoC)

1. ###### IoC容器简介

    IoC模式下，控制权发生了反转，即从应用程序转移到了IoC容器，所有组件不再由应用程序自己创建和配置，而是由IoC容器负责。
    IoC的实现方式：

    - `依赖查找(Dependency Lookup)`：容器中的受控对象通过容器的API来查找自己所依赖的资源和协作对象。

    - `依赖注入(Dependency Injection)`：容器全权负责组件的装配，它会把符合依赖关系的对象通过属性(JavaBean中的setter)或者是构造子传递给需要的对象。

    IoC容器是一个管理Bean的容器，Spring IoC容器主要基于以下两个接口：

    - `BeanFactory`
    - `ApplicationContext`

2. ###### 装配Bean

    - `@Configuration`
    
        @Configuration标注在类上，代表这是一个Java配置文件，用来配置Spring容器(应用上下文)。

    - `@Bean`

        @Bean标注在返回实例的方法上，将方法返回的对象装配到IoC容器中。如果未指定name属性，则将方法名作为Bean的名称。

    - `@ComponentScan`
    
        @ComponentScan从定义的扫描路径中找出标识了需要装配的类自动装配到Bean容器中。默认扫描当前包和其子包下的路径。

    - `@Component`
    
        @Component，标注在类上，标明这个类需要被扫描装配到Bean容器中。
        @Component是个通用注解，还可以针对不同的使用场景使用@Repository、 @Service、@Controller注解。
    

3. ###### 条件装配Bean

    - `@Conditional`
    
        配置的类需要实现Condition接口，实现matches方法。满足自定义情况下返回true则会装配Bean。

    - `@Profile`

        当属性文件中的`spring.profiles.active`或`spring.profiles.default`有配置时，@Profile标注的且属性值相同的Bean才会装配。

4. ###### 依赖注入

    - `@Autowired`
    
        首先根据类型找到对应的Bean，如果对应类型的Bean不是唯一的，那么它会根据其属性名称和Bean的名称进行匹配。
        @Autowired是一个默认必须找到对应Bean的注解，如果不能确定其标注属性一定会存在并且允许这个被标注的属性为null，那么可以配置属性required为false。
    
    - `@Primary`
    
        @Primary注解在Bean上，是一个修改优先权的注解。如果发现多个同样类型的Bean时，优先使用@Primary注解的Bean。

    - `@Quelifier`
    
        与@Autowired组合使用，通过类型和名称一起找到Bean。
     
5. ###### Bean的生命周期

    Spring Bean的初始化流程：

    1. 资源定位
    
        - xml中定义的Bean标签
        - @ComponentScan所定义的扫描包中的Bean
        - @Configuration类中，注解@Bean的方法返回的Bean

    2. Bean定义
    
        将Bean的定义保存在BeanDefinition的实例中

    3. Bean注册
    
        BeanDefinition注册到BeanFactory。
        默认情况下，之后Spring会继续完成Bean的实例化和依赖注入。如果配置了lazyInit(懒加载)则不会在此后完成实例化和依赖注入。

    4. 实例化
    
        在第一次从BeanFactory当中getBean()时， 这个Bean的实例才会真正生成

    5. 依赖注入

        搜索Bean中的@Autowired注解,，生成注入依赖的信息，初始化Bean。

        > 实例化和初始化的区别：实例化是在JVM的堆中创建了这个对象实例，此时它只是一个空的对象，所有的属性为Null。而初始化的过程就是讲对象依赖的一些属性进行赋值之后，调用某些方法来开启一些默认加载。

    Bean生命周期：[参考](https://chaycao.github.io/2020/02/15/%E5%A6%82%E4%BD%95%E8%AE%B0%E5%BF%86Spring-Bean%E7%9A%84%E7%94%9F%E5%91%BD%E5%91%A8%E6%9C%9F/)

    ![生命周期](life-cycle.png)

6. ###### Bean的作用域

    @Scope
    |作用域类型|作用域描述|
    |:--:|:--:|
    |singleton|默认值，IoC容器只存在单例|
    |prototype|每当从IoC容器中取出一个Bean，则创建一个新的Bean|
    |session|HTTP会话|
    |request|Web工程单次请求(request)|
    |globalSession|在一个全局的HTTP Session中，一个Bean定义对应一个实例。|

7. ###### profile

    - `@Value`
    
        通过@Value("${datasource.url}")注解，使用\${...}占位符读取配置在属性文件的内容。还可以使用Spring EL表达式(#{...})。

    - `@ConfigurationProperties`

        注解中配置的字符串与POJO的属性名称组成属性的全限定名去配置文件里查找，这样就能将对应的属性读入到POJO当中。

    - `@PropertySource`

        定义属性文件，把它加载到Spring的上下文中。`value`属性可以配置多个配置文件，使用`classpath:`前缀，意味着去类文件路径下找到属性文件。

##### 面向切面编程(Aspect Oriented Programming,AOP)

1. ###### AOP术语

    - 连接点(join point)
    
        对应的是具体被拦截的对象，因为Spring只能支持方法，所以被拦截的对象往往就是指特定的方法。

    - 切点(point cut)
    
        切面不单单应用于单个方法，也可能是多个类的不同方法，这时，可以通过正则式和指示器的规则去定义，从而适配连接点。切点就是提供这样的一个功能概念。
    
    - 通知(advice)
    
        就是按照约定的流程下的方法，分为前置通知(before advice)、后置通知(after advice)、环绕通知(around advice)、返回通知(afterReturning advice)和异常通知(afterThrowing advice)，它会根据约定织入流程中，需要明白它们在流程中的顺序和运行条件。

    - 切面(aspect)
    
        切面由切点和通知组成，Spring AOP根据切面定义的横切逻辑织入到切面所指定的连接点中。
    
    - 目标对象(target)
    
        即被代理对象。
    
    - 引介(introduction)
    
        引介是一种特殊的通知，它为类添加一些属性和方法，增强现有的Bean功能。
    
    - 织入(weaving)

        Spring采用动态代理技术，为原有服务对象生成代理对象，然后将与切点定义匹配的连接点拦截，并按约定将各类通知织入约定流程的过程。

    - 代理(proxy)

        为某一个对象创建一个代理对象，程序不直接用原本的对象，而是由创建的代理对象来控制原对象，通过代理类这中间一层，能有效控制对委托类对象的直接访问，也可以很好地隐藏和保护委托类对象，同时也为实施不同控制策略预留了空间
        - 动态代理：在程序运行时，运用反射机制动态创建而成
            - JDK动态代理：自带的，要求目标对象实现一个接口
            - CGLIB动态代理：基于继承来实现代理，所以无法对final类，private方法和static方法实现代理
        - 静态代理：由程序创建或特定工具自动生成源代码，在程序运行前，代理类的.class文件就已经存在

        > 如果目标对象实现类接口，则默认采用JDK动态代理
        > 如果目标对象没有实现接口，则采用CGLIB进行动态代理

2. ###### @AspectJ

    - `@EnableAspectJAutoProxy`
    
        启用@AspectJ注解配置方式
    
    - `@AspectJ`
    
        注解在Bean上声明为切面
    
    - `@Pointcut`
    
        定义切点

        - `execution`
        
            ```java
            @Pointcut("execution(* com.cxh.test.dao.TestDao.*(..))")
            ```
            它与方法执行连接点匹配。在这里，第一个通配符匹配任何返回值，第二个通配符匹配任何方法名称，(..)模式匹配任意数量的参数(零个或多个)。

        - `within`
        
            ```java
            @Pointcut("within(com.cxh.test.dao.TestDao)")
            ```
            这将匹配限制为某些类型的连接点。
            ```java
            @Pointcut("within(com.cxh..*)")
            ```
            还可以匹配com.cxh包或子包中的任何类型。

        - `this`
        
            ```java
            @Pointcut("this(com.cxh.test.dao.TestDao)")
            ```        
            匹配当前AOP代理对象类型,作用于代理对象

        - `target`

            ```java
            @Pointcut("target(com.cxh.test.dao.TestDao)")
            ```
            匹配当前目标对象类型,作用于目标对象

        - `args`
        
            用于匹配特定的方法参数
            ```java
            @Pointcut("execution(* *..find*(Long))")
            ```
            该切入点与以find开头且仅具有Long类型的一个参数的任何方法匹配
            ```java
            @Pointcut("execution(* *..find*(Long, ..))")
            ```
            匹配具有任意数量参数但第一个参数类型为Long的方法
        
        - `@target`
        
            匹配的目标对象的类有一个指定的注解

        - `@args`
        
            匹配方法参数所属的类型上有指定的注解
        
        - `@within`
        
            匹配必须包含某个注解的类里的所有连接点
        
        - `@annotation`
    
            匹配有指定注解的方法

    - `@Before`
    
        定义前置通知，通知方法会在目标方法调用之前执行
    
    - `@After`
    
        定义后置通知，通知方法会在目标方法返回或者抛出异常后调用
    
    - `@AfterReturning`
    
        定义返回通知，通知方法会在目标方法返回后调用
    
    - `@AfterThrowing`
    
        定义异常通知，通知方法会在目标方法抛出异常后调用
    
    - `@Around`

        定义环绕通知，该注解能够让你在调用目标方法前后，自定义自己的逻辑
    
    - `@DeclareParents`

        ```java
        @DeclareParents(value="concert.Performce+", defaultImpl=DefaultEncoreable.class)
        private Encoreable encoreable;
        ```
        @DeclareParents注解由三部分组成：
        - `value`属性指定了哪种类型的Bean要引入该接口。在本例中，也就是所有实现 Performance的类型。(标记符后面的加号表示是Performance的所有子类型，而不是Performance本身。)
        - `defaultImpl`属性指定了为引入功能提供实现的类。在这里，我们指定的是DefaultEncoreable提供实现。
        - @DeclareParents注解所标注的静态属性指明了要引入了接口。在这里，我们所引入的是Encoreable接口。

3. 