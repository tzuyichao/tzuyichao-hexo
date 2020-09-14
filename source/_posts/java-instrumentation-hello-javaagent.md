---
title: Java instrumentation - hello javaagent
date: 2020-09-13 11:25:51
tags: 
  - java instrumentation
---
想直接看程式碼[hello-javaagent Project](https://github.com/tzuyichao/java-basic/tree/master/instrumentation/hello-javaagent)、[hello-attach Project](https://github.com/tzuyichao/java-basic/tree/master/instrumentation/hello-attach)的話，直接點連結即可。

hello-javaagent Project講用<code>-javaagent</code>的方式使用java instrumentation的hello world project。這是個multi module maven project。有兩個module一個是client一個是agent部分。另外，還有一個run.xml的ant script裡面有兩個task用來執行client，一個task有啟動agent一個沒有，可以感覺一下差異。

hello-attach project則是使用attach的方式，有三個project，client subproject是用spring boot寫的client，attach subproject是用來attach agent jar到target jvm，最後agent subproject是agent的本體。

### Java Instrumentation

Instrumentation讓我們可以在既有（compiled)方法插入額外的bytecode，而達到收集使用中資料來進一步分析使用。instrumentation底層是透過<code>JVM Tool Interface</code>(<code>JVMTI</code>)來達成，JVMTI是JVM暴露出來的extension point。

在Java 5的時候讓我們可以在啟動的時候使用<code>-javaagent</code>的方式，指定agent。在Java 6的時候則更近一步提供了attach的方式，讓我們可以針對已啟動的JVM process執行agent的工作。

- 啟動JVM時啟動代理 (hello-javaagent project)
- 針對已啟動JVM啟動代理 (hello-attach project)

### hello-javaagent

#### 入口

入口class我們要提供<code>premain()</code>這個方法，需要兩個參數

> ``` java
> public static void premain(String agentArgs, Instrumentation inst)
> ```

第一個參數是啟動agent時，可以由命令列給予agent的參數，第二個則是JVM會提供agent class使用的Instrumentation的入口。

#### Instrumentation Interface

- void addTransformer(ClassFileTransformer transformer, boolean canRetransform)

註冊transformer

- Class[] getAllLoadedClasses()

取得被load到JVM的所有class

- void retransformClasses(Class<?>... classes) throws UnmodifiableClassException

針對已經loaded的classes重新在instrumentation機制由agent調整bytecode

#### Configuration

##### MANIFEST.MF

也可以用maven build file去產生，我選擇自己寫，然後在maven build file設定使用自己寫的MANIFEST.MF。

```
Premain-Class: main.AgentMain
Can-Redefine-Classes: true
Can-Retransform-Classes: true
```

maven build file相關設定方式見Maven Assembly Plugin。

##### Maven Assembly Plugin

因為有ASM的依賴，所以打包jar的時候要連同依賴的library一起包進去才行。以前都用copy reference，改用assembly plugin做看看。

```
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-assembly-plugin</artifactId>
        <version>${maven-assembly-plugin.version}</version>
        <configuration>
          <archive>
            <manifestFile>src/main/resources/META-INF/MANIFEST.MF</manifestFile>
          </archive>
          <descriptorRefs>
            <descriptorRef>jar-with-dependencies</descriptorRef>
          </descriptorRefs>
        </configuration>
        <executions>
          <execution>
            <id>make-assembly</id>
            <phase>package</phase>
            <goals>
              <goal>single</goal>
            </goals>
          </execution>
        </executions>
      </plugin>
```

#### Using ASM for manipulate bytecode

hello project的目的是在我們在意的**類別**裡在意的**method**被呼叫前和呼叫後在Console印個log下來。所以我們要做的事情是寫一個<code>ClassFileTransformer</code>，在這裡使用ASM處理bytecode。然後在<code>premain()</code>裡把我們寫的transformer加進去。

``` java
public class MyClassTransformer implements ClassFileTransformer {
    @Override
    public byte[] transform(ClassLoader loader, String className, Class<?> classBeingRedefined, ProtectionDomain protectionDomain, byte[] classfileBuffer) throws IllegalClassFormatException {
        if(!"app/MyTest".equals(className)) {
            return classfileBuffer;
        }
        ClassReader classReader = new ClassReader(classfileBuffer);
        ClassWriter classWriter = new ClassWriter(classReader, ClassWriter.COMPUTE_FRAMES | ClassWriter.COMPUTE_MAXS);
        ClassVisitor classVisitor = new MyClassVisitor(classWriter);
        classReader.accept(classVisitor, ClassReader.SKIP_FRAMES | ClassReader.SKIP_DEBUG);
        return classWriter.toByteArray();
    }
}
```

這裡我們在意的類別是app.MyTest這個類別，所以在transformer裡遇到不在乎的class就把該class的bytecode直接回傳；如果遇到MyTest類別，就透過ASM來處理一下。

``` java
public class MyClassVisitor extends ClassVisitor {
    public static final String NAME_CONSTRUCTOR = "<init>";
    public static final String NAME_MAIN = "main";
    public static final Set WhiteList = Collections.unmodifiableSet(Set.of(NAME_CONSTRUCTOR, NAME_MAIN));

    public MyClassVisitor(ClassVisitor classVisitor) {
        super(ASM8, classVisitor);
    }

    @Override
    public MethodVisitor visitMethod(int access, String name, String descriptor, String signature, String[] exceptions) {
        MethodVisitor methodVisitor = super.visitMethod(access, name, descriptor, signature, exceptions);
        if(WhiteList.contains(name)) {
            return methodVisitor;
        }
        return new MyMethodVisitor(ASM8, methodVisitor, access, name, descriptor);
    }
}
```

跟書上範例不一樣的是我不想看main，所以遇到constructor和main的話就把原本的methodVisitor丟回去不處理，反之則回傳我們新寫的<code>MyMethodVisitor</code>的instance回去。

``` java
public class MyMethodVisitor extends AdviceAdapter {
    protected MyMethodVisitor(int api, MethodVisitor methodVisitor, int access, String name, String descriptor) {
        super(api, methodVisitor, access, name, descriptor);
    }

    @Override
    protected void onMethodEnter() {
        super.onMethodEnter();
        mv.visitFieldInsn(Opcodes.GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
        mv.visitLdcInsn(">>> enter " + this.getName());
        mv.visitMethodInsn(Opcodes.INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V", false);
    }

    @Override
    protected void onMethodExit(int opcode) {
        super.onMethodExit(opcode);
        mv.visitFieldInsn(Opcodes.GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
        mv.visitLdcInsn(">>> exit " + this.getName());
        mv.visitMethodInsn(Opcodes.INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V", false);
    }
}
```

因為只是要在呼叫前後加一些bytecode進去，就可以利用asm-commons這包裡的<code>AdviceAdapter</code>來幫忙，<code>AdviceAdapter</code>提供了method enter/exit的方法給我們實作就可以達成目的。

這裡就只是很簡單的作出System.out.println(str)這樣的bytecode在原本method bytecode前後。

transformer都寫完就是在agent main裡面加進去

``` java
    public static void premain(String agentArgs, Instrumentation inst) {
        inst.addTransformer(new MyClassTransformer(), true);
    }
```

#### Cleint

跟書上的例子一樣
```
package app;

public class MyTest {
    public static void main(String[] args) {
        new MyTest().foo();
    }

    public void foo() {
        bar1();
        bar2();
    }

    private void bar1() {
        System.out.println("running bar1...");
    }

    private void bar2() {
        System.out.println("running bar2...");
    }
}
```

#### Execute it!!!

執行方式就是在java指令加上-javaagent your_agent_jar，或參考run.xml裡的run-with-javaagent這個target的內容。

``` xml
    <property name="AGENT_JAR" value="agent/target/agent-0.0.1-jar-with-dependencies.jar" />

    <target name="run-with-javaagent">
        <java jar="client/target/client-0.0.1.jar" fork="true">
            <jvmarg value="-javaagent:${AGENT_JAR}" />
        </java>
    </target>
```

執行結果如下

``` Console
D:\lab\java-basic\instrumentation\hello-javaagent>ant -f run.xml run-with-javaagent
Buildfile: D:\lab\java-basic\instrumentation\hello-javaagent\run.xml

run-with-javaagent:
     [java] >>> enter foo
     [java] >>> enter bar1
     [java] running bar1...
     [java] >>> exit bar1
     [java] >>> enter bar2
     [java] running bar2...
     [java] >>> exit bar2
     [java] >>> exit foo

BUILD SUCCESSFUL
Total time: 0 seconds
```


### hello-attach

#### 入口

入口class我們要提供<code>agentmain()</code>這個方法，需要兩個參數

> ``` java
> public static void agentmain(String agentArgs, Instrumentation inst)
> ```

第一個參數是啟動agent時，可以由命令列給予agent的參數，第二個則是JVM會提供agent class使用的Instrumentation的入口。

#### Configuration

就把原本的Premain-Class換成Agent-Class

```
Agent-Class: main.AgentMain
Can-Redefine-Classes: true
Can-Retransform-Classes: true
```

#### Using ASM for manipulate bytecode

<code>agentmain</code>內容，將transform加到instrumentation內之外，還要透過<code>retransformClasses()</code>把目標類別重新transform他們的bytecode。

```
    public static void agentmain(String agentArgs, Instrumentation inst) throws UnmodifiableClassException {
        System.out.println("Agent Running...");
        inst.addTransformer(new MyClassTransformer(), true);
        Class[] classes = inst.getAllLoadedClasses();
        for(Class clz : classes) {
            //System.out.println(clz.getName());
            if(clz.getName().equals("com.example.client.controller.HelloController")) {
                System.out.println("Reloading: " + clz.getName());
                inst.retransformClasses(clz);
                break;
            }
        }
    }
```

後面利用ASM Core API作的事情就跟之前專案類似，就不貼了。

#### Attach

使用attach的方式需要target JVM的process id，可以透過jps查詢或者這裡我使用一個簡單的Spring Boot程式當作target，啟動Spring Boot Application預設會把process id吐到Console裡。

``` java
    public static void main(String[] args) throws IOException, AttachNotSupportedException, AgentLoadException, AgentInitializationException {
        System.out.println("Attach agent to pid");
        String pid = args[0];
        String agent = args[1];
        System.out.println("pid: " + args[0]);
        System.out.println("agent: " + agent);
        VirtualMachine vm = VirtualMachine.attach(pid);
        try {
            vm.loadAgent(agent);
            System.out.println("Done.");
        } finally {
            vm.detach();
        }
    }
```

<code>loadAgent()</code>的路徑是target virtual machine的檔案系統裡的agent jar file路徑。見[JavaDoc](https://docs.oracle.com/en/java/javase/14/docs/api/jdk.attach/com/sun/tools/attach/VirtualMachine.html#loadAgent(java.lang.String,java.lang.String))


#### Cleint

用Spring Boot做一個簡單的hello api當作測試的client。可以觀察到agent執行時，agent相關程式印到Console的log會在client執行的Console印出來。

#### Execute it!!!

attach agent

```
D:\lab\java-basic\instrumentation\hello-attach\attach>java -jar target\attach-0.0.1.jar 17828 D:\lab\java-basic\instrumentation\hello-attach\agent\targe
t\agent-0.0.1-jar-with-dependencies.jar
Attach agent to pid
pid: 17828
agent: D:\lab\java-basic\instrumentation\hello-attach\agent\target\agent-0.0.1-jar-with-dependencies.jar
Done.
```

Srping boot console after attach agent

```
2020-09-14 13:15:00.687  INFO 17828 --- [  restartedMain] com.example.client.ClientApplication     : Started ClientApplication in 2.632 seconds (JVM running for 3.352)
Agent Running...
Reloading: com.example.client.controller.HelloController
<init>
hello
Modify hello()
```

呼叫hello api之後看到server log會印出我們agent添加的程式碼執行的log

```
2020-09-14 13:15:00.687  INFO 17828 --- [  restartedMain] com.example.client.ClientApplication     : Started ClientApplication in 2.632 seconds (JVM running for 3.352)
Agent Running...
Reloading: com.example.client.controller.HelloController
<init>
hello
Modify hello()
2020-09-14 13:16:52.757  INFO 17828 --- [nio-8080-exec-1] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring DispatcherServlet 'dispatcherServlet'
2020-09-14 13:16:52.757  INFO 17828 --- [nio-8080-exec-1] o.s.web.servlet.DispatcherServlet        : Initializing Servlet 'dispatcherServlet'
2020-09-14 13:16:52.770  INFO 17828 --- [nio-8080-exec-1] o.s.web.servlet.DispatcherServlet        : Completed initialization in 13 ms
>>> enter hello
>>> exit hello
```
