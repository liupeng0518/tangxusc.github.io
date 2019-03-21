---
title: "Lombok 原理分析与功能实现"
date: 2019-03-20T14:15:59+08:00
draft: false
categories:
- java
- Lombok
tags:
- java
- Lombok
keywords:
- java
- Lombok
---

> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 https://blog.mythsman.com/2017/12/19/1/

## 前言

这两天没什么重要的事情做，但是想着还要春招总觉得得学点什么才行，正巧想起来前几次面试的时候面试官总喜欢问一些框架的底层实现，但是我学东西比较倾向于用到啥学啥，因此在这些方面吃了很大的亏。而且其实很多框架也多而杂，代码起来费劲，无非就是几套设计模式套一套，用到的东西其实也就那么些，感觉没啥新意。刚这两天读” 深入理解 JVM” 的时候突然想起来有个叫 Lombok 的东西以前一直不能理解他的实现原理，现在正好趁着闲暇的时间研究研究。

## Lombok

### 代码

Lombok 是一个开源项目，源代码托管在 [GITHUB/rzwitserloot](https://github.com/rzwitserloot/lombok)，如果需要在 maven 里引用，只需要添加下依赖:

```
<dependency>
    <groupId>org.projectlombok</groupId>
    <artifactId>lombok</artifactId>
    <version>1.16.8</version>
</dependency>
```

### 功能

那么 Lombok 是做什么的呢？其实很简单，一个最简单的例子就是它能够实现通过添加注解，能够自动生成一些方法。比如这样的类:

```
@Getter
class Test{
    private String value;
}
```

我们用 Lombok 提供的 @Getter 来注解这个类，这个类在编译的时候就会变成:

```
class Test{
    private String value;
    public String getValue(){
        return this.value;
    }
}
```

当然 Lombok 也提供了很多其他的注解，这只是其中一个最典型的例子。其他的用法网上的资料已经很多了，这里就不啰嗦。
看上去是很方便的一个功能，尤其是在很多项目里有很多 bean，每次都要手写或自动生成 setter getter 方法，搞得代码很长而且没有啥意义，因此这个对简化代码的强迫症们还是很有吸引力的。
但是，我们发现这个包跟一般的包有很大区别，绝大多数 java 包都工作在运行时，比如 spring 提供的那种注解，通过在运行时用反射来实现业务逻辑。Lombok 这个东西工作却在编译期，在运行时是无法通过反射获取到这个注解的。
而且由于他相当于是在编译期对代码进行了修改，因此从直观上看，源代码甚至是语法有问题的。
一个更直接的体现就是，普通的包在引用之后一般的 IDE 都能够自动识别语法，但是 Lombok 的这些注解，一般的 IDE 都无法自动识别，比如我们上面的 Test 类，如果我们在其他地方这么调用了一下:

```
Test test=new Test();
test.getValue();
```

IDE 的自动语法检查就会报错，说找不到这个 getValue 方法。因此如果要使用 Lombok 的话还需要配合安装相应的插件，防止 IDE 的自动检查报错。
因此，可以说这个东西的设计初衷比较美好，但是用起来比较麻烦，而且破坏了代码的完整性，很多项目组 (包括我自己) 都不高兴用。但是他的实现原理却还是比较好玩的，随便搜了搜发现网上最多也只提到了他修改了抽象语法树，虽说从感性上可以理解，但是还是想自己手敲一敲真正去实现一下。

### 原理

翻了翻现有的资料，再加上自己的一些猜想，Lombok 的基本流程应该基本是这样：

*   定义编译期的注解
*   利用 JSR269 api(Pluggable Annotation Processing API) 创建编译期的注解处理器
*   利用 tools.jar 的 javac api 处理 AST(抽象语法树)
*   将功能注册进 jar 包

看起来还是比较简单的，但是不得不说坑也不少，搞了两天才把流程搞通。。。
下面就根据这个流程自己实现一个有类似功能的 Getter 类。

## 手撸 Getter

实验的目的是自定义一个针对类的 Getter 注解，它能够读取该类的成员方法并自动生成 getter 方法。

### 项目依赖

由于比较习惯用 maven，我这里就用 maven 构建一下项目，修改下当前的 pom.xml 文件如下：

```
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.mythsman.test</groupId>
    <artifactId>getter</artifactId>
    <version>1.0-SNAPSHOT</version>
    <packaging>jar</packaging>

    <name>test</name>

    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    </properties>

    <dependencies>

        <dependency>
            <groupId>com.sun</groupId>
            <artifactId>tools</artifactId>
            <version>1.8</version>
            <scope>system</scope>
            <systemPath>${java.home}/../lib/tools.jar</systemPath>
        </dependency>

    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.1</version>
                <configuration>
                    <source>1.8</source>
                    <target>1.8</target>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>
```

主要定义了下项目名，除了默认依赖的 junit 之外 (其实并没有用)，这里添加了 tools.jar 包。这个包实在 jdk 的 lib 下面，因此 scope 是 system，由于 ${java.home} 变量表示的是 jre 的位置，因此还要根据这个位置找到实际的 tools.jar 的路径并写在 systemPath 里。
由于防止在写代码的时候用到 java8 的一些语法，这里配置了下编译插件使其支持 java8。

### 创建 Getter 注解

定义注解 Getter.java:

```
package com.mythsman.test;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Target({ElementType.TYPE})
@Retention(RetentionPolicy.SOURCE)
public @interface Getter {
}
```

这里的 Target 我选择了 ElementType.TYPE 表示是对类的注解，Retention 选择了 RententionPolicy.SOURCE，表示这个注解只在编译期起作用，在运行时将不存在。这个比较简单，稍微复杂点的是对这个注解的处理机制。像 spring 那种注解是通过反射来获得注解对应的元素并实现业务逻辑，但是我们显然不希望在使用 Lombok 这种功能的时候还要编写其他的调用代码，况且用反射也获取不到编译期才存在的注解。
幸运的是 Java 早已支持了 JSR269 的规范，允许在编译时指定一个 processor 类来对编译阶段的注解进行干预，下面就来解决下这个处理器。

### 创建 Getter 注解的处理器

#### 基本框架

自定义的处理器需要继承 AbstractProcessor 这个类，基本的框架大体应当如下:

```
package com.mythsman.test;

import javax.annotation.processing.*;
import javax.lang.model.SourceVersion;
import javax.lang.model.element.TypeElement;
import java.util.Set;

@SupportedAnnotationTypes("com.mythsman.test.Getter")
@SupportedSourceVersion(SourceVersion.RELEASE_8)
public class GetterProcessor extends AbstractProcessor {

    @Override
    public synchronized void init(ProcessingEnvironment processingEnv) {
        super.init(processingEnv);
    }

    @Override
    public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment roundEnv) {
        return true;
    }
}
```

需要定义两个注解，一个表示该处理器需要处理的注解，另外一个表示该处理器支持的源码版本。然后需要着重实现两个方法，init 跟 process。init 的主要用途是通过 ProcessingEnvironment 来获取编译阶段的一些环境信息; process 主要是实现具体逻辑的地方，也就是对 AST 进行处理的地方。

具体怎么做呢？

#### init 方法

首先我们要重写下 init 方法，从环境里提取一些关键的类:

```
private Messager messager;
private JavacTrees trees;
private TreeMaker treeMaker;
private Names names;

@Override
public synchronized void init(ProcessingEnvironment processingEnv) {
    super.init(processingEnv);
    this.messager = processingEnv.getMessager();
    this.trees = JavacTrees.instance(processingEnv);
    Context context = ((JavacProcessingEnvironment) processingEnv).getContext();
    this.treeMaker = TreeMaker.instance(context);
    this.names = Names.instance(context);
}
```

我们提取了四个主要的类:

*   Messager 主要是用来在编译期打 log 用的
*   JavacTrees 提供了待处理的抽象语法树
*   TreeMaker 封装了创建 AST 节点的一些方法
*   Names 提供了创建标识符的方法

#### process 方法

process 方法的逻辑比较简单，但是由于这里的 api 对于我们来说比较陌生，因此写起来还是费了不少劲的：

```
@Override
public synchronized boolean process(Set<? extends TypeElement> annotations, RoundEnvironment roundEnv) {
    Set<? extends Element> set = roundEnv.getElementsAnnotatedWith(Getter.class);
    set.forEach(element -> {
        JCTree jcTree = trees.getTree(element);
        jcTree.accept(new TreeTranslator() {
            @Override
            public void visitClassDef(JCTree.JCClassDecl jcClassDecl) {
                List<JCTree.JCVariableDecl> jcVariableDeclList = List.nil();

                for (JCTree tree : jcClassDecl.defs) {
                    if (tree.getKind().equals(Tree.Kind.VARIABLE)) {
                        JCTree.JCVariableDecl jcVariableDecl = (JCTree.JCVariableDecl) tree;
                        jcVariableDeclList = jcVariableDeclList.append(jcVariableDecl);
                    }
                }

                jcVariableDeclList.forEach(jcVariableDecl -> {
                    messager.printMessage(Diagnostic.Kind.NOTE, jcVariableDecl.getName() + " has been processed");
                    jcClassDecl.defs = jcClassDecl.defs.prepend(makeGetterMethodDecl(jcVariableDecl));
                });
                super.visitClassDef(jcClassDecl);
            }

        });
    });

    return true;
}
```

步骤大概是下面这样：

1.  利用 roundEnv 的 getElementsAnnotatedWith 方法过滤出被 Getter 这个注解标记的类，并存入 set
2.  遍历这个 set 里的每一个元素，并生成 jCTree 这个语法树
3.  创建一个 TreeTranslator，并重写其中的 visitClassDef 方法，这个方法处理遍历语法树得到的类定义部分 jcClassDecl
    1.  创建一个 jcVariableDeclList 保存类的成员变量
    2.  遍历 jcTree 的所有成员 (包括成员变量和成员函数和构造函数)，过滤出其中的成员变量，并添加进 jcVariableDeclList
    3.  将 jcVariableDeclList 的所有变量转换成需要添加的 getter 方法，并添加进 jcClassDecl 的成员中
    4.  调用默认的遍历方法遍历处理后的 jcClassDecl
4.  利用上面的 TreeTranslator 去处理 jcTree

接下来再实现 makeGetterMethodDecl 方法:

```
private JCTree.JCMethodDecl makeGetterMethodDecl(JCTree.JCVariableDecl jcVariableDecl) {

    ListBuffer<JCTree.JCStatement> statements = new ListBuffer<>();
    statements.append(treeMaker.Return(treeMaker.Select(treeMaker.Ident(names.fromString("this")), jcVariableDecl.getName())));
    JCTree.JCBlock body = treeMaker.Block(0, statements.toList());
    return treeMaker.MethodDef(treeMaker.Modifiers(Flags.PUBLIC), getNewMethodName(jcVariableDecl.getName()), jcVariableDecl.vartype, List.nil(), List.nil(), List.nil(), body, null);
}

private Name getNewMethodName(Name name) {
    String s = name.toString();
    return names.fromString("get" + s.substring(0, 1).toUpperCase() + s.substring(1, name.length()));
}
```

逻辑就是读取变量的定义，并创建对应的 Getter 方法，并试图用驼峰命名法。

整体上难点还是集中在 api 的使用上，还有一些细微的注意点:
首先，messager 的 printMessage 方法在打印 log 的时候会自动过滤重复的 log 信息。
其次，这里的 list 并不是 java.util 里面的 list，而是一个自定义的 list，这个 list 的用法比较坑爹，他采用的是这样的方式:

```
package com.sun.tools.javac.util;

public class List<A> extends AbstractCollection<A> implements java.util.List<A> {
    public A head;
    public List<A> tail;
    
    //...
    
    List(A var1, List<A> var2) {
        this.tail = var2;
        this.head = var1;
    }
    
    public List<A> prepend(A var1) {
        return new List(var1, this);
    }
    
    public static <A> List<A> of(A var0) {
        return new List(var0, nil());
    }
    
    public List<A> append(A var1) {
        return of(var1).prependList(this);
    }
    
    public static <A> List<A> nil() {
        return EMPTY_LIST;
    }
    //...
}
```

挺有趣的，用这种叫 cons 而不是 list 的数据结构，添加元素的时候就把自己赋给自己的 tail, 新来的元素放进 head。不过需要注意的是这个东西不支持链式调用，prepend 之后还要将新值赋给自己。
而且这里在创建 getter 方法的时候还要把参数写全写对了，尤其是添加 this 指针的这种用法。

#### 测试类

上面基本就是所有功能代码了，接下来我们要写一个类来测试一下 (App.java)：

```
package com.mythsman.test;

@Getter
public class App {
    private String value;

    private String value2;

    public App(String value) {
        this.value = value;
    }

    public static void main(String[] args) {
        App app = new App("it works");
        System.out.println(app.getValue());
    }
}
```

不过，先不要急着构建，构建了肯定会失败，因为这原则上应该是两个项目。Getter.java 是注解类没问题，但是 GetterProcessor.java 是处理器，App.java 需要在编译期调用这个处理器，因此这两个东西是不能一起编译的，正确的编译方法应该是类似下面这样，写成 compile.sh 脚本就是：

```
#!/usr/bin/env bash

if [ -d classes ]; then
    rm -rf classes;
fi
mkdir classes

javac -cp $JAVA_HOME/lib/tools.jar com/mythsman/test/Getter* -d classes/

javac -cp classes -d classes -processor com.mythsman.test.GetterProcessor com/mythsman/test/App.java

javap -p classes com/mythsman/test/App.class

java -cp classes com.mythsman.test.App
```

其实是五个步骤:

1.  创建保存 class 文件的文件夹
2.  导入 tools.jar，编译 processor 并输出
3.  编译 App.java，并使用 javac 的 - processor 参数指定编译阶段的处理器 GetterProcessor
4.  用 javap 显示编译后的 App.class 文件 (非必须，方便看结果)
5.  执行测试类

好了，进入项目的根目录，当前的目录结构应该是这样的:

```
.
├── pom.xml
├── src
│   ├── main
│   │   ├── java
│   │   │   ├── com
│   │   │   │   └── mythsman
│   │   │   │       └── test
│   │   │   │           ├── App.java
│   │   │   │           ├── Getter.java
│   │   │   │           └── GetterProcessor.java
│   │   │   └── compile.sh
```

调用 compile.sh，输出如下:

```
Note: value has been processed
Note: value2 has been processed
Compiled from "App.java"
public class com.mythsman.test.App {
  private java.lang.String value;
  private java.lang.String value2;
  public java.lang.String getValue2();
  public java.lang.String getValue();
  public com.mythsman.test.App(java.lang.String);
  public static void main(java.lang.String[]);
}
it works
```

Note 行就是在 GetterProcessor 类里通过 messager 打印的 log，中间的是 javap 反编译的结果，最后一行表示测试调用成功。

#### Maven 构建并打包

上面的测试部分其实是为了测试而测试，其实这应当是两个项目，一个是 processor 项目，这个项目应当被打成一个 jar 包，供调用者使用；另一个项目是 app 项目，这个项目是专门使用 jar 包的，他并不希望添加任何额外编译参数，就跟 lombok 的用法一样。
简单来说，就是我们希望把 processor 打成一个包，并且在使用时不需要添加额外参数。
那么如何在调用的时候不用加参数呢，其实我们知道 java 在编译的时候会去资源文件夹下读一个 META-INF 文件夹，这个文件夹下面除了 MANIFEST.MF 文件之外，还可以添加一个 services 文件夹，我们可以在这个文件夹下创建一个文件，文件名是 javax.annotation.processing.Processor，文件内容是 com.mythsman.test.GetterProcessor。
我们知道 maven 在编译前会先拷贝资源文件夹，然后当他在编译时候发现了资源文件夹下的 META-INF/serivces 文件夹时，他就会读取里面的文件，并将文件名所代表的接口用文件内容表示的类来实现。这就相当于做了 - processor 参数该做的事了。
当然这个文件我们并不希望调用者去写，而是希望在 processor 项目里集成，调用的时候能直接继承 META-INF。

好了，我们先删除 App.java 和 compile.sh，添加下 META-INF 文件夹，当前目录结构应该是这样的：

```
.
├── pom.xml
├── src
│   └── main
│       ├── java
│       │   └── com
│       │       └── mythsman
│       │           └── test
│       │               ├── Getter.java
│       │               └── GetterProcessor.java
│       └── resources
│           └── META-INF
│               └── services
│                   └── javax.annotation.processing.Processor

```

当然，我们还不能编译，因为 processor 项目并不需要把自己添加为 processor(况且自己还没编译呢怎么调用自己)。。。完了，好像死循环了，自己在编译的时候不能添加 services 文件夹，但是又需要打的包里有 services 文件夹，这该怎么搞呢？
其实很简单，配置一下 maven 的插件就行，打开 pom.xml, 在 project/build / 标签里添加下面的配置:

```
<build>
   <resources>
        <resource>
            <directory>src/main/resources</directory>
            <excludes>
                <exclude>META-INF/**/*</exclude>
            </excludes>
        </resource>
    </resources>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-resources-plugin</artifactId>
            <version>2.6</version>
            <executions>
                <execution>
                    <id>process-META</id>
                    <phase>prepare-package</phase>
                    <goals>
                        <goal>copy-resources</goal>
                    </goals>
                    <configuration>
                        <outputDirectory>target/classes</outputDirectory>
                        <resources>
                            <resource>
                                <directory>${basedir}/src/main/resources/</directory>
                                <includes>
                                    <include>**/*</include>
                                </includes>
                            </resource>
                        </resources>
                    </configuration>
                </execution>
            </executions>
        </plugin>
        ...
    </plugins>
</build>
```

我们知道 maven 构建的第一步就是调用 maven-resources-plugin 插件的 resources 命令，将 resources 文件夹复制到 target/classes 中，那么我们配置一下 resources 标签，过滤掉 META-INF 文件夹，这样在编译的时候就不会找到 services 的配置了。然后我们在打包前 (prepare-package 生命周期) 再利用 maven-resources-plugin 插件的 copy-resources 命令把 services 文件夹重新拷贝过来不就好了么。
这样配置好了，就可以直接执行`mvn clean install`打包提交到本地私服:

```
myths@pc:~/Desktop/test$ mvn clean install
[INFO] Scanning for projects...
[INFO] 
[INFO] ------------------------------------------------------------------------
[INFO] Building test 1.0-SNAPSHOT
[INFO] ------------------------------------------------------------------------
[INFO] 
[INFO] --- maven-clean-plugin:2.5:clean (default-clean) @ getter ---
[INFO] 
[INFO] --- maven-resources-plugin:2.6:resources (default-resources) @ getter ---
[INFO] Using 'UTF-8' encoding to copy filtered resources.
[INFO] Copying 0 resource
[INFO] 
[INFO] --- maven-compiler-plugin:3.1:compile (default-compile) @ getter ---
[INFO] Changes detected - recompiling the module!
[INFO] Compiling 2 source files to /home/myths/Desktop/test/target/classes
[INFO] 
[INFO] --- maven-resources-plugin:2.6:testResources (default-testResources) @ getter ---
[INFO] Using 'UTF-8' encoding to copy filtered resources.
[INFO] skip non existing resourceDirectory /home/myths/Desktop/test/src/test/resources
[INFO] 
[INFO] --- maven-compiler-plugin:3.1:testCompile (default-testCompile) @ getter ---
[INFO] No sources to compile
[INFO] 
[INFO] --- maven-surefire-plugin:2.12.4:test (default-test) @ getter ---
[INFO] No tests to run.
[INFO] 
[INFO] --- maven-resources-plugin:2.6:copy-resources (process-META) @ getter ---
[INFO] Using 'UTF-8' encoding to copy filtered resources.
[INFO] Copying 1 resource
[INFO] 
[INFO] --- maven-jar-plugin:2.4:jar (default-jar) @ getter ---
[INFO] Building jar: /home/myths/Desktop/test/target/getter-1.0-SNAPSHOT.jar
[INFO] 
[INFO] --- maven-install-plugin:2.4:install (default-install) @ getter ---
[INFO] Installing /home/myths/Desktop/test/target/getter-1.0-SNAPSHOT.jar to /home/myths/.m2/repository/com/mythsman/test/getter/1.0-SNAPSHOT/getter-1.0-SNAPSHOT.jar
[INFO] Installing /home/myths/Desktop/test/pom.xml to /home/myths/.m2/repository/com/mythsman/test/getter/1.0-SNAPSHOT/getter-1.0-SNAPSHOT.pom
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time: 3.017 s
[INFO] Finished at: 2017-12-19T19:57:04+08:00
[INFO] Final Memory: 16M/201M
[INFO] ------------------------------------------------------------------------
```

可以看到这里的 process-META 作用生效。

#### 调用 jar 包测试

重新创建一个测试项目 app：

```
.
├── pom.xml
└── src
    └── main
        └── java
            └── com
                └── mythsman
                    └── test
                        └── App.java
```

pom.xml:

```
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.mythsman.test</groupId>
    <artifactId>app</artifactId>
    <version>1.0-SNAPSHOT</version>
    <packaging>jar</packaging>

    <name>main</name>
    <url>http://maven.apache.org</url>

    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    </properties>

    <dependencies>
        <dependency>
            <groupId>com.mythsman.test</groupId>
            <artifactId>getter</artifactId>
            <version>1.0-SNAPSHOT</version>
        </dependency>
    </dependencies>
    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.1</version>
                <configuration>
                    <source>1.8</source>
                    <target>1.8</target>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>
```

App.java:

```
package com.mythsman.test;

@Getter
public class App {
    private String value;

    private String value2;

    public App(String value) {
        this.value = value;
    }

    public static void main(String[] args) {
        App app = new App("it works");
        System.out.println(app.getValue());
    }
}
```

编译并执行:
```

mvn clean compile && java -cp target/classes com.mythsman.test.App
```

最后就会在构建成功后打印”it works”。

## 参考资料

[GITHUB/lombok](https://github.com/rzwitserloot/lombok)

[使用 lombok 简化 Java 代码](https://segmentfault.com/a/1190000005133786)

[Java 注解 (3)- 注解处理器](https://blog.zenfery.cc/archives/78.html)

[stackoverflow/How does lombok work?](https://stackoverflow.com/questions/6107197/how-does-lombok-work)

[【翻译】Javac 骇客指南](https://my.oschina.net/superpdm/blog/129715)

[Javac 早期 (编译期)](https://www.cnblogs.com/wade-luffy/p/6050331.html)

[利用 maven 中 resources 插件的 copy-resources 目标进行资源 copy 和过滤](http://xigua366.iteye.com/blog/2080668)
