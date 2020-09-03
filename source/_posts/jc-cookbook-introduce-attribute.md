---
title: Javac (JC) Cookbook - How to introduce new attribute into exist class
date: 2020-09-02 23:50:40
tags:
  - java
  - javac
---
### Question

對已經存在的類別加入新的attribute

### Answer

作法是透過<code>TreeMaker#VerDef()</code>來產生新的variable的定義，然後透過<code>JCClassDecl</code>的<code>defs#prepend()</code>加到類別的Class Tree上。

以下是使用<code>TreeMaker#VerDef()</code>產生<code>serialVersionUID</code>的定義和初始化。

``` java
    private JCTree.JCVariableDecl genSerialVersionUIDVariable() {
        JCTree.JCModifiers modifiers = treeMaker.Modifiers(PRIVATE + STATIC + FINAL);
        Name id = names.fromString("serialVersionUID");
        JCTree.JCExpression varType = treeMaker.Type(new Type.JCPrimitiveType(TypeTag.LONG, null));
        JCTree.JCExpression init = treeMaker.Literal(1L);
        return treeMaker.VarDef(modifiers, id, varType, init);
    }
```

以下是把VariableTree加到ClassTree的寫法

``` java
        jcClassDecl.defs = jcClassDecl.defs.prepend(genSerialVersionUIDVariable());
```

這個VariableTree對應的java程式碼如下

``` java
private static final long serialVersionUID = 1L;
```

### Discussion



