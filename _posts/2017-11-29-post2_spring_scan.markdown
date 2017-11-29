---
layout: post
title: Spring-->自动扫描
date:  2017-11-29
img:  /post2/spring.jpg
tags: [Java框架,Spring]
author:  Taylor Toby
---

# 先说点其他的

&emsp;最近看了一下React，还有Python，确实是打开了新世界的大门，尤其是Python可玩性很高，也利用相关的工具从豆瓣的小组上海租房中爬取了一些信息，给我自己找房子提供了一些参考。不过终究这些在短时间内是不能在项目中大量应用的，于是还是觉得开始重新看一些Java的框架。鉴于CDS二期项目开发到现在，三期更多的可能是性能或者结构上的优化，再或者CDS结束之后可能开始新项目吧，所以现在开始重新看看Spring的一些内容，并做一些记录。

# Spring如何发现Bean

## 目录结构
<img src="../assets/img/post2/jiegou1.jpg"/><br/>
首先Animal接口和Human接口都有一个方法eat()，Children实现Animal接口，Xiaoming实现Human接口，Config的代码如下，没有任何代码
```
package entiey;
import org.springframework.context.annotation.ComponentScan;

@ComponentScan()
public class Config {

}
```
完成这部分代码引入了spring-core、spring-beans和spring-context，为了完成测试引入了junit和spring-test，包之间的依赖关系如下图所示
<img src="../assets/img/post2/relation.jpg"/><br/>

```
package entiey;

import interfacePack.Animal;
import interfacePack.Human;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

@Component
public class Children implements Animal{
    @Autowired
    private Human ming;

    public void eat() {
        ming.eat();
    }
}
```
我们在Children中持有了Human的实例并且使用@Component注解让Spring自动装配

<img src="../assets/img/post2/test.jpg"/><br/>
测试用例中我们通过配置类Config类来获取上下文，获取了Animal和uman的实例，并且分别调用他们的eat()方法。执行test2()方法，控制台正确的输出内容。

这里Config类中没有任何代码，仅通过@ComponentScan()注解就让Spring发现了需要实例Bean，原因是因为@ComponentScan()没有入参时默认扫描Config类所在的包及其子包下面有@Component注解的类并且实例化他们

当然@ComponentScan()也可以有入参，@ComponentScan(basePackages = {"entity","interfacePack"})同样可以实现上面的功能，意思是在这两个路径下搜索带有@Component注解的类并实例化。

另外还可以使用@ComponentScan(basePackageClasses = {Xiaoming.class,Children.class})，通过指定类来扫描类所在的路径下的类。

XML中通过<context:component-scan />标签配置相应的路径