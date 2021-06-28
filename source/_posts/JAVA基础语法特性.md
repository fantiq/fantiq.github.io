---
title: JAVA基础语法特性
tags: [JAVA]
categories: [JAVA]
date: 2021-05-31 20:45:22
---

总结下JAVA语法方面的特点，JAVA在强类型的前提下实现了较为丰富的语法表达能力，这两点给软件开发提供了强大的工程能力。

<!-- more -->

## 基本数据类型

#### 接口 / 注解

###### 接口的定义

```
interface MyInterface {
    int attr = 0; // 接口里定义的属性 默认是 public static
    void method();
}
```

接口中定义的属性默认是 **public static**

接口定义的方法只能是 **抽象** **非静态** 方法

接口中不能定义静态代码块

###### 注解定义

```
@interface MyAnno {
    int attr() default 0;
}
```

注解是一种特殊的接口，注解在编译后其实其本质还是接口，注解是对定义类信息的增强，具体使用方式下面总结

#### 类 / 抽象类

###### 类的定义如下 

```
class MyClass {
    public final static int id = 0;
    public static void method(int code) {}
}
```

类的主要组成部分有 **属性** 和 **方法**，他们共有的有 `类型` `修饰` 以及方法有的`参数列表`，其中修饰部分包括 `访问权限` `以及`性质`。

对于其中的修饰 `static` 与 `final` ，`static` 代表静态，被`static` 修饰的 类、属性、方法 其性质会发生变化，他会脱离定义的类，不依赖类而存在的，可以参考c语言中的`static`，编译后运行的二进制文件中这些被静态修饰的数据会放在初始化区域的。对于`final` 就是**最终**的意思，也就是不能被改变了，对于属性就是其中的值是不能被改变的，包括饮用的对象，对于类和方法，其涉及到的改变可能是类的继承以及方法的覆盖，`final` 修饰后这些也是不能改变的了。

抽象类相对简单，类中存在一个抽象方法（没有方法题）这个类就是抽象类 需要用 `abstract`来修饰

###### 内部类

内部类就是在类的内部再次定义类，如果用`static`修饰的话就是`静态内部类` 。内部类需要依赖宿主类而存在，若要实例化内部类，需要先实例化宿主类，内部类可以访问宿主类中的所有属性和方法，宿主类却不行。访问类中的内容是需要实例化的，内部类访问宿主类却可以是因为内部类实例化需要依赖宿主类的实例化，也就是说当内部类实例化了宿主类一定实例化了。对于静态内部类可以按照静态的思路来看待，声明为静态后他就是一个独立的类定义，与宿主类并无关系。

#### 枚举

```
enum MyEnum {
    FOO(1, "foo"),
    BAR(2, "bar"),
    ;
    private final int code;
    private final String desc;
    MyEnum(int code, String desc) {
        this.code = code;
        this.desc = desc;
        // MyEnum.values() 遍历枚举
        // MyEnum.valueOf("") 通过枚举字符串查询枚举
    }
}
```

## 面向对象特性

#### 访问权限

访问权限表格如下

| --        | 类内部 | 继承子类 | 同包名类 | 其他 |
| --------- | ------ | -------- | -------- | ---- |
| public    | Y      | Y        | Y        | Y    |
| default   | Y      | Y        | Y        | N    |
| protected | Y      | Y        | N        | N    |
| private   | Y      | N        | N        | N    |



#### 方法的重载与重写

重写 `@Override` 是子类可以重写父类的方法，完全将父类的方法覆盖调

```
class Parent {
   void method() {
       return 1;
   }
}
class Child extends Parent {
    // 子类重写父类的方法
    @Override
    void method() {
        super.method(); // 子类调用父类方法
        return 2;
    }
}
```

重载 `@Overload` 是指可以定义同名不同参的方法，返回类型可相同可不同，不同参数列表可以是数量不同也可以是参数类型不同，距离如下：

```
class MyClass {
    void method(int id) {}
    // 参数类型不同
    id method(String id) {}
    // 参数数量不同
    String method(String id, int other) {}
}
```



#### 泛型

泛型是将类型定义后推，泛型可以用在 `接口` `类` 和 `方法`上面

```
// 接口定义
interface MyInterface<T> {
    T method();
}

// 类定义
class MyClass<T> {
    T method();
    // 方法中定义
    public<D> void run(D arg) {}
}
```

泛型中的类型擦除是指代码被编译成字节的时候泛型中指定的类型会被擦除（删除），这带来的问题是在反射的时候无法得到这个类型，当然这样做是为了增加一定的自由，比如如下的泛型使用：

```
List list = new ArrayList<Integer>();
```



## 注解与反射

#### 注解

注解的使用分为三个部分 `定义` `使用` `读取`

###### 注解定义

```
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@interface MyAnnotation {
    String[] value() default {};
}
```

其中的 `@Target(ElementType.METHOD)` 指定了注解可以使用的范围 ，这里定义是指定了只能用在方法上

其中的 `@Retention(RetentionPolicy.RUNTIME)` 指定了注解信息保留的阶段，可选值有 `SOURCE` `CLASS` `RUNTIME`。`SOURCE`指定注解信息只保留在源代码中，编译期就会丢弃注解信息。`CLASS`指定注解信息保留到编译生成的字节码，但是不会被加载到JVM中，这些注解信息在编译过程中是能够被读取到的。`RUNTIME`是将注解信息保留到在JVM中运行，这样代码里是能够通过反射读取到了。

###### 注解的使用

```
class MyClass {
    @MyAnnotation(value = {"a", "b", "c"})
    void method() {}
    
    public static void main(String[] args) {
        // 通过反射获取方法中的注解信息
        MyAnnotation myAnnotation = MyClassclass.getMethod("method").getAnnotation(MyAnnotation.class);
        // 读取注解中的信息
        myAnnotation.value();
    }
}
```

#### 反射

反射其实就是对字节码信息的封装，方便通过api读取信息，获取反射的方法可以通过类的全名字符串的形式`Class.forName("类的全名包括包名")`，也可以通过类名称的形式 `foo.bar.MyClass.class`，也可以从一个实例化对象中获取 `instance.getClass()`

## 动态代理

#### JDK

#### cglib



## JVM

#### 类加载

#### 内存分配

#### GC回收算法

[JVM系列](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzI4NDY5Mjc1Mg==&action=getalbum&album_id=1326602114365276164)