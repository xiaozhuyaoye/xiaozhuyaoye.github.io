---
layout: post
title: 读书笔记--Spring AOP
date:  2017-12-22
img:  /post4/logo.jpg
tags: [Java框架,Spring]
author:  Taylor Toby
---

# 一. 大纲

1 编写切点<br/>
2 通过注解和XML创建切面<br/>
3 通过注解和XML创建环绕通知<br/>
4 通过注解和XML处理通知中的参数<br/>

# 二. 编写切点

&emsp;Spring AOP中使用AspectJ的切点表达式来定义切点，因为Spring AOP是基于代理实现的，所以仅支持部分的AspectJ指示器，如下图：<br/>
<div align="center"><img src="../assets/img/post4/AspectJ Indicator.jpg" /></div>
&emsp;这当中只有execution指示器是实际执行匹配的，其他的指示器都是用来限制匹配的。详细可以参考http://blog.csdn.net/qwe6112071/article/details/50951720

# 三. 创建切面

## 3.1 通过注解创建简单的切面

```
package entity;            //entity包下的Machine接口，没什么用

public interface Machine {
    public void work();
}
-----------------------------------------------------------------------------------
package entity;           //entity包下的Machine接口的实现类WashingMachine

@Component                //自动注入
public class WashingMachine implements Machine {
    public void work() {
        System.out.println("washing clothes ...");
    }
}
-----------------------------------------------------------------------------------
package config;           //config包下的MachinePerformance

@Aspect                   //切面
@Component                //自动注入
public class MachinePerformance {

    @Before("execution(* *.*.*(..))")              //定义切点
    public void start(){
        System.out.println("机器开始工作");
    }

    @After("execution(* entity.*.*(..))")
    public void end(){
        System.out.println("机器结束工作");
    }

    @AfterReturning("execution(* entity.*.*(..))")
    public void endReturn(){
        System.out.println("机器结束工作(成功)");
    }

    @AfterThrowing("execution(* entity.*.*(..))")
    public void endExcept(){
        System.out.println("机器结束工作(失败)");
    }
}
-----------------------------------------------------------------------------------
package config;             //config包下的Part3Config类

@Configuration              //配置类
@EnableAspectJAutoProxy     //启动自动代理
@ComponentScan(basePackageClasses = {Machine.class ,MachinePerformance.class})  //自动扫描路径
public class Part3Config {

}
-----------------------------------------------------------------------------------
@RunWith(SpringJUnit4ClassRunner.class)                    //测试类
@ContextConfiguration(classes = Part3Config.class)
public class Part3Test {
    @Autowired
    private Machine washingMachine;

    @Test
    public void test1(){
        washingMachine.work();
    }
}
```

## 3.2 使用XML创建简单的切面

```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:aop="http://www.springframework.org/schema/aop"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd http://www.springframework.org/schema/aop http://www.springframework.org/schema/aop/spring-aop.xsd">

    <bean class="config.PointCutXml" id="pointCutXml"></bean>   //切面bean

    <bean class="entity.WashingMachine" id="washingMachine"></bean>

    <aop:aspectj-autoproxy></aop:aspectj-autoproxy>  //启用代理
    <aop:config>
        <aop:aspect ref="pointCutXml">
            <aop:before method="start" pointcut="execution(* entity.WashingMachine.*(..))"></aop:before>
            <aop:after method="end" pointcut="execution(* entity.WashingMachine.*(..))"></aop:after>
        </aop:aspect>
    </aop:config>
</beans>

@ContextConfiguration          //测试类
@RunWith(SpringJUnit4ClassRunner.class)
public class Part3Test2 {
    @Autowired
    private WashingMachine washingMachine;

    @Test
    public void test1(){
        washingMachine.work();
    }
}
```

## 3.3 使用注解提取重复的切点定义

```
public class MachinePerformance {
    @Pointcut("execution(* entity.*.*(..))")       //定义一个空方法，使用@Pointcut()注解
    public void myPointCut(){}

    @Before("myPointCut()")
    public void start(){
        System.out.println("机器开始工作");
    }

    @After("myPointCut()")
    public void end(){
        System.out.println("机器结束工作");
    }

    @AfterReturning("execution(* entity.*.*(..))")
    public void endReturn(){
        System.out.println("机器结束工作(成功)");
    }
}
```

## 3.4 使用XML提取重复的切点

```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:aop="http://www.springframework.org/schema/aop"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd http://www.springframework.org/schema/aop http://www.springframework.org/schema/aop/spring-aop.xsd">

    <bean class="config.PointCutXml" id="pointCutXml" />

    <bean class="entity.WashingMachine" id="washingMachine" />

    <aop:aspectj-autoproxy />
    <aop:config>
        <aop:aspect ref="pointCutXml">
            <aop:pointcut id="point1" expression="execution(* entity.WashingMachine.*(..))" />
            <aop:before method="start" pointcut-ref="point1" />
            <aop:after method="end" pointcut-ref="point1" />
        </aop:aspect>
    </aop:config>
</beans>
```

# 四. 创建环绕通知

## 4.1 使用注解创建环绕通知

```
package config;

@Aspect
@Component
public class MachineAroundPerformance {
    @Around("execution(* entity.WashingMachine.*(..))")
    public void around(ProceedingJoinPoint point){
        try {
            System.out.println("start work ...");
            long startTime = System.currentTimeMillis();
            point.proceed();
            System.out.println("end work ...");
            long endTime = System.currentTimeMillis();
            System.out.println("执行了"+ (endTime - startTime) + "毫秒");
        } catch (Throwable throwable) {
            throwable.printStackTrace();
            System.out.println("异常");
        }
    }
}
```

## 4.2 使用XML创建环绕通知

```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:aop="http://www.springframework.org/schema/aop"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd http://www.springframework.org/schema/aop http://www.springframework.org/schema/aop/spring-aop.xsd">

<bean class="config.PointCutXml" id="pointCutXml" />

<bean class="entity.WashingMachine" id="washingMachine" />

<aop:aspectj-autoproxy />
<aop:config>
    <aop:aspect ref="pointCutXml">
        <aop:pointcut id="point1" expression="execution(* entity.WashingMachine.*(..))" />
        <aop:before method="start" pointcut-ref="point1" />
        <aop:after method="end" pointcut-ref="point1" />
    </aop:aspect>

    <aop:aspect ref="pointCutXml">          //环绕通知
        <aop:around method="around" pointcut-ref="point1" />
    </aop:aspect>
</aop:config>
</beans>
```

# 五. 处理通知中的参数

## 5.1 使用注解处理通知的参数

```
package entity;    //新的实体，方法有入参

@Component
public class Printer {
    public void print(String info){
        System.out.println("working ... "+info);
    }
}

package config;        //新的切面

@Aspect
@Component
public class HandleParameter {  //表达式中入参指定类型，args指定参数名，位置数量要一致。

    @Before("execution(* entity.Printer.*(String)) && args(info)")
    public void methodWithParam(String info){
        System.out.println("入参："+info);
    }

}
```

## 5.2 使用XML处理通知中的参数

```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:aop="http://www.springframework.org/schema/aop"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd http://www.springframework.org/schema/aop http://www.springframework.org/schema/aop/spring-aop.xsd">

    <bean class="config.PointCutXml" id="pointCutXml" />
    <bean class="config.HandleParameter" id="handleParameter" />
    <bean class="entity.WashingMachine" id="washingMachine" />
    <bean class="entity.Printer" id="printer" />

<aop:aspectj-autoproxy />
<aop:config>
    <aop:aspect ref="handleParameter">
        <aop:before method="methodWithParam" pointcut="execution(* entity.Printer.*(String)) and args(info)"/>
    </aop:aspect>
</aop:config>
</beans>
```

其他内容在https://github.com/xiaozhuyaoye/SpringInActionDemos/tree/master/Part3 中