---
title: Java Method Overloading Extra Story
date: 2020-08-25 16:36:40
tags:
---
很久沒寫東西，其實我不知道面試問這個的意義。是要考倒求職者彰顯自己的能力?還是甚麼目的?
真的面試到Java那麼熟的碼農，大多數台灣企業做的東西用的上這些知識又有幾間? 公司又能花多少錢養這種工程師? XD

## Quick Start

### Method Overloading

Java 101教我們的Method Overloading機制允許我們在一個Java class裡可以有多個相同method name的methods，但argument list必須不一樣。

也就是說如果只有return type不一樣就不行，試著寫下列程式在compile(通常IDE就會給你紅色波浪)的時候就會發生錯誤。

``` java
public class TestMethodOverload {
    public void test() {

    }

    public int test() {
        return 0;
    }
}
```

在Terminal編譯的話會看到下列的錯誤

```
PS D:\tmp> javac .\TestMethodOverload.java
.\TestMethodOverload.java:6: error: method test() is already defined in class TestMethodOverload
    public int test() {
               ^
1 error
```

### Extra Story

問題來了，如果同一個類別裡，有兩個同名的方法，argument list一樣，只有回傳型態不一樣，在JVM是合法的嗎?

答案是【是合法的】，JVM是允許有這樣的情況，method overloading這樣的規定是Java語言的規範，所以是java compiler會抱怨你這樣寫，但是如果你能產生這種情況的bytecode class，JVM是不在乎的。

證明下面這個例子，用BCEL產生一個簡單的Java Class，HelloJVM類別裡面有兩個test1()只有return type的差異，一個，一個回int一個回double。然後順便弄個main method讓我們可已執行一下這兩個test1()。

``` java
        final String ClassName = "HelloJVM";
        ClassGen classGen = new ClassGen(ClassName,
                "java.lang.Object",
                "<generated>",
                Constants.ACC_PUBLIC | Constants.ACC_SUPER,
                null);
        ConstantPoolGen constantPoolGen = classGen.getConstantPool();
        InstructionList instructionList = new InstructionList();
        InstructionFactory instructionFactory = new InstructionFactory(classGen);

        MethodGen test1MethodGen = new MethodGen(Constants.ACC_PUBLIC,
                Type.INT,
                Type.NO_ARGS,
                null,
                "test1",
                ClassName,
                instructionList,
                constantPoolGen);
        instructionList.append(new PUSH(constantPoolGen, 5));
        instructionList.append(InstructionFactory.createReturn(Type.INT));

        test1MethodGen.setMaxLocals();
        test1MethodGen.setMaxStack();
        classGen.addMethod(test1MethodGen.getMethod());
        instructionList.dispose();

        MethodGen test2MethodGen = new MethodGen(Constants.ACC_PUBLIC,
                Type.DOUBLE,
                Type.NO_ARGS,
                null,
                "test1",
                ClassName,
                instructionList,
                constantPoolGen);
        instructionList.append(new PUSH(constantPoolGen, 10.112));
        instructionList.append(InstructionFactory.createReturn(Type.DOUBLE));

        test2MethodGen.setMaxLocals();
        test2MethodGen.setMaxStack();
        classGen.addMethod(test2MethodGen.getMethod());
        instructionList.dispose();

        MethodGen mainMethodGen = new MethodGen(Constants.ACC_PUBLIC | Constants.ACC_STATIC,
                Type.VOID,
                new Type[] {
                        new ArrayType(Type.STRING, 1)
                },
                new String[] {"argv"},
                "main",
                ClassName,
                instructionList,
                constantPoolGen);
        ObjectType p_stream = new ObjectType("java.io.PrintStream");

        instructionList.append(instructionFactory.createNew(ClassName));
        instructionList.append(InstructionFactory.createDup(1));
        instructionList.append(instructionFactory.createInvoke(ClassName,
                "<init>",
                Type.VOID,
                new Type[] {},
                (short) Constants.INVOKESPECIAL));
        instructionList.append(InstructionFactory.createStore(Type.OBJECT, 1));
        instructionList.append(instructionFactory.createFieldAccess("java.lang.System",
                "out",
                p_stream,
                (short) Constants.GETSTATIC));
        instructionList.append(InstructionFactory.createLoad(Type.OBJECT, 1));
        instructionList.append(instructionFactory.createInvoke(ClassName,
                "test1",
                Type.INT,
                Type.NO_ARGS,
                (short) Constants.INVOKEVIRTUAL));
        instructionList.append(instructionFactory.createInvoke("java.io.PrintStream",
                "println",
                Type.VOID,
                new Type[] {Type.INT},
                (short) Constants.INVOKEVIRTUAL));
        instructionList.append(instructionFactory.createFieldAccess("java.lang.System",
                "out",
                p_stream,
                (short) Constants.GETSTATIC));
        instructionList.append(InstructionFactory.createLoad(Type.OBJECT, 1));
        instructionList.append(instructionFactory.createInvoke(ClassName,
                "test1",
                Type.DOUBLE,
                Type.NO_ARGS,
                (short) Constants.INVOKEVIRTUAL));
        instructionList.append(instructionFactory.createInvoke("java.io.PrintStream",
                "println",
                Type.VOID,
                new Type[] {Type.DOUBLE},
                (short) Constants.INVOKEVIRTUAL));

        instructionList.append(InstructionFactory.createReturn(Type.VOID));

        mainMethodGen.setMaxStack();
        mainMethodGen.setMaxLocals();
        classGen.addMethod(mainMethodGen.getMethod());

        instructionList.dispose();

        classGen.addEmptyConstructor(Constants.ACC_PUBLIC);

        try {
            classGen.getJavaClass().dump("HelloJVM.class");
        } catch(IOException e) {
            e.printStackTrace();
        }
```

[完整程式碼](https://github.com/tzuyichao/java-basic/blob/master/java-basic/src/main/java/bcel/Test1BCEL.java)，接下來是來執行一下產生的HelloJVM class，順便用javap看一下。


```
D:\lab\java-basic\java-basic>java HelloJVM
5
10.112

D:\lab\java-basic\java-basic>javap HelloJVM.class
Compiled from "<generated>"
public class HelloJVM {
  public int test1();
  public double test1();
  public static void main(java.lang.String[]);
  public HelloJVM();
}
```

