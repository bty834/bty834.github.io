---
title: Java lambda表达式原理简述
categories: [ 编程,Java ]
tags: [ java lambda]
---

## 背景

不同于脚本语言可以直接调用函数，Java作为面向对象的语言需要提前创建类并实例化对象来调用实例方法，使用起来十分笨重。

比如我们需要构造一个如下`ActionListener`接口的类：
```java
public interface ActionListener { 
    void actionPerformed(ActionEvent e);
}
```
需要重新写一套`class TestActionListener implements ActionListener {...}` ，
如果我们有N种不同的实现，就需要写N次，很麻烦，lambda出现之前，可以使用匿名类的方式省去显式class的创建：

```java
button.addActionListener(new ActionListener() { 
  public void actionPerformed(ActionEvent e) { 
    System.out.println("one")
  }
});
button.addActionListener(new ActionListener() { 
  public void actionPerformed(ActionEvent e) { 
    System.out.println("two")
  }
});
```
但匿名类有以下缺点：
1. 语法笨重
2. 变量和this关键字指向不清晰
3. Inflexible class-loading and instance-creation semantics
4. Inability to capture non-final local variables 
5. Inability to abstract over control flow

关于变量和this关键字指向不清晰，可以看以下案例：
```java
   void caseOne() {
        String a = "a";
        Runnable runnable = new Runnable() {
            @Override
            public void run() {
                System.out.println(a); // "a"
            }
        };
        runnable.run();
    }
    
    ---
    
    void caseTwo() {
        String a = "a";
        Runnable runnable = new Runnable() {
            String a = "b";
            @Override
            public void run() {
                System.out.println(this.a); // "b" this始终表示当前匿名类对象
                System.out.println(a); // "b" 匿名类中有变量a时，匿名类外同名变量失效
            }
        };
        runnable.run();
    }
    
```
lambda表达式的出现消除了第1和第2点，其中lambda中使用this关键字始终指向外部对象，这点和匿名类不一样。lambda避开了第3点的繁琐的类创建和实例化，第4点lambda增加了外部字段的final or effectively final检测
，第4点和第5点问题并未被lambda表达式解决。

## 基本原理

我们创建一个lambda表达式并反编译看下bytecode:
```java
import java.util.function.Function;

public class TestLambda {
    public static void sayHelloWorld() {
        String world = "World";
        Function<String,String> func = (hello) -> hello + world;
        func.apply("hello");
    }
}
```
bytecode: 
```java
// class version 52.0 (52)
// access flags 0x21
public class TestLambda {

  // compiled from: TestLambda.java
  // access flags 0x19
  public final static INNERCLASS java/lang/invoke/MethodHandles$Lookup java/lang/invoke/MethodHandles Lookup

  // access flags 0x1
  public <init>()V
   L0
    LINENUMBER 3 L0
    ALOAD 0
    INVOKESPECIAL java/lang/Object.<init> ()V
    RETURN
   L1
    LOCALVARIABLE this LTestLambda; L0 L1 0
    MAXSTACK = 1
    MAXLOCALS = 1

  // access flags 0x9
  public static sayHelloWorld()V
   L0
    LINENUMBER 6 L0
    LDC "World"
    ASTORE 0
   L1
    LINENUMBER 7 L1
    ALOAD 0
    INVOKEDYNAMIC apply(Ljava/lang/String;)Ljava/util/function/Function; [
      // handle kind 0x6 : INVOKESTATIC
      java/lang/invoke/LambdaMetafactory.metafactory(Ljava/lang/invoke/MethodHandles$Lookup;Ljava/lang/String;Ljava/lang/invoke/MethodType;Ljava/lang/invoke/MethodType;Ljava/lang/invoke/MethodHandle;Ljava/lang/invoke/MethodType;)Ljava/lang/invoke/CallSite;
      // arguments:
      (Ljava/lang/Object;)Ljava/lang/Object;, 
      // handle kind 0x6 : INVOKESTATIC
      TestLambda.lambda$sayHelloWorld$0(Ljava/lang/String;Ljava/lang/String;)Ljava/lang/String;, 
      (Ljava/lang/String;)Ljava/lang/String;
    ]
    ASTORE 1
   L2
    LINENUMBER 8 L2
    ALOAD 1
    LDC "hello"
    INVOKEINTERFACE java/util/function/Function.apply (Ljava/lang/Object;)Ljava/lang/Object; (itf)
    POP
   L3
    LINENUMBER 9 L3
    RETURN
   L4
    LOCALVARIABLE world Ljava/lang/String; L1 L4 0
    LOCALVARIABLE func Ljava/util/function/Function; L2 L4 1
    // signature Ljava/util/function/Function<Ljava/lang/String;Ljava/lang/String;>;
    // declaration: func extends java.util.function.Function<java.lang.String, java.lang.String>
    MAXSTACK = 2
    MAXLOCALS = 2

  // lambda中用到的外部变量作为方法参数传入，返回值类型为实现的接口方法的返回值类型
  // access flags 0x100A
  private static synthetic lambda$sayHelloWorld$0(Ljava/lang/String;Ljava/lang/String;)Ljava/lang/String;
   L0
    LINENUMBER 7 L0
    NEW java/lang/StringBuilder
    DUP
    INVOKESPECIAL java/lang/StringBuilder.<init> ()V
    ALOAD 1
    INVOKEVIRTUAL java/lang/StringBuilder.append (Ljava/lang/String;)Ljava/lang/StringBuilder;
    ALOAD 0
    INVOKEVIRTUAL java/lang/StringBuilder.append (Ljava/lang/String;)Ljava/lang/StringBuilder;
    INVOKEVIRTUAL java/lang/StringBuilder.toString ()Ljava/lang/String;
    ARETURN
   L1
    LOCALVARIABLE world Ljava/lang/String; L0 L1 0
    LOCALVARIABLE hello Ljava/lang/String; L0 L1 1
    MAXSTACK = 2
    MAXLOCALS = 2
}

```

可以看到lambda表达式的产生了一个 invokedynamic + 一个静态方法 + 一个静态字段的字节码。

### invokedynamic


在 Java7 之前，JVM 提供了如下 4 种【方法调用】指令：

|                                      | `invokevirtual` | `invokestatic` | `invokeinterface` | `invokespecial`                   |
|--------------------------------------|-----------------|----------------|-------------------|-----------------------------------|
| 适用于                                  | instance method | class method   | interface method  | constructor, super class, private |
| 是否有this pointer                      | 是               | 否              | 是                 | 是                                 |
| 是否进行virtual lookup (运行时才能知道具体调用那个方法) | 是，使用`vtable`    | -              | 否，使用`itable`      | _**否**_                           |
| 是否static linking(编译器就知道调用哪个方法)       | -               | 是              | -                 | 是                                 |

`invokespecial`既包含this pointer也有static linking，且还不进行virtual lookup，为什么？因为`invokespecial`的适用方法如构造函数无法被override，
无需virtual lookup，但构造函数需要初始化实例属性，所以需要this pointer

说回`invokedynamic`: 用于处理新型的方法分派，**允许应用级别的代码来确定执行哪一个方法调用**，只有在调用要执行的时候，才会进行这种判断，

比如，用户编写lambda表达式：`() -> a.test(); b.test();` Jvm在执行该lambda表达式时，
会根据用户在lambda中编写的内容（即调用a和b的test方法）来执行。


实现上，`invokedynamic`使用`java.lang.invoke.LambdaMetafactory#metafactory`方法（也叫引导方法，`Bootstrap method`）返回`java.lang.invoke.CallSite`对象，
`CallSite`对象返回`java.lang.invoke.MethodHandle`方法句柄并执行由lambda生成的静态方法。

注意，lambda表达式并不是`invokedynamic`调用执行的，`invokedynamic`调用只是生成了lambda方法对应的CallSite对象，具体执行还得靠`invokeinterface`或`invokevirtual`。


Jvm最初是为了Java而设计的，但后面出现很多语言均以Jvm作为底层实现，如groovy等，
`invokedynamic`的引入则是为了兼容其他动态类型语言的 dynamic invocation。比如 `def(a,b)`在动态类型语言中，
a和b可以在运行时传入不同类型，但在`invokedynamic`之前Jvm都是静态类型的，无法动态支持这种方法调用，因此引入`invokedynamic`支持这一特性。
![](/assets/2024/09/20/indy.png)

而Java本身是静态类型，所以"动态"在Java中体现不出来，上面的CallSite返回的也是ConstantCallSite，只会在第一次调用生成，后面的调用会直接走MethodHandle指向的方法逻辑，不会重复生成CallSite。

如果`invokedynamic`最初是Jvm为了兼容动态语言推出的，那为什么Java要用呢？因为Java虽然是强类型，但是生成执行代码的方式也是动态的？是走内部类？还是MethodHandle？还是动态代理？
这个方式可以由用户来定义，比如lambda就是通过MethodHandle实现的。

关于`java.lang.invoke.MethodHandle`:
![](/assets/2024/09/20/methodHandler.png)

以上都是Jvm的内部实现，用户代码其实只有一个lambda，我们使用Java代码来模拟下上面的调用方式，模拟之前先来了解一个名词`duck typing`:

> If it looks like a duck and quacks like a duck, it's a duck

同理，`Supplier`接口接收空参数，返回String类型，如果我们实现了一个类接收空参数，返回String类型，那它就实现了`Supplier`接口，
当然在Java这种强类型语言肯定有一个类型转换过程。

```java
import java.lang.invoke.CallSite;
import java.lang.invoke.LambdaMetafactory;
import java.lang.invoke.MethodHandle;
import java.lang.invoke.MethodHandles;
import java.lang.invoke.MethodType;
import java.util.Collections;
import java.util.function.Supplier;

public class LambdaExpTest {


    public static String innerLambdaCode() {
        return "i'm lambda code";
    }

    // 等同于 Supplier<String> supplier = () -> "i'm lambda code";
    public static void mockLambdaInJvm() throws Throwable {
        MethodHandles.Lookup lookup = MethodHandles.lookup();

        // lambda实现的是Supplier接口，该方法需要接收lambda使用到的外部参数并返回一个Supplier，类似一个转换方法
        // The parameter types represent the types of capture variables; 入参是lambda使用到的外部参数类型
        // the return type is the interface to implement 出参是lambda实现的接口类型
        MethodType convertMethodType = MethodType.methodType(Supplier.class, Collections.emptyList());

        // Supplier的方法入参出参类型
        MethodType supplierMethodType = MethodType.methodType(Object.class, Collections.emptyList());

        CallSite callSite = LambdaMetafactory.metafactory(
            lookup,
            "get",  // 调用的Supplier接口的方法名
            convertMethodType,
            supplierMethodType,
            // lambda内部逻辑委托给静态方法LambdaExpTest#innerLambdaCode执行
            lookup.findStatic(LambdaExpTest.class, "innerLambdaCode", MethodType.methodType(String.class)),
            supplierMethodType);

        MethodHandle factory = callSite.getTarget();
        Supplier<String> r = (Supplier<String>)factory.invoke();
        System.out.println(r.get());
    }
    public static void main(String[] args) throws Throwable {
        // 打印 i'm lambda code
        mockLambdaInJvm();
    }
}
```

## 参考
- [Lambdas in Java: A Peek under the Hood • Brian Goetz • GOTO 2013](https://www.youtube.com/watch?v=MLksirK9nnE)
- [why-are-java-8-lambdas-invoked-using-invokedynamic](https://stackoverflow.com/questions/30002380/why-are-java-8-lambdas-invoked-using-invokedynamic)
- [understanding-java-method-invocation-with-invokedynamic](https://blogs.oracle.com/javamagazine/post/understanding-java-method-invocation-with-invokedynamic)
- [why-use-reflection-to-access-class-members-when-methodhandle-is-faster](https://stackoverflow.com/questions/30677670/why-use-reflection-to-access-class-members-when-methodhandle-is-faster?noredirect=1&lq=1)
- [what-is-a-bootstrap-method](https://stackoverflow.com/questions/30733557/what-is-a-bootstrap-method)
- [whats-invokedynamic-and-how-do-i-use-it](https://stackoverflow.com/questions/6638735/whats-invokedynamic-and-how-do-i-use-it)
- [what-is-duck-typing](https://stackoverflow.com/questions/4205130/what-is-duck-typing)
- [VirtualCalls](https://wiki.openjdk.org/display/HotSpot/VirtualCalls)
- [State of the Lambda](https://cr.openjdk.org/~briangoetz/lambda/lambda-state-final.html)
- [An Introduction to Invoke Dynamic in the JVM](https://www.baeldung.com/java-invoke-dynamic)
- [浅析 JVM invokedynamic 指令和 Java Lambda 语法｜得物技术](https://xie.infoq.cn/article/94c7db97ac7a8d555c2335518)
- [Rethinking Java String Concatenation](https://www.youtube.com/watch?v=tgX38gvMpjs)
