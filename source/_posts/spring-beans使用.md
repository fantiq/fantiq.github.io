---
title: spring-beans使用
tags: [spring,java,beans]
categories: [spring]
date: 2019-08-06 16:32:22
---
在spring框架中其核心功能是IoC的实现，Spring实现IoC靠的是组件 `spring-beans` ，搞清楚 beans 的原理有利于理解 spring 框架的设计思路。
Beans是对类如何实例化的定义，这些定义是通过配置的形式提现的，spring目前支持的定义格式有 `Properties` `Groovy` `XML` 以及后来在组件 `spring-context` 中通过 `注解` 的实现形式。容器的本质是一个HashMap 用于存储对象实例化后的地址 或者 存储如何实例化对象的数据(配置)。

<!-- more -->
## 1 spring-beans 整体概览
#### 1.1 容器的基本信息

在 `spring-beans` 组件中作为容器的类是 `org.springframework.beans.factory.support.DefaultListableBeanFactory`, 其提供方法 `public void registerBeanDefinition(String beanName, BeanDefinition beanDefinition) throws BeanDefinitionStoreException` 用来注册对象信息到容器，提供方法 `public Object getBean(String name) throws BeansException` 读取对象（这个方法是其从父类中继承过来的）。

容器中的HashMap定义如下：
```
private final Map<String, BeanDefinition> beanDefinitionMap = new ConcurrentHashMap(256);
```

组件 `spring-beans` 提供了对应的配置文件的解析类完成容器中的注册过程，这些类都继承了 `org.springframework.beans.factory.support.AbstractBeanDefinitionReader`。他们通过解析队形格式的配置文件 获取到实例化类过程的定义，并将这些数据包装成 `BeanDefine` 对象，最终通过方法 `registerBeanDefinition` 存储到容器的 HashMap中。
如下是这些类的清单：

|类|作用|
|--|--|
|org.springframework.beans.factory.xml.XmlBeanDefinitionReader|解析xml格式的配置 |
|org.springframework.beans.factory.groovy.GroovyBeanDefinitionReader|groovy格式的配置形式|
|org.springframework.beans.factory.support.PropertiesBeanDefinitionReader|properties格式的配置形式|

#### 1.2 一个简单的容器注入以及获取实例化对象的例子

需要实例化的对象
```
# java/foo/bar/BeanStub.java
package foo.bar;

class BeanStub{
    // ...
}
```

实例化过程的配置
```
# resources/beans.xml
<beans>
    <bean id="beanStub" class="foo.bar.BeanStub" />
</beans>
```

注册以及实例化过程
```
# java/foo/bar/Client.java

import org.springframework.beans.factory.support.DefaultListableBeanFactory;
import org.springframework.beans.factory.xml.XmlBeanDefinitionReader;
import org.springframework.core.io.ClassPathResource;
import org.springframework.core.io.Resource;

public static void main(String[] args) {
    // 实例化容器
    DefaultListableBeanFactory factory = new DefaultListableBeanFactory();
    // 实例化XML格式配置文件解析对象
    XmlBeanDefinitionReader reader = new XmlBeanDefinitionReader(factory);
    // 加载配置文件并解析 最终注册到容器
    Resource resource = new ClassPathResource("beans.xml");
    reader.loadBeanDefinitions(resource);

    // 通过id从容器中获取实例化对象
    System.out.println(factory.getBean("beanStub").getClass().getName());
}
---------- result ----------
foo.bar.BeanStub
```

#### 1.3 关于配置文件
在使用组件 `spring-beans` 的过程主要就是配置文件的编写，配置文件是用来描述 类被容器实例化的过程，这个过程的重点是类实例化成对象后其属性的值，所以配置文件的描述也是围绕着给对象属性赋值进行的，类实例化后成为对象，同一个类实例化的不同对象之间的区别是他们的属性值。
实例化过程的核心就是提供数据给容器，使其实例化的时候将设置的值设置到对象的属性上，配置文件的配置过程也是基于此展开的。

在面向对象的设计中，构造方法就是提供一个对外的接口给对象的属性进行初始化，其作为一个方法 内部提供了更多的可能对对象进行构造。在设计模式中对象的构造过程是通过 工厂模式来实现的，工厂模式其实是 构造方法的一个变体，其本质还是调用构造方法，工厂模式 给 调用构造方法之前的时机 提供了代码空间 其实可以理解为是对构造方法的一种扩充形式。

在JAVA中存在 JavaBean 的概念，也就是对象的每一个属性对应一个 getter/setter 方法，对对象属性的操作可以通过这些方法实现，相对于构造方法 这种形式从概念上更明确，函数体中的代码功能是单纯的对对应属性的读写。

以上，对象的构造过程其实是对对象属性值的复制过程，这个过程有两种形式，一个是通过 JavaBean 定义的 getter/setter 方法实现，一个是通过面相对象中提供的构造方法实现，构造方法又可以衍生出 工厂模式的 构造形式。

## 2 对象实例化的配置
这里使用XML的形式分析配置，配置的分析过程主要关注点在 功能以及提供的数据，xml形式的配置结构如下:
```
<beans>
    <bean ..../>
    <bean ....>
        <....>
    </bean>
</beans>
```
由于重点在 标签 `<bean>` 上面，下面的示例中只写标签 `<bean>` 。

对容器的操作代，整体的调用形式已在上面的代码例子中给出，下面的示例仅会用到 `factory.getBean("xxxx")` 的形式表示从容器中读取实例化对象，将重点放在验证逻辑上。

#### 2.1 基本配置形式

###### 2.1.1 必须属性 id name class
一个简单的配置形式如下:
```
<bean id="beanName" class="ClassName"/>
```
其中属性 `class` 表明了要实例化类的类全名(包括包名)，对于一个bean 这个属性是必须的，只有提供了这个属性值 容器才知道要实例化的类是哪个。

属性 `id` 和 `name` 都可以作为一个 bean 的标识 在获取bean的时候使用，他们的区别之处在与下面几条
1. 一个 bean 的 `id` 必须是唯一的 不可与其他 bean 重复，而 `name` 在同一个配置文件中也是不可重复的 在不同配置文件中允许重复 但是存在后面的定义覆盖前面的定义的情况
2. `name` 的值中可以使用 ` ` `:` `,` 字符分割值 并将这些分割后的值作为 bean 的别名，而 `id` 中不会有这样的处理 它的值只能作为一个整体
3. bean中可以不声明 `id` `name` 两个属性，这时候 bean 的 BeanName 取值是类全名

举例如下：
配置文件
```
<!-- 通过id定义 -->
<bean id="foo" class="xxx.xxxx.StubBean0"/>
<!-- 通过name定义 -->
<bean name="bar baz" class="xxx.xxxx.StubBean1"/>
<!-- 不定义 默认使用类的包全名 -->
<bean class="xxx.xxxx.StubBean2"/>
```
验证代码
```
String[] beanNames = new String[]{"foo", "bar", "baz", "xxx.xxxx.StubBean2"};
for(String beanName : beanNames) {
    System.out.println(beanName+"\t"+factory.getBean(beanName).getClass().getName());
}
---------- result ----------
xxx.xxxx.StubBean0
xxx.xxxx.StubBean1
xxx.xxxx.StubBean1
xxx.xxxx.StubBean2
```

###### 2.1.2 钩子方法 init-method destroy-method
注册完成的bean在实例化的时候 组件提供了两个钩子方法，属性 `init-method` 定义bean完成实例化后要调用的bean方法，属性 `destroy-method` 定义当 `spring-context` 中的 context 关闭的时候要调用的bean方法。
举例如下:
bean类定义
```
class StubBean0{
    public void initHook() {
        System.out.println("init hook");
    }
    public void destroyHook() {
        System.out.println("destroy hook");
    }
}
```
配置文件
```
<bean id="myBean" class="xxx.xxxx.StubBean0"
      init-method="initHook" destroy-method="destroyHook"
/>
```
验证代码
```
factory.getBean("myBean");
---------- result ----------
init hook
```

这里的验证代码没有使用 conext 所以 destory-method 定义的方法并没有执行
###### 2.1.3 对象构造形式 scope
scope 属性定义了 bean 在什么样的范围(生命周期)中需要实例化新的对象并返回，其去值见下表

|值|作用|
|--|--|
|singleton|默认值 单例模式|
|prototype|原型模式 每次返回的都是新对象|
|request|一次http请求周期中返回新对象|
|session||
|globalSession||
|application||
|websocket||

举例验证下 `singleton` `prototype` 这两个值
配置文件 
```
<bean id="myBean0" class="com.javastudy.framework.ioc.springbeans.StubBean0" scope="singleton"/>
<bean id="myBean1" class="com.javastudy.framework.ioc.springbeans.StubBean0" scope="prototype"/>
```
验证代码
```
System.out.println(factory.getBean("myBean0") == factory.getBean("myBean0"));
System.out.println(factory.getBean("myBean1") == factory.getBean("myBean1"));
---------- result ----------
true
false
```
#### 2.2 提供构造参数
###### 2.2.1 通过getter/setter实例化
对于符合JavaBean概念的类，在实例化的时候可以通过其 setter 方法给属性赋值，在配置文件中使用标签 `<property>`。标签 `<property>` 作为标签 `<bean>` 的子元素使用，其属性 `name` 对应的是类属性的名称, 其属性 `value` 对应的是属性的值。对于属性的值 也可以在 此标签的子元素 使用标签 `<value>` 声明，且标签 `<value>` 提供属性 `type` 用以说明值的类型， 如下例子是等价的:
```
<!-- 使用属性 value 声明值 -->
<property name="name" value="fantasy"/>

<!-- 使用标签value声明值 -->
<property name="name">
    <value>fantasy</value>
</property>

<!-- 使用标签value声明值 且使用属性 type 注明值的类型 -->
<property name="name">
    <value type="java.lang.String">fantasy</value>
</property>
```

以上声明形式只能用与 简单变量类型，如果是集合的类型则不适用，需要使用如下形式
```
<!-- HashMap -->
<property name="map">
    <map>
        <entry key="xx" value="xx"/>
        <entry key="yy" value="yy"/>
        <entry key="zz" value="zz"/>
    </map>
</property>

<!-- List -->
<property name="list">
    <list>
        <value>x</value>
        <value>y</value>
        <value>z</value>
    </list>
</property>

<!-- Set -->
<property name="set">
    <set>
        <value>x</value>
        <value>y</value>
        <value>z</value>
    </set>
</property>
```
其中的标签 `<map>` `<entry>` 中还可以使用属性 `key-type` `value-type` 类声明HashMap对应的 key value 的数据类型，这两个属性也同样可以应用到标签 `<list>` `<set>` 中

还有一些特殊的类型
如果你要给属性赋值为 `null` 应该这么写
```
<property name="name">
    <null/>
</property>
```
如果属性的类型是对象，这个对象就必须是一个bean，并用如下的形式声明
```
<bean id="foo" class="xxx.xxxx.Foo"/>
<bean id="bar" class="xxx.xxxx.Bar">
    <property name="pFoo">
        <ref bean="foo"/>
    </property>
    <!-- 或者写成如下形式 -->
    <property name="pFoo" ref="foo"/>
</bean>
```
关于属性依赖其他bean的情况 下面段落会 集中讲解。

举例如下
bean类定义
```
class StubBean0{
    int id;
    String name;
    HashMap<String, String> skill;
    Set<String> tags;

    public int getId() {
        return id;
    }

    public void setId(int id) {
        this.id = id;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public HashMap<String, String> getSkill() {
        return skill;
    }

    public void setSkill(HashMap<String, String> skill) {
        this.skill = skill;
    }

    public Set<String> getTags() {
        return tags;
    }

    public void setTags(Set<String> tags) {
        this.tags = tags;
    }
}
```

配置文件
```
<bean id="myBean0" class="com.javastudy.framework.ioc.springbeans.StubBean0">
    <property name="id" value="10012" />
    <property name="name" value="fantasy" />
    <property name="skill">
        <map>
            <entry key="php" value="10"/>
            <entry key="clang" value="8"/>
            <entry key="java" value="6"/>
            <entry key="golang" value="2"/>
        </map>
    </property>

    <property name="tags">
        <set>
            <value>PMP</value>
            <value>Develop</value>
            <value>Architect</value>
            <value>Expert</value>
        </set>
    </property>
</bean>
```

验证代码
```
StubBean0 stubBean0 = (StubBean0) factory.getBean("myBean0");
System.out.println(stubBean0.getId() + "\t" + stubBean0.getName());
System.out.println(stubBean0.getSkill());
System.out.println(stubBean0.getTags());
---------- result ----------
10012   fantasy
{php=10, clang=8, java=6, golang=2}
[PMP, Develop, Architect, Expert]
```
###### 2.2.2 继承属性值
在配置文件中定义的是一个个的bean 这些bean中定义了属性的初始值，`spring-beans` 提供了一种继承这些属性值声明的方法，方便我们在定义bean的时候能够复用一些属性值的定义。可以使用标签 `<bean>` 中的属性 `parent` 其值就是需要继承属性定义的bean的name。

用法举例
bean类定义：
```
class StubBean1{
    int id;

    public int getId() {
        return id;
    }

    public void setId(int id) {
        this.id = id;
    }
}
class StubBean2{
    int id;
    String name;

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public int getId() {
        return id;
    }

    public void setId(int id) {
        this.id = id;
    }
}
```

配置文件定义：
```
<bean id="parent" class="xxx.xxxx.StubBean1">
    <property name="id" value="20"/>
</bean>

<bean id="children" class="xxx.xxxx.StubBean2" parent="parent">
    <property name="name" value="fantiq"/>
</bean>
```

验证代码：
```
StubBean2 stubBean2 = (StubBean2) factory.getBean("children");
System.out.println(stubBean2.getName() + "\t" + stubBean2.getId());
---------- result ----------
fantiq  20
```
以上例子是在 bean `children` 中的属性的定义 继承了 bean `parent` 中的属性定义，如果属性有冲突 则会用继承bean的属性值覆盖被继承bean的属性值。这个例子中的父bean对应的是一个实际的类，也可以使用一个不对应类的bean定义 此时这个bean 需要声明成抽象的 `abstract="true"` ，不可以通过 getBean实例化这个bean，其配置文件可以写成下面样子:
```
<bean id="parent" abstract="true">
    <property name="id" value="220"/>
</bean>

<bean id="children" class="xxx.xxxx..StubBean2" parent="parent">
    <property name="name" value="fantiq"/>
</bean>
```
抽象的 bean 不能被实例化 自然不用定义`class` 属性，抽象bean的定义主要就是建立一个 属性初始值定义的模版。


###### 2.2.3 通过构造方法实例化
构造方法创建对象的方式与通过 setter方法创建的形式很相似，只不过构造方法使用标签 `<constructor-arg>` 声明参数，构造方法的声明形式与属性的声明形式是一样的。构造方法的形式是将定义的这些值传递给构造方法，由构造方法内部代码决定如何处理这些值。

###### 2.2.4 通过工厂模式实例化
工厂模式是构造方法的一个变种，工厂模式又可以分为两种形式
一种是基于当前bean的方法创建对象，方法必须是静态的 因为方法要在对象创建前调用。通过属性 `factory-method` 定义实例化bean使用的静态方法，配置形式如下:
```
<bean id="main" class="xxx.xxxx.Stub" factory-method="selfFactoryMethod"/>
```
另一种是调用其他bean中的方法，此时方法没有要求必须是静态，因为其他的bean需要先实例化的。使用属性 `factory-bean` 定义实例化bean要使用的已经实例化的BeanName，使用属性 `factory-method` 定义对应 BeanName中的方法名字, 配置形式如下:
```
<bean id="factory" class="xxx.xxxx.StubFactory" />
<bean id="main" class="xxx.xxxx.Stub" factory-bean="factory" factory-method="thirdFactoryMethod"/>

```

工厂方法不用必须返回指定类型的对象，他可以返回任何类型 但不能是 void。

也可以给工厂方法传递参数，配置形式如同构造方法的配置形式。


bean 类定义
```
// bean 内部的静态工厂方法
class StubBean1{
    public static boolean factory(String name) {
        System.out.println("self static factory method get args: " + name);
        return true;
    }
}
// 外部工厂方法
class StubBean2{
    public boolean factoryThird(String name) {
        System.out.println("third factory method get args:" + name);
        return true;
    }
}
```

配置文件
```
<bean id="myBeans1" class="xxx.xxxx.StubBean1" factory-method="factory">
    <constructor-arg name="name" value="foo"/>
</bean>

<bean id="factoryClass" class="xxx.xxxx.StubBean2"/>
        <bean id="myBeans2" class="xxx.xxxx.StubBean1" factory-bean="factoryClass" factory-method="factoryThird">
    <constructor-arg name="name" value="bar"/>
</bean>
```

验证代码：
```
factory.getBean("myBeans1");
factory.getBean("myBeans2");
---------- result ----------
self static factory method get args: foo
third factory method get args:bar
```

#### 2.3 依赖关系
###### 2.3.1 依赖的声明形式 ref
上面讲了两种主要的给类对象添加属性的方式，一种是通过 getter/setter 方法 一种是通过构造方法。在定义属性值的时候存在多种情况，如果是基本的数据类型的话，比如 `Integer` `String` `Boolean` `int` `double` `boolean` 这些基本类型可以直接通过 标签 `<property>` 或 `<constructor-arg>` 的属性 `name` `value` 实现值的定义。若是 `Map` `List` `Set` 这种复杂类型的值 则需要通过对应的标签 `<map><entry key="" value=""><....></map>` `<list><value>xxxxxx</value></list>` `<set><value>xxxxxx</value></set>` 定义值。

还有一种值是对象地址，也就是说 类A的属性值的类型是 类B，在组件 `spring-beans` 中所有的对象都被定义为一个 `bean` ，那么定义的类构造过程中 属性类型是类的话 也应该是一个bean。对于这样的定义 可以使用属性 `ref` 其值对应的是要依赖的对象的BeanName。

举例如下

bean类定义
```
class StubBean1{ }
class StubBean2{
    StubBean1 stubBean1;
    StubBean2(StubBean1 depend) {
        this.stubBean1 = depend;
    }
}
```

配置文件
```
<bean id="dependBean" class="xxx.xxxx.StubBean1"/>
<bean id="myBean" class="xxx.xxxx.StubBean2">
    <constructor-arg name="depend" ref="dependBean" />
</bean>
```

验证代码
```
StubBean2 stubBean2 = (StubBean2) factory.getBean("myBean");
System.out.println(stubBean2.stubBean1.getClass().getName());
---------- result ----------
xxx.xxxx.StubBean1
```

以上的例子是通过构造方法的形式进行依赖的，如果是属性的形式 做法也一样。

以上属性值类型是类的情况，定义形式由如下两种等价的方式
```
<constructor-arg name="depend" ref="dependBean" />

<constructor-arg name="depend">
    <ref bean="dependBean"/>
</constructor-arg>
```

###### 2.3.2 依赖的声明形式 depends-on
上一节说明了属性值是对象(bean) 情况该如何定义，这个在IoC概念中被称作 `依赖`，其实可以理解为是对属性值的定义。标签 `<bean>` 中存在属性 `depends-on` ，用以声明 bean 在被实例化的时候需要将 属性 `depends-on` 中列出来的 BeanNames 先实例化。

举例使用如下

bean类定义：
```
class StubBean1{
    StubBean1() {
        System.out.println("StubBean1");
    }
}
class StubBean2{
    StubBean2() {
        System.out.println("StubBean2");
    }
}

class StubBean3{
    StubBean3() {
        System.out.println("StubBean3");
    }
}
```

配置文件
```
<bean id="dependBean1" class="xxx.xxxx.StubBean1"/>
<bean id="dependBean2" class="xxx.xxxx.StubBean2"/>
<bean id="myBean" class="xxx.xxxx.StubBean3" depends-on="dependBean1 dependBean2" />
```

验证代码
```
factory.getBean("myBean");
---------- result ----------
StubBean1
StubBean2
StubBean3
```

`depends-on` 与 `ref` 的区别：
`ref` 是用于明确的定义 属性或构造方法参数 对应值要使用的 BeanName
`depends-on` 是用于说明 众多的 bean 实例化的先后顺序

###### 2.3.3 依赖自动判断 autowire
对于bean属性值类型是 类 的情况，可以通过属性 `ref` 显性的配置，还有一种属性 `autowire` 可以用于自动识别 bean 中属性值对应的应该是 容器中的哪个 bean。autowire 提供的值主要有两个 `byName` 和 `byType`，`byName` 通过要实例化的bean属性名字 与 容器中定义bean的 `name` `id` 进行匹配来实现自动的关联，`byType` 通过要实例化的bean属性定义的类型 与 容器中定义的beans对应类 进行匹配实现自动关联。

这个功能仅适用 实现了 getter/setter 方法的bean，不适用构造方法的形式。

举例如下

bean类定义
```
class StubBean1{ }
class StubBean2{ }

class StubBean3{
    StubBean1 dependBean1;
    StubBean2 dependBean2;

    public StubBean1 getDependBean1() {
        return dependBean1;
    }

    public void setDependBean1(StubBean1 dependBean1) {
        this.dependBean1 = dependBean1;
    }

    public StubBean2 getDependBean2() {
        return dependBean2;
    }

    public void setDependBean2(StubBean2 dependBean2) {
        this.dependBean2 = dependBean2;
    }
}
```

配置文件
```
<bean id="dependBean1" class="xxx.xxxx.StubBean1"/>
<bean id="dependBean2" class="xxx.xxxx.StubBean2"/>
<bean id="myBean" class="xxx.xxxx.StubBean3" autowire="byName"/>
```
验证代码
```
StubBean3 stubBean3 = (StubBean3) factory.getBean("myBean");
System.out.println(stubBean3.dependBean1.getClass().getName());
System.out.println(stubBean3.dependBean2.getClass().getName());
---------- result ----------
xxx.xxxx.StubBean1
xxx.xxxx.StubBean2
```
以上例子 定义的bean `<bean id="xx" class="xxx" autowire="byName"/>`，此时 bean定义类中属性变量名称与容器中定义的bean名称进行匹配。使用值 `byType` 则不必匹配BeanName与属性变量名称 只需要属性类型 与 容器中bean对应类的类型 进行匹配就行，上面例子中的配置文件改成如下形式也是可以的

```
<bean id="bean1" class="xxx.xxxx.StubBean1"/>
<bean id="bean2" class="xxx.xxxx.StubBean2"/>
<bean id="myBean" class="xxx.xxxx.StubBean3" autowire="byType"/>
```








参考链接:
> [Spring IoC容器](https://docs4dev.com/docs/zh/spring-framework/4.3.21.RELEASE/reference/beans.html)
