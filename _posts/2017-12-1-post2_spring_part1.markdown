---
layout: post
title: Spring中Bean的声明与装配
date:  2017-12-1
img:  /post2/logo.jpg
tags: [Java框架,Spring]
author:  Taylor Toby
---

# 一. Spring中Bean的声明与自动装配

## 1.1 目录结构的说明

&emsp;首先说明一下项目的目录结构。项目下有四个package，分别是config、type、entity、test，里面的内容如下：
* **config包：**

	* Config类：可以与XML文件互换，是Bean的关系的配置类，因为现在我们是自动装配，所以这个类里并没有什么内容，只有一个@ComponentScan(basePackageClasses = {Machine.class, People.class})注解定义自动扫描的路径。

* **type包：**

	* Machine接口：包含一个work方法。
	* People接口：包含一个work方法。

* **entity包：**

	* WashingMachine类：实现了Machine接口，work方法中输出“Start washing clothes ......”。
	* XiaoMing类：实现了People接口，并持有Machine的实现类对象，work方法直接调用Machine的实现类对象的work方法。

* **test包：**

	* 该包下面的类均为测试类。

## 1.2 Bean的是声明和自动装配

* 需要让Spring发现的Bean在类名上用@Component注解。
* 需要自动装配的地方使用@Autowired注解，比如XiaoMing中的Machine对象以及测试用例中的对象。

```
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(classes = Config.class)
public class MachineTest {
    @Autowired
    private People people;
    @Autowired
    private Machine machine;

    @Test
    public void test1(){
        System.out.println("People is null:  " + (people == null));
        System.out.println("Machine is null:  " + (machine == null));
        people.work();
    }
}
```
&emsp;运行结果两个对象均不为空，并且正确的输出了内容(people中的work方法直接调用的machine中的，说明people持有的machine对象也正确的装配进去了)。

参考代码在part1/AutoWired路径中  https://github.com/xiaozhuyaoye/SpringInActionDemos

# 二. Spring中Bean的手动装配--通过JavaConfig

&emsp;现在基于第一部分的代码进行改造，分为两个部分：删除，新增。

## 2.1 删除

&emsp;因为这部分不用到自动装配，而是我们手动装配Bean，所以先把自动装配用到的注解删除掉。包括：Config中的ComponentScan注解，现在不用自动扫描了，我们手动指定Bean的路径、WashingMachine和XiaoMing类中的Conponent注解。为了测试的便利，MachineTest中的对象我们还是让他自动注入。

## 2.2 新增

&emsp;首先XiaoMing类中持有了Machine对象，手动装配的时候我们有构造器注入或者set方法注入的方式，所以这里在声明了Machine对象之后，创建空的构造器以及Machine对象为入参的构造器以及Machine属性的set方法。
&emsp;其次，在Config中需要增加我们需要的Bean的声明。以下是Config的代码，我们来分析一下。

```
public class Config {
    @Bean
    public WashingMachine washingMachine(){
        return new WashingMachine();
    }

    /*@Bean
    public XiaoMing xiaoMing(){
        return new XiaoMing(washingMachine());
    }*/

    /*@Bean
    public XiaoMing xiaoMing(WashingMachine washingMachine){
        return new XiaoMing(washingMachine);
    }*/

    @Bean
    public XiaoMing xiaoMing(WashingMachine washingMachine){
        XiaoMing xiaoMing = new XiaoMing();
        xiaoMing.setWashingMachine(washingMachine);
        return xiaoMing;
    }
}
```
&emsp;首先从没有任何依赖的Machine对象开始声明，提供一个方法，返回Machine对象，方法名会默认成为Bean的名称，也可以在@Bean注解中使用name属性指定。

&emsp;接着来声明创建XiaoMing对象，有三种方式，前两种是使用XiaoMing的含参构造器。第一种方法是在实例化XiaoMing对象的时候传入一个方法，该方法返回一个Machine的实例，这个方法如何创建Machine实例可以任意实现，甚至不交给Spring托管，那么如果有两个对象都持有Machine对象并且都是使用这种方式注入的话则这两个对象持有的Machine实例不是同一个，如果该方法是交给Spring托管的那么前面的这种情况下应该是指向同一个对象的。

&emsp;第二种方法是需要一个Machine的实例作为入参，这个时候Spring会自动从上下文中寻找与之匹配的Bean传入该方法中。

&emsp;第三种方法也需要Machine实例作为入参，Spring在传入Machine实例后，调用XiaoMing的不带参构造器，然后调用其中的set方法。

&emsp;上面的三种方法达到的效果是一样的，可以从Part1/RigOutByJava路径下找到代码测试 https://github.com/xiaozhuyaoye/SpringInActionDemos

# 三. Spring中Bean的手动装配--通过XML

&emsp;这个部分同样先删除再添加来达到通过XML声明装配规则的目的。

&emsp;首先，删除Java配置类，其他的不需要做变更。之前通过配置类来说明装配的规则，删除掉之后我们就来创建XML文件来表明这个规则。

```
<bean id="washingMachine" class="entity.WashingMachine" />

<!--<bean id="xiaoMing" class="entity.XiaoMing">
    <constructor-arg name="machine" ref="washingMachine" />
</bean>-->

<!--<bean id="xiaoMing" class="entity.XiaoMing">
    <constructor-arg index="0" ref="washingMachine" />
</bean>-->

<!-- <bean id="xiaoMing" class="entity.XiaoMing">
    <property name="washingMachine" ref="washingMachine" />
</bean>-->

<!--<bean id="xiaoMing" class="entity.XiaoMing">
    <property name="washingMachine" ref="washingMachine" />
    <property name="str" value="Hello World" />
    <property name="list">
        <list>
            <value>"a"</value>
        </list>
    </property>
    <property name="map">
        <map>
            <entry key="b" value="b" />
            <entry key="c" value="c"/>
        </map>
    </property>
    <property name="set">
        <set>
            <value>"d"</value>
            <value>"e"</value>
            <value>"f"</value>
        </set>
    </property>
    <property name="properties">
        <props>
            <prop key="g">"g"</prop>
            <prop key="h">"h"</prop>
            <prop key="i">"i"</prop>
            <prop key="j">"j"</prop>
        </props>
    </property>
</bean>-->
```
&emsp;还是先从没有任何依赖的WashingMachine类来声明，我们这里指定了id和class属性，id是bean的唯一标识，创建出来的bean的名称也是id，class指明了类所在的路径。

&emsp;接下来我们声明XiaoMing，有三种方式，第一种通过含参构造器注入。同样是指定id跟class属性，接着使用<constructor-arg>标签，name属性，表明构造器参数的名称，ref属性，它的值指向了名称为这个值的Bean，Spring会寻找这个Bean并将其作为创建XiaoMing时的参数。

&emsp;第二种方式跟第一种是类似的，第一种方式中使用的是参数的名称，第二种是使用构造器参数的顺序作为参数，使用index属性。这两种方式各有优缺点，比如修改了参数名称但是参数顺序没有变更的情况下，第一种方式就出错了，参数顺序变化的情况下第二种方式也会出问题。

&emsp;第三种方式我们使用属性注入，使用<property>标签，其中的name参数表示属性的名称，ref属性表明引用的Bean的名字。

&emsp;最后一个是在属性注入中，分别在<property>注入对象，字符串，List，Map，Set，Properties的方法。在构造器中注入这些参数的写法也是一样的，这里用属性注入为例子。

可以在可以从Part1/RigOutByXML路径下找到代码测试 https://github.com/xiaozhuyaoye/SpringInActionDemos

# 四. 混合配置

## 4.1 在JavaConfig中引入XML配置

&emsp;我们计划在XML中声明Machine的实例，然后在JavaConfig中引入XML的配置。JavaConfig中的代码如下所示，使用@ImportResource注解。

```
@Configuration
@ImportResource("classpath:ImportXMLToJavaConfig.xml")
public class ImportXMLToJavaConfig {
    @Bean
    public XiaoMing xiaoMing(WashingMachine washingMachine){
        return new XiaoMing(washingMachine);
    }
}
```

## 4.2 在XML中引入JavaConfig配置

&emsp;我们在JavaConfig中声明Machine的实例，然后在XML中引入JavaConfig的配置。XML中的代码如下所示，直接在<bean>标签中声明一个Bean，class属性中指定JavaConfig的路径就可以了

```
@Configuration
@ImportResource("classpath:ImportXMLToJavaConfig.xml")
public class ImportXMLToJavaConfig {
    @Bean
    public XiaoMing xiaoMing(WashingMachine washingMachine){
        return new XiaoMing(washingMachine);
    }
}
```

参考代码在part1/MixedConfig路径中  https://github.com/xiaozhuyaoye/SpringInActionDemos