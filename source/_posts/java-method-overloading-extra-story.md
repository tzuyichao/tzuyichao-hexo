---
title: Java Method Overloading Extra Story
date: 2020-08-25 16:36:40
tags:
---
很久沒寫東西，其實我不知道面試問這個的意義。是要考倒求職者彰顯自己的能力?還是甚麼目的?
真的面試到Java那麼熟的碼農，大多數台灣企業做的東西用的上這些知識又有幾間? XD

## 前言

### Method Overloading

### Extra Story


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

