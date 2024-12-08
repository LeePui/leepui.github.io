---
layout:     post
title:      "翻译：Embracing invokedynamic to tame class loaders in Java agents"
subtitle:   "通过 invokedynamic 驯服 Java 代理中的类加载器"
date:       2024-12-8 16:46:06
author:     "国道蛋黄派"
catalog: true
published: true
header-style: text
categories: 
  - elastic-apm
tags:
  - agent
  - invokedynamic
  - byte-buddy
  - elastic-apm
---

# 译者注
该文翻译自 [Embracing invokedynamic to tame class loaders in Java agents](https://www.elastic.co/cn/blog/embracing-invokedynamic-to-tame-class-loaders-in-java-agents)，原文发布于 2021年11月22日。作者是：[Felix Barnsteiner](https://www.elastic.co/cn/blog/author/felix-barnsteiner),[Rafael Winterhalter](https://www.elastic.co/cn/blog/author/rafael-winterhalter).  (分别是elastic apm java agent的作者以及byte-buddy的作者)  
这篇文章对本人在java agent开发中，类加载器相关问题解决上启发很大，所以决定翻译出来，供大家学习。也强烈推荐读者阅读原文以及Elastic APM Java Agent的相关[源码](https://github.com/elastic/apm-agent-java/blob/main/apm-agent-bootstrap/src/main/java/bootstrap/dispatcher/IndyBootstrapDispatcher.java)。  
同时该文章需要的前置知识较多，译者能力有限，翻译过程中可能存在理解偏差，仅供参考。  
建议读者在阅读本文前，先阅读 [Byte Buddy 基本使用方法](https://bytebuddy.net/#/tutorial) , [Java 字节码指令invokedynamic](https://docs.oracle.com/javase/specs/jvms/se17/html/jvms-6.html#jvms-6.5.invokedynamic) 以及类加载器相关知识。  
另外译者在后续空余时间，也会继续分享一些在开发java agent过程中的设计以及实现。
### 一些翻译约定：
- agent：代理

# 正文

Byte Buddy 的一大优点是，它允许你在不需要手动处理字节码的情况下编写 Java 代理。为了对方法进行插桩，代理作者只需用纯 Java 编写要注入的代码。这使得编写 Java 代理变得更加简便，并避免了复杂的入门要求。  
  
在第一次成功的实验之后，代理作者通常会遇到 JVM 带来的复杂性障碍：类加载器（OSGi，真是让人头疼！）、类的可见性、对内部 API 的依赖、类路径扫描器以及版本冲突等等。  
  
在本文中，我们将探讨一种较为新颖的方式，旨在突破这一复杂性的壁垒。该架构基于 invokedynamic 字节码指令，这种字节码最著名于其支持 Java Lambda 表达式的能力，它为插桩代码的编写提供了一个简洁的思维模型。额外的优势在于，这一方法还支持在运行时更新代理的版本，而无需重启被插桩的应用程序。Elastic APM Java 代理早在一年多前就已启动向[这种基于 invokedynamic 的架构迁移](https://github.com/elastic/apm-agent-java/issues/1337)，并且最近已经顺利完成了迁移工作。  

## 传统的 advice 调度方法存在的问题 (Issues with traditional advice dispatching approaches)
让我们考虑一个简单的例子：一个代理程序，它想要测量 Java servlet 的响应时间。在所谓的 advice 方法中，可以定义在实际方法执行前或执行后运行的代码。还可以访问被插桩方法的参数。  
```java
@Advice.OnMethodEnter
public static long enter() {
    return System.nanoTime();
}

@Advice.OnMethodExit
public static void exit(
        @Advice.Argument(0) HttpServletRequest request,
        @Advice.Enter long startTime) {
    System.out.printf(
            "Request to %s took %d ns%n",
            request.getRequestURI(),
            System.nanoTime() - startTime);
}
```
在 Byte Buddy 中，advice 应用到被插桩方法的方式主要有两种。
  
## 内联 advice (inlint advice)
默认情况下，enter 和 exit 的 advice 会被复制到目标方法中，就像原始类的作者将代理的代码直接添加到方法中一样。如果被插桩的方法是用普通的 Java 编写的，那么它看起来可能是这样的：
```java
protected void service(HttpServletRequest req, HttpServletResponse resp) {
    long startTime = System.nanoTime();
    // original method body
    System.out.printf(
            "Request to %s took %d ns%n",
            request.getRequestURI(),
            System.nanoTime() - startTime);
}
```
这种方式的优点在于，advice 可以访问所有在被插桩方法中通常可以访问的值或类型。在上述示例中，这使得能够访问 `javax.servlet.http.HttpServletRequest`，尽管代理本身并不直接包含这个接口。由于代理代码是在目标方法内运行的，它会直接使用该方法本身已能访问的类型定义。

但缺点是，advice 代码不再在其定义的上下文中执行。因此，举个例子，你无法在 advice 方法中设置断点，因为它实际上并没有被调用。记住：这些方法只是作为模板使用。

真正的问题在于，将代码提取到 advice 方法之外，或者调用任何通常可以从 advice 访问的方法，变得不再可能。由于所有代码现在都从被插桩方法执行，代理可能在一个与被插桩方法完全无关的类加载器中运行，因此即使是公共方法，也可能无法从被插桩代码中调用。我们将在下一节中进一步探讨这个问题。  

> 译者注：  
> 这里作者说明的是内联的advice方法存在的问题，即: 
> 1. advice 类的代码只能当做模板使用。因此，你无法在 advice 方法中设置断点，因为它实际上并没有被调用。对于调试来说，非常不方便。
> 2. 代理可能在一个与被插桩方法完全无关的类加载器中运行，因此agent应用里面的公共方法，可能无法从被插桩代码中调用。  
  
## 委托型 advice （Delegated advice）
对于一种相似但仍然非常不同的方法，可以指示 Byte Buddy 委托给 advice 方法。这可以通过 `@Advice.OnMethodEnter(inline = false)` 注解属性来控制。默认情况下，Byte Buddy 会通过静态方法调用委托到 advice 方法。被插桩的方法将会变成如下所示：
```java
protected void service(HttpServletRequest req, HttpServletResponse resp) {
    long startTime = AdviceClass.enter();
    // original method body
    AdviceClass.exit(req, startTime);
}
```
与之前类似，代理的开发者需要确保 advice 代码对被插桩方法可见。如果被插桩的方法与代理代码不共享类加载器层次结构，那么在调用上述方法时，插桩将抛出 `NoClassDefFoundError`。即便委托的 advice 能够从代理中访问，像 `HttpServletRequest` 这样的参数类型也可能无法被代理的类加载器访问。这将导致错误在调用代理的 advice 时转移到代理代码中。  
> 译者注：  
> 这里作者说明的是委托型advice方法存在的问题，即：  
> 1. 委托型advice方法，就是在被插桩方法中，调用advice类中的静态方法。 
> 2. 被插桩方法有两种情况能访问advice方法：  
> 2.1 使用同一个类加载器，那么就不会存在类加载器问题。  
> 2.2. advice方法是被插桩方法的类加载器的父加载器加载。  
  
## 类加载器问题 (ClassLoader issues)
默认情况下，代理在附加到 JVM 时会被添加到系统类加载器，而 `java.lang.instrument.Instrumentation`接口提供了将代理添加到引导类加载器的方法。理论上，将类添加到引导类加载器使得它们可以在任何地方可见。然而，一些类加载器（例如 OSGi）仅允许从系统类加载器或引导类加载器加载特定的类（如 `java.*`、`com.sun.*`）。一种常见的解决方案是对所有类加载器进行插桩，并显式地将某些包中的类的类加载重定向到引导类加载器。
> 译者注：代理通过-javaagent参数附加到JVM时，该路径由系统类加载器加载。  

但是，将类添加到系统类加载器和引导类加载器也带来了负面影响。额外的类可能会减慢类路径扫描器的速度，甚至导致应用程序启动失败。有关示例，请参见 [elastic/apm-agent-java#364](https://github.com/elastic/apm-agent-java/pull/364)。此外，对于这样的持久类加载器，无法卸载类，这在设计一个希望提供运行时自我卸载功能的代理时是一个问题。  


从概念上讲，当一个 advice 类想要调用通常由代理提供的不同方法，但这些方法可能无法直接访问时，解决这些类加载器问题的方式只有两种：第一种方法是将代码注入到被插桩类的类加载器中，以便可以直接从那里查找这些方法。第二种方法是定义一个新的类加载器，作为前一个类加载器的子类，在这个新的类加载器中实现自定义加载逻辑，从而使额外的类型能够被定位到。  
  

对于第一种方法，Byte Buddy 提供了可以将类注入到任意类加载器中的工具（`net.bytebuddy.dynamic.loading.ClassInjector`）。虽然这看起来是一个直接的解决方案，但它也带来了重大缺陷。更灵活的注入器建立在内部 API 之上，如 `sun.misc.Unsafe` / `jdk.internal.misc.Unsafe`。即便是看起来更安全的类注入策略，如 `UsingReflection`，也使用了一些巧妙的变通方法，绕过了最近 Java 版本中引入的安全机制，这些机制通常会禁止通过 `Unsafe::putBoolean` 访问私有字段。直到今天，Oracle 限制访问内部 API 并加强反射 API 可见性，和发现新漏洞以绕过这些限制之间的博弈仍在持续。同时，官方的通过方法句柄查找的方式几乎与代理不兼容，其集成仍是一个开放问题[https://bugs.openjdk.java.net/browse/JDK-8200559](https://bugs.openjdk.java.net/browse/JDK-8200559)。因此，基于目前 Oracle 旨在进一步封锁的、不安全的 API 来构建完整的代理架构，似乎相当冒险。  
  
第二种方法是将所有的 advice 和辅助类加载到一个子类加载器中。这种方式无需依赖不安全的 API，因为类加载器是由代理开发者实现的，并且一个类加载器可以访问其父类加载器定义的所有类型。  
  
将辅助类加载到专用类加载器中，而不是将其注入到被插桩类的类加载器中的另一个优势是，可以卸载这些类。这使得可以完全将代理从应用程序中分离，并在不留下任何之前版本痕迹的情况下附加新版本的代理，这也被称为代理的实时更新（live-updating）。Byte Buddy 已经允许通过重新转换（re-transformation）撤销它应用的所有插桩。当没有其他对代理辅助类加载器的引用泄漏时，这使得其所有对象、类，甚至整个类加载器都可以被垃圾回收。  
  
这种方法的一个复杂之处在于，advice 类对被插桩类不可见。前面例子中的 `HttpServlet::service` 方法通过静态方法调用 `AdviceClass`。这将在运行时导致 `NoClassDefFoundError`，因为 `AdviceClass` 在 `HttpServlet::service` 方法的上下文中是不可见的。原因在于 `AdviceClass` 是由被插桩类（`HttpServlet`）的子类加载器加载的。虽然 `AdviceClass` 可以访问被插桩类可见的类，例如 HttpServletRequest 参数，但反过来却无法访问被插桩类可见的内容。
> 译者注：这一段是核心，说出了agent的类加载器问题。下面开始就是如何通过`invokedynamic`来解决这种问题。  
  
## 引入基于 invokedynamic 的 advice 分发方法 (Introducing an invokedynamic-based advice dispatching approach)
除了通过静态方法调用分发 advice，还有一种较少被知晓的替代方法。通过 `net.bytebuddy.asm.Advice.WithCustomMapping::bootstrap`，你可以指示 Byte Buddy 在被插桩方法中插入一个 `invokedynamic` 字节码指令。这个指令是在 Java 7 中引入的，旨在更好地支持 JVM 中的动态语言，如 Groovy 和 JRuby。

简而言之，`invokedynamic` 调用分为两个阶段：首先查找 `CallSite`，然后调用 `CallSite` 持有的 MethodHandle。如果相同的 invokedynamic 指令再次执行，将会调用初始查找时得到的 `CallSite`。  
  
以下示例展示了一个 invokedynamic 指令在方法字节码中的表现。  
```java
// InvokeDynamic #1:exit:(Ljavax/servlet/ServletRequest;long)V</p> <p>invokedynamic #1076, 0
```  
`CallSite` 的查找发生在一个所谓的引导方法（bootstrap method）中。这个方法接收几个用于查找的参数，如 advice 类名、方法名以及表示参数和返回类型的 advice 的 `MethodType`。以下示例展示了引导方法在一个类的字节码中的声明方式。
```java
BootstrapMethods:
  1: #1060 REF_invokeStatic java/lang/IndyBootstrapDispatcher.bootstrap:(Ljava/lang/invoke/MethodHandles$Lookup;Ljava/lang/String;Ljava/lang/invoke/MethodType;[Ljava/lang/Object;)Ljava/lang/invoke/CallSite
    Method arguments:
      #1049 org.example.ServletAdvice
      #1050 1
      #12 javax/servlet/http/HttpServlet
      #1072 service
      #1075 REF_invokeVirtual javax/servlet/http/HttpServlet.service:(Ljavax/servlet/HttpServletRequest;Ljavax/servlet/HttpServletResponse;)V
```
包含引导方法的类（在此例中为 `java/lang/IndyBootstrapDispatcher.bootstrap`）必须对所有被插桩的类可见。因此，该类需要添加到引导类加载器中。为了确保与过滤类加载器（如 OSGi 加载器）的兼容性，该类被放置在 `java.lang` 包中。  
  
虽然这种方法并没有完全避免类注入，但仅注入单一类可以减少代理添加的永久类的数量，从而减少未来 JDK 版本不再允许这种注入时对现有代理进行重构的需求。  
  
在 Elastic APM Java 代理中，引导方法将创建一个新的类加载器，其父类加载器是被插桩类的类加载器，并从中加载 advice 类及所需的任何辅助类。然后，我们可以根据引导方法中提供的 advice 类名（例如：`org.example.ServletAdvice`），从这个新创建的类加载器中加载对应的 advice 类。  
  
通过引导方法的其他参数，我们可以构造一个 `MethodHandle` 和一个 `CallSite`，用于表示我们创建的子类加载器中的 advice 方法。对于我们的需求，目标方法始终保持不变，因此可以返回一个 `ConstantCallSite`，从而允许 JIT 内联 advice 方法。  
  
现在，我们仅依赖一个类对被插桩方法可见（即 `java.lang.IndyBootstrapDispatcher`），可以进一步通过专用类加载器加载代理的其他类，从而实现代理的隔离。正如上一节所述，通过将代理的类隐藏在常规类加载器层次结构之外，可以避免兼容性问题，例如与类路径扫描器的冲突。这种隔离还允许代理携带任何依赖库（如 Byte Buddy 或日志库），而无需对这些依赖进行重定位（即 “shade” 到代理的命名空间）。这不仅简化了代理的调试过程，也无需担心应用程序类加载器层次结构中可能存在的类冲突。关于这种隔离类加载器的具体实现，可以参考 Elastic APM Java 代理源码中的 [ShadedClassLoader](https://github.com/elastic/apm-agent-java/blob/43b0e11917a4f6eddb38b02bfe7a5917985058d9/elastic-apm-agent/src/main/java/co/elastic/apm/agent/premain/ShadedClassLoader.java#L1)  
  
生成的类加载器层次结构如下所示：  
![class_hierarchy](/img/in-post/elastic_apm/class_hierarchy.png)  
  
需要注意的是，用于加载 advice 和与特定库相关的辅助类的代理辅助类加载器拥有两个父加载器：被插桩类的类加载器（例如，servlet 容器为每个 Web 应用程序创建的类加载器）和代理类加载器。这种设计使得 advice 和辅助类既可以访问被插桩类的类加载器可见的类型，又可以访问代理类加载器可见的类型。尽管内置的类加载器不支持多父类加载器，但自己实现一个相对简单。Byte Buddy 还提供了一个名为 `net.bytebuddy.dynamic.loading.MultipleParentClassLoader` 的实现。
> 译者注：在java的classloader中，一个classloader只能有一个父类。(`private final ClassLoader parent`)，这里作者的意思是需要自己实现一个类加载器或使用byte-buddy，来支持多父类加载器。  
  
总结来说，本节描述了**如何使用 invokedynamic 指令调用由被插桩类的定义类加载器的子类加载器加载的 advice 方法**。这样设计使代理能够隐藏其类，避免暴露给应用程序，同时提供了一种从被插桩的应用程序类中调用隔离方法的机制。这种方式非常有用，因为由该类加载器加载的 advice 和其他类可以访问被插桩库的类，而 advice 代码仍然以常规代码的形式执行。此外，这种方法避免了将 advice 和辅助类直接注入目标类加载器的需求，因为这种注入目前只能通过 Oracle 正在逐步锁定的内部 API 实现。  
  
## AssignReturned
尽管使用内联或委托的 advice 是通过相同的 API 实现的，并且在表面上看起来非常相似，但二者之间仍然存在差异。委托型 advice 很难在被插桩方法的作用域内直接修改值。而在内联情况下，advice 方法可以直接为带注解的参数赋值，Byte Buddy 会在内联过程中将这些赋值转换为对应值的替换。

例如，以下内联 advice 会将被插桩方法的第一个参数（此处是一个 Runnable）替换为一个包装实例。这个包装实例同样实现了 Runnable 接口，并将任何后续调用报告回代理：
```java
@Advice.OnMethodEnter
public static void enter(
        @Advice.Argument(value = 0, readOnly = false) Runnable callback) {
    callback = new TracingRunnable(callback);
}
```
由于上述代码是内联的，advice 直接替换了被插桩方法第一个参数的值。因此，被插桩方法的执行效果就如同调用者已经将 `TracingRunnable` 作为参数传递给它一样。

然而，当使用委托方式时，这种替换是无法实现的。在委托模式下，新值只会赋给 advice 方法的参数，而不会影响被插桩方法的参数赋值。换句话说，在 advice 方法执行完毕后，被插桩方法的参数仍然保持原来的 Runnable 值。  

为了在使用委托型 advice 时实现这种赋值功能，Byte Buddy 最近引入了 Advice.AssignReturned 后处理器。Advice 的后处理器是一种在 advice 方法被调用之后执行的处理器，允许执行额外的操作，而这些操作与具体应用的 advice 是独立的。更重要的是，即使 advice 是通过委托调用的，后处理器生成的代码也总是被内联到被插桩方法中。这使得如果 advice 方法返回了某个值，可以将这些值写入被插桩方法的作用域。

作为对常规 Advice 实现的扩展，后处理器需要先通过以下方法手动注册：
```java
Advice.withCustomBinding()
    .with(new Advice.AssignReturned.Factory());
```
顾名思义，这个后处理器允许将 advice 方法返回的值赋给被插桩方法的参数。为了实现上述示例，可以指示后处理器将返回值赋给被插桩方法的第一个参数，具体实现方式如下：
```java
@Advice.OnMethodEnter(inline = false)
@Advice.AssignReturned.ToArguments(@ToArgument(0))
public static Runnable enter(@Advice.Argument(0) Runnable callback) {
    return new TracingRunnable(callback);
}
```
与内联示例一样，被插桩方法现在将观察到其第一个参数已被替换为 `TracingRunnable`，这是通过后处理器完成的。此外，除了赋值给方法参数外，还可以将值赋给类字段、方法的返回值、抛出的异常，甚至是 this 引用（对于非静态方法）。

在某些情况下，可能需要进行多个值的赋值。对于内联 advice，这可以通过直接在 advice 方法中将多个值赋给每个带注解的参数来轻松实现。而对于委托型 advice，同样可以通过返回一个数组作为返回类型，并指定返回数组中各个索引对应的值来实现多个赋值。  
  

扩展这个假设的例子，假设被插桩方法还需要一个 ExecutorService 作为第二个参数，我们可以通过在 advice 方法的返回数组中提供一个新创建的缓存线程池作为第二个值来强制使用它。通过为 advice 方法的赋值操作添加注解，每个赋值只需要指明返回数组中哪个索引对应哪个赋值的值即可。  
```java
@Advice.OnMethodEnter(inline = false)
@Advice.AssignReturned.ToArguments(
  @ToArgument(value = 0, index = 0, typing = DYNAMIC),
  @ToArgument(value = 1, index = 1, typing = DYNAMIC))
public static Runnable enter(@Advice.Argument(0) Runnable callback) {
    return new Object[] {
        new TracingRunnable(callback),
        Executors.newCachedThreadPool()
    };
}
```
最后，由于 Object 类型的数组可能包含无法赋值的值，注解必须指定使用动态类型。这种情况下，Byte Buddy 会在赋值之前尝试对值进行类型转换。为了避免可能的 `ClassCastException` 影响被插桩的应用程序，可以配置后处理器以屏蔽这些异常。
```java
new Advice.AssignReturned().Factory()
    .withSuppressed(ClassCastException.class)
```
如果未在数组包含无法赋值的值时配置动态类型，在类插桩过程中将会抛出异常。尽管这会导致插桩失败，但不会对应用程序本身造成影响。
> 译者注：
> 1. 这里主要是说明byte-buddy中`@Advice.Assign***`的用法。译者也写过一篇[ByteBuddy @Advice.Assign 注解使用详解](/byte-buddy/2024/11/30/byte-buddy-advice-assign/)，感兴趣的可以看看。  
> 2. 在后置处理器里，需要对拿到的动态类型进行强转，否则会报错。

## 权衡 (Trade-offs)
这种架构的一个限制是无法支持 Java 6 应用程序，因为它依赖于 Java 7 中引入的 `invokedynamic` 字节码指令。然而，对于 Elastic APM Java 代理来说，这并不是问题，因为它从未支持过 Java 6。事实上，许多其他代理甚至已经不再支持 Java 7，其市场占有率根据不同研究大约在 1-5% 之间。  
除了要求运行环境为 Java 7 或更高版本外，被插桩的类也必须是字节码版本 51（即编译目标为 Java 7 或更高）。这是因为对于更早的类文件版本，无法使用 `invokedynamic` 指令。这对某些库来说可能会是个问题，尤其是较旧的 JDBC 驱动程序，代理可能需要对它们进行插桩，而这些驱动程序通常是用非常旧的类文件版本编译的。 
  
不过，这个问题有一个相对简单的解决方案：通过使用 `ClassVisitor`，可以让 ASM 将字节码重写为类文件版本 51（Java 7）。这种方法自引入 Elastic APM Java 代理以来，已被证明是稳定且可靠的。虽然这种方法会带来一些性能开销，但只有在被插桩类的类文件版本低于 51 的较少情况下才需要这样做，因此性能影响相对有限。  
  
需要注意的是，早期版本的 Java 7（更新 60 之前，发布于 2014 年 5 月）和 Java 8（更新 40 之前，发布于 2015 年 3 月）在 invokedynamic 和 MethodHandle 支持上存在一些已知的漏洞。因此，如果检测到运行环境是这些 JVM 版本，Elastic APM Java 代理会自动禁用自身。  
  
> 译者注：  
> 1. 使用invokedynamic指令，需要JDK7及以上版本。 同时一些比较老的jar，如JDBC驱动，需要使用ASM将字节码重写为类文件版本 51（Java 7）。  
> 2. 这里有一个坑是byte-buddy并不直接支持ASM的`ClassVisitor`，需要使用byte-buddy-dep来实现。具体实现可以参照[PatchBytecodeVersionTo51Transformer](https://github.com/elastic/apm-agent-java/blob/main/apm-agent-core/src/main/java/co/elastic/apm/agent/bci/bytebuddy/PatchBytecodeVersionTo51Transformer.java)。  

## 后续步骤 (Next steps)
要深入了解 Elastic APM Java Agent 及其如何帮助您识别和解决应用程序中的性能问题，请参阅[官方文档](https://www.elastic.co/guide/en/apm/agent/java/current/intro.html)。   
如果您有兴趣构建自己的 Java 代理，请访问 [Byte Buddy](https://bytebuddy.net/) 获取更多信息。
