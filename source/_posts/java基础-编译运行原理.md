---
title: java基础-编译运行原理
tags: [java,编译原理]
categories: [java]
date: 2019-08-13 14:55:46
---
java的运行过程与c程序的运行过程非常相似，c程序由编译器将c源代码直接编译成了机器码（当然中间还有预处理 汇编的步骤），可以在操作系统中直接运行，java多了个中间字节码。java编译器将java源代码编译成字节码(xxx.class) 字节码是二进制文件，其格式类似于 c中的elf格式，elf格式是由操作系统识别的，java字节码是给 java的运行环境 jvm 识别的。 jvm 实现了字节码的运行环境，这样不依赖系统运行环境来实现跨平台，其实现形式类似操作系统的实现形式，都有程序运行需要的 指令存储、栈、堆 等内存段，jvm以多线程的模式运行程序（字节码），线程的调度依赖操作系统的调度算法，对硬件的操作还是要依赖操作系统提供的接口实现。
<!-- more -->

## 1. 相关指令
主要指令如下：

|指令名称|作用|
|--|--|
|javac|将java源代码编译成字节码|
|javap|反编译字节码|
|java|将字节码加载到jvm中运行|

## 1 编译过程

#### 1.1 基本编译过程
编译过程就是将 `.java` 源码编译成 `.class` 字节码，这个过程可以通过软件工具 `javac` 来实现，一个例子如下：创建源文件 `Stub.java`  内部定义一个类 `class Stub{}`，然后编译这个文件内容 执行命令 `javac Stub.java` 最后得到类文件 `Stub.class`。
以上是java源码编译的一个简单例子，这个定义中没有 `main` 方法，是不能使用 `java` 命令执行的。
需要注意的几个点：
1. 源代码文件名的后缀必须是 `.java` 
2. 文件名不用必与内部定义的类名称一致
3. 文件中可以定义多个类 编译后内部定义的class会分别生成 `.class` 文件
以上可以看出，编译器对文件后缀有要求，对文件名称 内部定义的类 没有具体要求，编译器读取文件内容进行解析 针对每个类生成对应的字节文件 文件名称是定义的类名，文件后嘴使用 `.class`

命令 `javac` 可以使用参数 `-verbose` 看到具体的编译过程：
```
# javac -verbose Stub.java
解析开始时间 RegularFileObject[Stub.java]]
[解析已完成, 用时 16 毫秒]
[源文件的搜索路径: .]
[类文件的搜索路径: xxxx/xxxx/xx.jar,xxxx,.]
[正在加载ZipFileIndexFileObject[/Library/Java/JavaVirtualMachines/jdk1.8.0_192.jdk/Contents/Home/lib/ct.sym(META-INF/sym/rt.jar/jaang/Object.class)]]
[正在检查Stub]
[正在加载ZipFileIndexFileObject[/Library/Java/JavaVirtualMachines/jdk1.8.0_192.jdk/Contents/Home/lib/ct.sym(META-INF/sym/rt.jar/jao/Serializable.class)]]
[正在加载ZipFileIndexFileObject[/Library/Java/JavaVirtualMachines/jdk1.8.0_192.jdk/Contents/Home/lib/ct.sym(META-INF/sym/rt.jar/jaang/AutoCloseable.class)]]
[已写入RegularFileObject[Stub.class]]
[共 176 毫秒]
```


#### 1.2 包package 对编译结果的影响
先把上面的例子做下修改，给他加上 `main` 方法
```
class Stub{
    public static void main(String[] args) {
        System.out.println("running...");
    }
}
```
执行命令 `javac Stub.java` 得到字节文件 `Stub.class`，然后执行字节码文件 `java Stub` 得到结果 `running...` ，这个结果符合我们的预期。
对上面代码做进一步修改，给他加上包定义
```
package foo.bar.baz;
class Stub{
    public static void main(String[] args) {
        System.out.println("running...");
    }
}
```
执行命令 `javac Stub.java` 编译通过并得到字节码文件`Stub.class` ，这说明 package 的定义并不是必须要根java源代码文件的文件夹结构一致的，为了方便管理目前项目会将 package与源文件的文件夹结构保持一致，但这并不是编译器的强制要求。

然后尝试执行这个字节码文件 `java Stub` 结果报错 `错误: 找不到或无法加载主类 Stub`。分析这个错误能看出jvm并不能找到我们要他执行的类 `Stub`，这里有两点需要明确的
1. java 命令的参数是一个字符串而不是一个文件名或文件路径，这个字符串是需要jvm执行的类全名
2. 类全名=包名称+类名称 类名称其实是需要加上包名称的，没有包名称的类才是 类名称与类全名相等

以上可以分析出来 错误原因，`java` 命令的参数 是一个不存在的类 需要调整参数为 `java foo.bar.baz.Stub` ，再次执行发现还是找不到类 `错误: 找不到或无法加载主类 foo.bar.baz.Stub`。
虽然看起来这个错误跟上次的错误一样但是并不是相同的原因导致的错误：
当我们执行`java <类名称>` 的时候，jvm需要先去读取字节码文件，成功找到字节码文件将字节码文件内容读入并执行，执行的时候需要找到以字节码的文件格式定义的类，找到这个类的定义后才能进一步找到main方法作为指令执行地址。

第一次报错的情况是这样的，执行命令 `java Stub`，给jvm的类名称是 `Stub` ，在加载的时候 jvm是能够找到文件 `Stub.class`的，但是当jvm读取字节码的内容后 并不能找到类 `Stub` 的定义，因为在源代码中定义了包名称 ，这个时候字节码中类的定义是 `foo.bar.baz.Stub` 所以会报错 。

第二次报错的情况是这样的，执行命令的时候 参数给了正确的类名称，这个时候 jvm会根据这个类名称查找对应的字节码文件，其搜寻路径是 当前文件夹以及 classpath 定义的路径。这个类名称是`foo.bar.baz.Stub` jvm会在如下路径查找文件 `foo/bar/baz/Stub.class` ，在这个例子中自然是找不到的（找不到还会去 classpath 路径中查找），所以报错。

所以想要执行成功需要将字节码文件 `Stub.class` 放到正确的能被 jvm 加载的路径 `foo/bar/baz/Stub.class` 中，调整文件夹结构如下：
```
Stub.class
foo
    - bar
        - baz
            Stub.class
```
然后执行命令 `java foo.bar.baz.Stub` 得到了预期结果 `running...`

命令 `javac` 提供了参数 `-d` 指定生成的字节码存储的文件夹位置 并且 会以 jvm加载规则的文件组织形式生成字节码文件：
```
# mkdir bin
# javac -d bin Stub.java
# tree bin
bin
└── foo
    └── bar
        └── baz
            └── Stub.class
# java -cp bin foo.bar.baz.Stub
running...
```
这里运行的时候需要参数 `-cp` 来指定 `classpath` ，告诉jvm搜索类的路径。

#### 1.3 javac 常用参数

javac 主要参数说明

|参数|作用|
|--|--|
|-g|编译的时候生成调试信息，这些信息会存储在class文件中，对代码断点调试的时候需要用到这些信息|
|-verbose|打印出编译过程产生的日志信息|
|-classpath|编译文件存在依赖的时候需要遍历哪些路径寻找 class文件|
|-sourcepath|编译文件存在依赖的时候需要遍历哪些路径寻找 java文件|
|-d|生成到字节码文件存储在哪个文件夹里|
|-source|指定输入到编译器的源代码符合哪个版本|
|-target|指定编译器输出的字节码对应jvm的版本|
|-bootclasspath|指定引导类jar包路径|
|-extdirs|指定扩展类jar包路径|
|-processor|指定编译过程需要调用的 注释处理程序 (AbstractProcessor)类全名|
|-processorpath|注视处理程序字节文件存储的路径|

#### 1.4 将字节码文件打包成 jar
一般的字节码编译后会按照类名称的组织形式存储在文件夹中，这种形式在发布 部署的时候不方便，java中提供一种jar格式可以对字节码进行压缩打包，并且jvm支持这种格式 只需要提供参数 `-jar` 声明就可以了。

创建文件 `StubA.java` `StubB.java` 并使用包名称 `foo.bar.baz` ，创建存储字节码文件的文件夹 `mkdir bin`，编译文件 `javac -d bin StubB.java StubA.java`，此时可以执行 `java -cp bin foo.bar.baz.StubA` 运行字节码了，将字节码打包成jar包可以执行命令 `jar -cvf Demo.jar -C bin foo`

但是这样打包并不能通过 `java -jar Demo.jar` 来执行，是因为需要给jar包中创建配置文件 `MANIFEST.MF` ，其配置内容主要是配置项 `Main-Class` ， 用来指定需要执行的类名称，这里需要的配置项是 `Main-Class: foo.bar.baz.StubA`，通过命令 `jar -cvfm Demo.jar MANIFEST.MF -C bin com` 得到包 `Demo.jar` ，可以直接通过命令 `java -jar Demo.jar` 来执行了。
`jar` 命令参数说明

|参数|作用|
|--|--|
|-c|创建jar文件|
|-x|解压jar文件|
|-v|显示创建/解压的过程细节|
|-f|指定文件名称|
|-m|指定配置文件|
|-C|指定字节码的路径|

解压jar包可以执行 `jar -xvf xxx.jar` ，不过这个命令好像不能指定解压到的目录，不过jar本质上就是通过zip压缩得到的文件，可以通过 `unzip` 进行解压 然后通过参数 `-d` 指定解压到的目录。

这里需要注意的是参数 -C 用来指定字节码所在的目录，层级要到 跟 包名称对应的文件夹结构那一层，然后后面的参数用来指定 包层级路径，则这个层级下的所有字节文件都会打包到 jar文件中。

## 2. 执行过程

#### 2.1 常用命令
字节码的执行相对简单，可以使用命令 `java` 来启动jvm执行指定的类。java 命令的参数是类全名并不是文件名称，字节码文件的查询读取过程是 jvm 自动完成的，这点在 `1.2` 章节做了详细解释。
jvm在查询的过程中默认从当前文件夹开始搜索，也可以通过参数 `-cp` 或 `-classpath` 指定 jvm查询字节码的文件夹。
也可以通过参数 `-D` 传递参数到 jvm，在代码层面可以通过方法 `System.getProperty("name")` 获取值，在java中这个被称作系统(jvm)属性，举例如下：
```
class Stub{
    public static void main(String[] args) {
        System.out.println(System.getProperty("foo"));
    }
}
```
执行命令 `java -Dfoo=aa222bbb` 得到结果 `aa222bbb`

#### 2.2 JVM代理
jvm在执行的过程中提供了一系列的钩子，可以通过外部代码干预jvm的执行过程，这些外部代码的实现形式可以分为两种：
一种是 native 方法，通过c/c++代码实现，以动态链接库 （xxx.so）的形式提供给jvm，这种被称为 JVMTI（JVM Tool Interface）。
其中参数 `-agentlib` `-agentpath` 是用来配置动态链接库的。

另一种是通过java代码的形式实现，需要通过 `instrumentation` 中的接口实现，这个代码以jar包的形式 在执行`java`命令的时候 以参数`-javaagent:agent.jar=OptionsString` 的形式指定，具体用法举例如下：
需要先制作jar包，创建文件 `MyJvmAgent.java` 并提供方法 `public static void premain(String agentOpt, Instrumentation instrumentation)` 提供给jvm调用，内容如下
```
import java.lang.instrument.ClassFileTransformer;
import java.lang.instrument.IllegalClassFormatException;
import java.lang.instrument.Instrumentation;
import java.security.ProtectionDomain;

public class MyJvmAgent {
    public static void premain(String agentOpt, Instrumentation instrumentation) {
        instrumentation.addTransformer(new ClassFileTransformer() {
            @Override
            public byte[] transform(ClassLoader loader, String className, Class<?> classBeingRedefined, ProtectionDomain protectionDomain, byte[] classfileBuffer) throws IllegalClassFormatException {
                System.out.println("class:" + className);
                return null;
            }
        });
    }
}
```
方法中以匿名方法的形式实现了方法 `java.lang.instrument.Instrumentation.addTransformer()` 其中的参数意义如下：

|类型|作用|
|--|--|
|ClassLoader loader|加载类使用的 ClassLoader对象|
|String className|类名称|
|Class<?> classBeingRedefined|???|
|ProtectionDomain protectionDomain|???|
|byte[] classfileBuffer|类对应的字节码|

然后将其打包成jar包，需要配置文件 `MANIFEST.MF`
```
Premain-Class: MyJvmAgent

```
打包命令 :
```
javac MyJvmAgent.java # 编译成字节码
jar -cvfm MyJvmAgent.jar MANIFEST.MF MyJvmAgent.class MyJvmAgent\$1.class # 打包成jar
```
需要一个执行类 `Stub.java`
```
public class Stub {
    public static void main(String[] args) {
        System.out.println("method : main");
    }
}
```
运行
```
javac Stub.java # 编译对象
java -javaagent:MyJvmAgent.jar Stub # 指定代理程序执行
```

#### 2.3 运行资源参数设置

-X



## 3. 字节码分析
源代码经编译后生成的字节码是可以直接给jvm来执行的二进制文件，其内部以jvm规范的字节形式进行组织，可以使用命令 `javap` 对字节码进行分析（反编译），javap常用的参数如下：

|参数|作用|
|--|--|
|-v -verbose|输出反汇编后的详细信息|
|-c|对代码进行反编译|
|-p|显示所有类和成员|
|-l|输出行号 和 本地变量表|
|-cp|指定字节码文件的扫描路径|
|-bootclasspath|指定引导类字节码文件路径|

创建一个简单类进行反编译
```
package foo.bar.baz;
class Stub{ }
```
进行编译 反编译操作 `javac Stub.java && javap -v Stub` 得到如下结果
```
Classfile /workspace/tests/java/JavaStudy/tmp/Stub.class
  Last modified 2019-8-13; size 194 bytes
  MD5 checksum 3e33fabc04ec68cecb1e6b6784bb552f
  Compiled from "Stub.java"
class foo.bar.baz.Stub
  minor version: 0
  major version: 52
  flags: ACC_SUPER
Constant pool:
   #1 = Methodref          #3.#10         // java/lang/Object."<init>":()V
   #2 = Class              #11            // foo/bar/baz/Stub
   #3 = Class              #12            // java/lang/Object
   #4 = Utf8               <init>
   #5 = Utf8               ()V
   #6 = Utf8               Code
   #7 = Utf8               LineNumberTable
   #8 = Utf8               SourceFile
   #9 = Utf8               Stub.java
  #10 = NameAndType        #4:#5          // "<init>":()V
  #11 = Utf8               foo/bar/baz/Stub
  #12 = Utf8               java/lang/Object
{
  foo.bar.baz.Stub();
    descriptor: ()V
    flags:
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: return
      LineNumberTable:
        line 2: 0
}
SourceFile: "Stub.java"
```

这里面主要的部分是 
```
Constant pool:
    #1 = Methodref ......
```
和
```
{
    ......
}
```

第一部分被称作 常量池 (`Constant pool`) , 这部分其实类似于 elf中的符号表，是对代码中的一些可以看作常量的内容统一管理起来了。重要的是第二部分，这里的例子中能看出来是一个方法 `foo.bar.baz.Stub()` 这个方法其实就是编译器自动帮我们加进去的构造方法，在源代码中并没有定义。然后 `descriptor` 定义了参数以及返回值，`code` 部分是jvm能够识别的汇编指令，这个就是方法中的代码了。

使用一个丰富的数据进行查看：
```
package foo.bar.baz;
class Stub{
    protected String[] property;

    Stub() {
    }

    public int calc(int num1, int num2) {
        return num1 * num2;
    }
}
```

执行 `javac Stub.java && javap -v Stub` 重点看下代码去的内容
```
{
  protected java.lang.String[] property; // 属性定义 类似于c中的预处理
    descriptor: [Ljava/lang/String;      // 属性类型的描述方式
    flags: ACC_PROTECTED                 // 访问权限

  foo.bar.baz.Stub();                    // 自定义的构造方法
    descriptor: ()V
    flags:
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: return
      LineNumberTable:                  // 对应的源代码行数
        line 5: 0
        line 6: 4

  public int calc(int, int);                // 方法定义
    descriptor: (II)I                       // 参数 返回值类型
    flags: ACC_PUBLIC                       // 访问权限
    Code:
      stack=2, locals=2, args_size=2        // 对应指令
         0: iload_1
         1: iconst_2
         2: imul
         3: ireturn
      LineNumberTable:
        line 9: 0
}
```
几个关键标志：
`descriptor` 标明数据类型，若是对象类型则替换为对象类名称，若是基本类型则有对应基本类型的标识 如下表：

|描述符|代码|
|--|--|
|B|byte|
|C|char|
|S|short|
|I|int|
|J|long|
|F|float|
|D|double|
|Z|boolean|
|V|void|
|L<...>;|对象|
|[<...>|数组|

其中对象的表示方式 `L<...>;` 其 `<...>` 代表对象的类名称，比如参数类型是 `java.lang.String` 则其表示形式是 `Ljava/lang/String;`
其中数组的表示方式 `[<...>` 其 `<...>` 代表类型描述符，必须源代码定义形式 `int[]` 对应的表示形式是 `[I` ，对象 `foo.bar.Baz[]` 的表示形式是 `[Lfoo/bar/Baz;`

若是属性 其 `descriptor` 只需要标明属性的类型。
若是方法 其 `descriptor` 需要标明 参数类型和返回类型，其形式如下 `(参数类型...)返回类型`。

`flags` 标明数据的修饰情况，其与源码的对应形式如下表：

|标志名|源码名|说明|
|--|--|--|
|ACC_PUBLIC|public|访问权限|
|ACC_PROTECTED|protected|访问权限|
|ACC_PRIVATE|private|访问权限|
|ACC_FINAL|final|常量|
|ACC_STATIC|static|静态|
|ACC_INTERFACE|interface|接口|
|ACC_ABSTRACT|abstract|抽象|
|ACC_ENUM|enum|枚举|
|ACC_ANNOTATION|annotation|注解|
|ACC_SUPER||兼容|
|ACC_SYNTHETIC||编译器产生|

`Code` 标明方法中的代码部分，其主要内容是jvm标准的汇编指令，其中也标明了函数运行所需要的栈信息

|||
|--|--|
|stack|运行这段代码所需栈大小|
|locals|临时变量所需内存|
|args_size|函数参数个数 若是非静态方法 jvm还会默认添加参数 this进来|

编译器其实会做很多优化工作，比如定义一个 privet 属性，但是这个属性在内部并没有被使用，编译器会将这个属性移除。

## 4. jvm内存布局

























