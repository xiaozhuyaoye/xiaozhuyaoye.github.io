---
layout: post
title: Spring的高级装配
date:  2017-12-5
img:  /post2/logo.jpg
tags: [Java框架,Spring]
author:  Taylor Toby
---

&emsp;对于简单的依赖关系，<a href="https://xiaozhuyaoye.github.io/post2_spring_part1/">《Spring中Bean的声明与装配》</a>一文中介绍的集中方式已经能够解决，但是日常的开发过程中，还是会有一些特殊一点的需求，Spring也提供了方法去解决一些需求。本文主要介绍5个内容：profile、条件化地注入bean、处理装配过程中的歧义、创建不同作用域的bean、运行时注入属性。

# 一. Spring profile

&emsp;profile主要解决的是不同环境注入不同的bean或者通过不同的方式获得bean的需求。假设开发、测试跟生产分别使用安装在不同服务器的数据库，那么Spring在获得数据源的时候，是需要根据情况决定具体的。再比如说开发阶段引入了一些调试、测试的工具类，部署到生产跟测试环境的时候不需要带过去。虽然可以写不同的XML，但是如果这样的话在不同的环境部署的时候都需要重新构建，有可能会引入新的bug。这种情况下使用profile就可以将问题简化，同一个部署文件比如war包，根据运行时的环境决定要注入哪些内容。

## 1.1 如何配置profile bean

&emsp;在Java配置中，我们使用@Profile注解来指定某个bean属于哪个profile，例如下面的代码声明了配置类中的bean，运行在"dev"环境时才注入。

```
@Configuration
@Profile("dev")
public class DevelpomentProfileConfig {
	......
}
```

&emsp;上面的示例中注解加在了类上面，所以这个类中声明的所有的bean都只有在"dev"环境中才会被注入。在Spring3.2以后的版本，@Profile也可以加在方法上面，配合@Bean注解使用。

```
@Configuration
public class Config() {

	@Profile("dev")
	pulic DataSource devDataSource(){
		......
	}

	@Profile("pro")
	pulic DataSource proDataSource(){
		......
	}
}
```

&emsp;在XML中配置profile bean使用<beans>标签的profile属性，示例如下（为了简洁XML不完整，只突出主要部分）：

```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans">

  <beans profile="dev">
    <jdbc:embedded-database id="dataSource" type="H2">
      <jdbc:script location="classpath:schema.sql" />
      <jdbc:script location="classpath:test-data.sql" />
    </jdbc:embedded-database>
  </beans>
  
  <beans profile="prod">
    <jee:jndi-lookup id="dataSource"
      lazy-init="true"
      jndi-name="jdbc/myDatabase"
      resource-ref="true"
      proxy-interface="javax.sql.DataSource" />
  </beans>
</beans>
```

## 1.2 如何激活profile

&emsp;Spring在确定哪个profile处于激活状态时，需要依赖两个独立的属性：
- spring.profiles.active
- spring.profiles.default

&emsp;在没有设置spring.profiles.active属性的情况下，spring会使用spring.profiles.default的值，如果两个属性均没有设置值，就相当于没有激活的profile，这种情况下就只会创建没有定义在profile中的bean。

&emsp;有多种方式来设置这两个属性：
- 作为DispatcherServlet的初始化参数
- 作为Web应用的上下文参数
- 作为JNDI条目
- 作为环境变量
- 作为JVM的系统属性
- 在集成测试类上使用@ActiveProfiles注解设置

# 二. 根据条件创建Bean

&emsp;假如我们希望bean只有在类路径下包含某个包的时候才创建，或者希望某个bean在另外一个特定的bean声明后在会被创建。Spring 4引入了一个新的@Conditional注解，可以来解决这个问题（XML中可以使用表达式语言来解决这个问题）。

&emsp;现在我们有一个配置类Config和一个是一类WashingMachine，配置类中手动的声明了WashingMachine的bean，接着我们加上@Conditional注解并指定一个条件类ExistConditional。代码如下（实体类WashingMachine没有任何内容所以没有贴代码）：

```
@Configuration
public class Config {      //配置类，手动声明了washingMachine

    @Bean
    @Conditional(ExistConditional.class)
    public WashingMachine washingMachine(){
        return new WashingMachine();
    }
}

@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(classes = Config.class)
public class Test {                             //测试类的代码，自动注入了washingMachine
    @Autowired
    private WashingMachine washingMachine;

    @org.junit.Test
    public void testNull(){
        System.out.println("washingMachine is null?  " + (washingMachine == null));
    }
}

public class ExistConditional implements Condition{  //条件类
    public boolean matches(ConditionContext conditionContext, AnnotatedTypeMetadata annotatedTypeMetadata) {
        Environment environment = conditionContext.getEnvironment();
        return "Windows 10".equals(environment.getProperty("os.name"));
    }
}
```

&emsp;@Conditional传入的条件类ExistConditional实现了org.springframework.context.annotation.Condition接口，这个接口中只有一个方法matches，方法体中指定你的条件并返回布尔值，如果返回false则这个bean不会被创建。

&emsp;上面实现的matches方法有两个入参，是ConditionContext和AnnotatedTypeMetadata的两个实体。这两个实体包含了十分丰富的信息，可以帮助我们获取各种参数从而辅助我们进行条件的判断。

- 1. ConditionContext也是一个接口，包含了五个方法，如下：

```
    BeanDefinitionRegistry getRegistry();    //检查bean的定义

    ConfigurableListableBeanFactory getBeanFactory();     //检查bean的存在和属性

    Environment getEnvironment();     //获取环境变量的值

    ResourceLoader getResourceLoader();     //返回ResourceLoader所加载的资源

    ClassLoader getClassLoader();     //返回ClassLoader加载并检查类是否存在
```

- 2. AnnotatedTypeMetadata也是一个接口，包含了五个方法，如下：

```
    boolean isAnnotated(String var1);

    @Nullable
    Map<String, Object> getAnnotationAttributes(String var1);

    @Nullable
    Map<String, Object> getAnnotationAttributes(String var1, boolean var2);

    @Nullable
    MultiValueMap<String, Object> getAllAnnotationAttributes(String var1);

    @Nullable
    MultiValueMap<String, Object> getAllAnnotationAttributes(String var1, boolean var2);
```
&emsp;通过AnnotatedTypeMetadata的实例我们可以获得使用@Conditional的方法上是否还有其他的注解以及注解的信息。

&emsp;这个测试案例中我们通过ConditionContext实例获取到Environment再获取我们当前运行的环境，当运行在Windows 10的时候返回true。可以结合代码自行测试，地址在文章末尾。

# 三. 处理自动装配的歧义

&emsp;上一篇文章中介绍了使用注解让Spring进行自动装配，减少显式的配置数量。不过那里里面的实例十分的简单，Machine接口下只有一个实现类。假如说现在Machine接口下有三个实现类，我去让Spring注入Machine的一个实例的时候，会不会出现问题呢？

```
public class AirConditioning implements Machine

public class SweepingRobot implements Machine

public class WashingMachine implements Machine

@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(classes = Config.class)
public class Test {
    @Autowired
    private Machine machine;

    @org.junit.Test
    public void testNull(){
        machine.work();
    }
}
```

&emsp;AirConditioning、SweepingRobot、WashingMachine三个类均实现了Machine接口，测试用例中我们声明让Spring自动注入一个Machine类型的实例。执行test()方法，产生如下异常：

```
org.springframework.beans.factory.UnsatisfiedDependencyException: Error creating bean with name 'Test': Unsatisfied dependency expressed through field 'machine'; nested exception is org.springframework.beans.factory.NoUniqueBeanDefinitionException: No qualifying bean of type 'machine.Machine' available: expected single matching bean but found 3: airConditioning,sweepingRobot,washingMachine
```
&emsp;在创建Machine实例的时候，Spring找到了3个具体的实现，如果Spring找到了不止一个符合条件的对象的时候会抛出异常，不会创建实例。要解决这个问题，我们可以通过标示首选的bean和限定自动装配的bean两种方法来解决。

## 3.1 标示首选的bean

&emsp;在声明bean的时候，我们可以使用@Primary注解将一个bean标示为首选bean，该注解可以配合@Component注解使用，也可以配合@Bean注解在JavaConfig中显示的声明。XML中使用<bean>标签中的primary属性。XML配置的示例如下:

```
<bean id="washingMachine" class="entity.WashingMachine" primary="true"/>

<bean id="airConditioning" class="entity.AirConditioning"/>

<bean id="sweepingRobot" class="entity.SweepingRobot"/>
```

&emsp;需要注意的是，只能标示一个bean为首选项，标示多个bean为首选项跟没有标示是一样的，Spring都无法确定要选择哪个类进行实例化。

# 3.2 使用限定标识符

```
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(classes = Config.class)
public class Test {
    @Autowired
    @Qualifier("airConditioning")
    private Machine machine;

    @Test
    public void testNull(){
        machine.work();
    }
}
```

&emsp;把之前的@Primary注解去掉，这个时候所有的实体又都回到了“平等”的地位。这里我们在自动注入的时候使用了@Qualifier注解并且传入了参数"airConditioning"。在使用了@Component注解的类被创建的时候类名首字母小写的字符串会被作为ID，创建的时候如果没有指定限定符的话那么默认的限定符就是ID，所以我们在这里可以正常的注入AirConditioning实例。我们也可以在声明bean的时候使用@Qualifier指定它的限定名。(比如上面的实例，在SweepingRobot上使用@Qualifier("floor")指定限定名为"floor"，自动注入时的参数也换成"floor")

&emsp;使用默认的限定符会有一些局限，加入类名发生了变更那么配置也需要一起变更。如果我们指定了限定名，那么即使类名发生了变更，配置上也不需要做修改。

&emsp;@Qualifier相当于一个tag，我们在类上面使用这个注解相当于给这个类一个标签，在创建bean的时候如果限定条件给的充足，那么是可以限制成只有一个满足条件的类。但是是不是也存在这种情况，比如SweepingRobot添加了一个"housework"标签表示他是用来做家务的，WashingMachine也添加了一个"housework"标签。这个时候我可以进一步细分，SweepingRobot是跟家务里面的地板有关，我再添加一个"floor"标签，WashingMachine是洗衣服的，再添加一个"cloths"标签。声明的时候指定"homework"和"floor"就可以唯一确定是谁。Java并不允许出现重复的注解，虽然在Java8以后定义注解时使用了@Repeatable注解的注解允许重复出现，但是@Qualifier在定义的时候并没有使用@Repeatable注解。想要实现这种功能，我们可以自定义限定符注解。实例如下。

```
@Component
@Cloths
@Housework
public class SweepingRobot implements Machine{
    public void work() {
        System.out.println("I can sweep the floor");
    }
}

@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(classes = Config.class)
public class Test {
    @Autowired
    @Housework
    @Cloths
    private Machine machine;

    @org.junit.Test
    public void testNull(){
        machine.work();
    }
}
```

# 四. 创建不同作用域的bean

&emsp;默认情况下，Spring应用中上下文的bean都是单例的。如果bean包含状态时，单例的bean就不是很合理了，因为bean的状态发生了变化，后续重用的时候就有可能会出现问题。

&emsp;Spring中定义了多中作用域，可以基于这些作用域创建bean：

  - 单例(Singleton)：整个应用中只创建一个bean的实例
  - 原型(Prototype)：每次注入或者通过应用上下文获取的时候都会创建一个新的bean
  - 会话(Session)：在web应用中，为每个会话创建一个bean
  - 请求(Request)：在web应用中，为每个请求创建一个bean

&emsp;单例作用域是默认的作用域，如果要选择其他的作用与要使用@Scope注解，可以配合@Component和@Bean注解一起使用。比如：

```
@Component
@Scope(ConfigurableBeanFactory.SCOPE_PROTOTYPE)   //参数也可以直接用字符串"prototype"代替
public class Notepad {}
```

&emsp;上面四种作用域，要制定使用哪种的方式其实都是一样的。但是web应用中声明时还需要指定proxyMode的值。假设现在有一个购物车类ShoppingCar，作用域为session，ShoppingService是进行结账的服务，需要对购物车进行处理，所以持有了一个ShoppingCar的代理对象。这里持有的是代理对象也是可以理解的，因为ShoppingCar作用域是session，只有当用户进入创建了绘画之后才会被创建，但是ShoppingService可能是Singleton的，在Singleton初始化的时候ShoppingCar是没有的，所以这个时候先给他一个代理类，跟ShoppingCar暴露一样的接口，只有当调用的时候，代理才会将调用委托给会话内的真正的ShoppingCar bean。

&emsp;所以当上面例子中的ShoppingCar是接口时，proxyMode的值设置为ScopedProxyMode.INTERFACES，是具体的实现类的时候设置为ScopedProxyMode.TARGET_CLASS。XML中的配置如下：

```
<bean id="car" class="xx.xx" scope="session">
  <aop:scoped-proxy />  默认CGLib，proxy-target-class设置为false会生成基于接口的代理
</bean>
注意要在XML配置中声明aop的命名空间
```

# 五. 运行时注入

&emsp;一些时候，我们会把参数放到配置文件中，再让Spring从配置文件中读取参数，创建bean的时候将使用这些参数。下面的方法可以实现这个功能。

## 5.1 使用属性占位符注入外部的值

&emsp;我们可以在配置类中声明属性源，自动注入Environment对象，通过Environment获取相应的属性。我们通过一个示例来说明一下。

```
定义一个WashingMachine类，一个含参构造器，一个show方法输出相应的信息。

@Component
public class WashingMachine {
    private String brand;
    private double price;

    public WashingMachine(String brand, double price) {
        this.brand = brand;
        this.price = price;
    }

    public void show(){
        System.out.println(brand+"牌洗衣机售价"+price+"元");
    }
}

定义配置类，通过@PropertySource注解声明属性源，并且自动注入Environment对象
通过Environment获得Property作为WashingMachine构造器的入参

@Configuration
@PropertySource(value = "classpath:info/WashingMachineInfo.properties",encoding = "GBK")
public class Config {
    @Autowired
    private Environment env;

    @Bean
    public WashingMachine washingMachine(){
        return new WashingMachine(env.getProperty("brand"),env.getProperty("price",Double.class));
    }
}
```

&emsp;通过Environment获取Property有几种方法：
```
String getProperty(String key)    通过key获取字符串形式的property

String getProperty(String key,String defaultValue)    在没有该property时返回默认值defaultValue

T getProperty(String key,Class<T> type)   指定返回值类型

T getProperty(String key,Class<T> type,T defaultValue)

String getRequiredProperty(String key)  获取不到property时抛出异常IllegalStateException   

T getRequiredProperty(String key,Class<T> type)

String[] getActiveProfiles()  返回激活的profile名称的数组

String[] getDefaultProfiles()  返回默认的profile名称的数组

boolean acceptsProfiles(String... profiles)  如果environment支持给定的profile返回true
```

## 5.2 解析属性占位符

&emsp;在XML文件中要使用占位符的话，要使用"${...}"来包装属性名称。比如使用下面的配置声明WashingMachine

```
<bean id="washingMachine" class="entity.WashingMachine">
    <constructor-arg name="brand" value="${brand}" />
    <constructor-arg name="price" value="${price}" />
</bean>
```

&emsp;要使用占位符，我们要指定"数据源"，也就是持有配置文件内容的对象。我们可以使用PropertyPlaceholderConfigurer或者PropertySourcesPlaceholderConfigurer（支持通过Environment及其属性源来解析占位符）。如果对文件内容没有什么特殊处理，比如加密解密，我们可以在XML中指定。

```
<context:property-placeholder file-encoding="gbk" location="info/WashingMachineInfo.properties"/>
```

本文的代码地址：https://github.com/xiaozhuyaoye/SpringInActionDemos