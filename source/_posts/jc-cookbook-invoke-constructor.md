---
title: Javac (JC) Cookbook - How to create instance via invoke default constructor
date: 2020-09-02 09:46:09
tags: 
  - java
  - javac
---
lombok除了JSR 269之外的黑科技。簡單來說lombok是透過Java 6推出的JSR 269 (Pluggable Annotation Processing API)和修改AST這兩個東西，讓我們寫程式更輕鬆。

<code>JCTree</code>就是讓我們可以處理Java AST的API。兩件事要先講，首先根據<code>JCTree</code>的JavaDoc寫類別名稱看到JC是javac的縮寫。接著，這些API是internal使用，有可能改變或刪除，如果要用後果自行負責。

> This is NOT part of any supported API. If you write code that depends on this, you do so at your own risk. This code and its internal interfaces are subject to change or deletion without notice.

## How to create instance via invoke default constructor

### Question

如何透過JCTree API增加一個呼叫default constructor產生物件?

### Answer

透過<code>TreeMaker#NewClass()</code>
``` java
    public JCNewClass NewClass(JCExpression encl,
                             List<JCExpression> typeargs,
                             JCExpression clazz,
                             List<JCExpression> args,
                             JCClassDecl def)
    {
        return SpeculativeNewClass(encl, typeargs, clazz, args, def, false);
    }
```

在<code>JCTree</code>可以看到<code>JCNewClass</code>類別。

``` java
    /**
     * A new(...) operation.
     */
    public static class JCNewClass extends JCPolyExpression implements NewClassTree {
 
```

下面是一個簡單的例子，透過呼叫<code>StringBuilder</code>的default constructor產生一個<code>StringBuilder</code>，後續使用<code>StringBuilder#append(str: String)</code>。

``` java
        JCTree.JCExpression varType = query("java", "lang", "StringBuilder");

        Name result = names.fromString("result");
        JCTree.JCModifiers resultModifiers = treeMaker.Modifiers(Flags.PARAMETER);

        JCTree.JCNewClass stringBuilderClass = treeMaker.NewClass(null,
                com.sun.tools.javac.util.List.nil(),
                varType,
                com.sun.tools.javac.util.List.nil(),
                null);

        JCTree.JCVariableDecl resultIdent = treeMaker.VarDef(resultModifiers, result, varType, stringBuilderClass);
```

對應產生的Java code就是

``` java
        StringBuilder result = new StringBuilder();
```

### Discussion

#### Anonymous class: 以Runnable為例

舉例來說，我們希望在AST裡加上如下列的內容

``` java
        Runnable r = new Runnable() {
            public void run() {

            }
        };
```

就會使用到<code>TreeMaker#NewClass()</code>的第五個參數。


``` java
        JCTree.JCExpression runnableType = query("java", "lang", "Runnable");

        Name runnable = names.fromString("r");
        JCTree.JCModifiers rModifiers = treeMaker.Modifiers(Flags.PARAMETER);
        ListBuffer<JCTree.JCStatement> runStatements = new ListBuffer<>();
        JCTree.JCBlock rBody = treeMaker.Block(0, runStatements.toList());

        JCTree.JCMethodDecl runMethod = treeMaker.MethodDef(
                treeMaker.Modifiers(PUBLIC),
                names.fromString("run"),
                treeMaker.Type(new Type.JCVoidType()),
                com.sun.tools.javac.util.List.nil(),
                com.sun.tools.javac.util.List.nil(),
                com.sun.tools.javac.util.List.nil(),
                rBody,
                null
        );

        List<JCTree> defs = List.of(runMethod);

        JCTree.JCClassDecl classDecl = treeMaker.AnonymousClassDef(treeMaker.Modifiers(PARAMETER),
                defs);

        JCTree.JCNewClass rClass = treeMaker.NewClass(null,
                com.sun.tools.javac.util.List.nil(),
                runnableType,
                com.sun.tools.javac.util.List.nil(),
                classDecl);

        JCTree.JCVariableDecl rIdent = treeMaker.VarDef(rModifiers, runnable, runnableType, rClass);
```
