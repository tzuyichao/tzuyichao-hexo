---
title: Javac (JC) Parse Java Source
date: 2020-09-05 23:26:48
tags:
  - java
  - javac
---
再提一次這篇用到的類別大多數在他們source的javadoc都有寫到

> This is NOT part of any supported API. If you write code that depends on this, you do so at your own risk.  This code and its internal interfaces are subject to change or deletion without notice.

Java沒有使用flex這些工具產生lexer或bison產生parser，選擇自己用java打造lexer和parser。

### Scanner

<code>com.sun.tools.javac.parser.Scanner</code>這就是Java的Tokenizer。以下的例子是修改自dive into jvm bytecode一書，修改的原因是因為我練習時使用Java 14跟作者用的應該是不同版本，因此就跟javadoc警語一樣，使用者風險請自行負責。


``` java
package compile;

import com.sun.tools.javac.parser.Scanner;
import com.sun.tools.javac.parser.ScannerFactory;
import com.sun.tools.javac.parser.Tokens;
import com.sun.tools.javac.util.Context;

/**
 * from dive into jvm bytecode
 */
public class ParseTest {
    public static void main(String[] args) {
        ScannerFactory scannerFactory = ScannerFactory.instance(new Context());
        Scanner scanner = scannerFactory.newScanner("int k = i + j;", false);

        scanner.nextToken();
        Tokens.Token token = scanner.token();
        while(token.kind != Tokens.TokenKind.EOF) {
            if(token.kind == Tokens.TokenKind.IDENTIFIER) {
                System.out.print(token.name() + " - ");
            }
            System.out.println(token.kind);
            scanner.nextToken();
            token = scanner.token();;
        }
    }
}
```

程式執行結果如下

```
int
k - token.identifier
=
i - token.identifier
+
j - token.identifier
';'
```

### JavaCompiler

以下的例子是使用<code>JavaCompiler</code>寫個的程式parse簡單的java source code，走訪AST取得所有的variable。


``` java
package compile;

import com.sun.source.tree.Tree;
import com.sun.tools.javac.main.JavaCompiler;
import com.sun.tools.javac.main.Option;
import com.sun.tools.javac.tree.JCTree;
import com.sun.tools.javac.tree.TreeTranslator;
import com.sun.tools.javac.util.Context;
import com.sun.tools.javac.util.Options;

public class JavaCompilerTest {
    public static void main(String[] args) {
        Context context = new Context();
        Options.instance(context).put(Option.ENCODING, "UTF-8");

        JavaCompiler compiler = new JavaCompiler(context);
        compiler.genEndPos = true;
        compiler.keepComments = true;

        @SuppressWarnings("deprecation") JCTree.JCCompilationUnit compilationUnit = compiler.parse("src/main/java/client/User.java");

        compilationUnit.defs.stream()
                .forEach(jcTree -> {
                    System.out.println(jcTree.toString());
                    System.out.println(jcTree.getKind());
                    listVariable(jcTree);
                });
    }

    private static void listVariable(JCTree tree) {
        tree.accept(new TreeTranslator() {
            @Override
            public void visitClassDef(JCTree.JCClassDecl classDecl) {
                classDecl.defs.stream()
                        .filter(it -> it.getKind().equals(Tree.Kind.VARIABLE))
                        .map(it -> (JCTree.JCVariableDecl) it)
                        .forEach(it -> {
                            System.out.println(it.getName() + ": " + it.getKind() + ": " + it.getType());
                        });
            }
        });
    }
}
```

因為<code>@SuppressWarnings("deprecation")</code>的礙眼，參考<code>SimpleJavaFileObject</code>自己實作一個<code>JavaFileObject</code>而做了第二個例子。因為很簡單，我們無法使用<code>SimpleJavaFileObject</code>的建構子。這個類別的建構子是protected，而且看起來沒有static factory method可以簡單的方式產生物件，和使用建構子的類別的方法看起來大多是private method，加上<code>JavaFileObject</code>好像沒很複雜，應該可以做一個簡單版本來測試看看parse tree。

``` java
package compile;

import javax.lang.model.element.Modifier;
import javax.lang.model.element.NestingKind;
import javax.tools.JavaFileObject;
import java.io.*;
import java.net.URI;
import java.nio.CharBuffer;
import java.nio.file.Files;
import java.nio.file.Path;
import java.nio.file.Paths;
import java.util.Objects;
import java.util.stream.Collectors;
import java.util.stream.Stream;

public class MySimpleJavaFileObject implements JavaFileObject {
    private URI uri;
    private Kind kind;

    public MySimpleJavaFileObject(URI uri) {
        Objects.requireNonNull(uri);
        if (uri.getPath() == null) {
            throw new IllegalArgumentException("URI must have a path: " + uri);
        }
        if (!uri.getPath().toLowerCase().endsWith(Kind.SOURCE.extension)) {
            throw new UnsupportedOperationException("Only support SOURCE kind");
        }
        this.uri = uri;
        this.kind = Kind.SOURCE;
    }

    @Override
    public Kind getKind() {
        return this.kind;
    }

    @Override
    public boolean isNameCompatible(String simpleName, Kind kind) {
        String baseName = simpleName + kind.extension;
        return kind.equals(getKind())
                && (baseName.equals(toUri().getPath())
                || toUri().getPath().endsWith("/" + baseName));
    }

    @Override
    public NestingKind getNestingKind() {
        return null;
    }

    @Override
    public Modifier getAccessLevel() {
        return null;
    }

    @Override
    public URI toUri() {
        return uri;
    }

    @Override
    public String getName() {
        return toUri().getPath();
    }

    @Override
    public InputStream openInputStream() throws IOException {
        throw new UnsupportedOperationException();
    }

    @Override
    public OutputStream openOutputStream() throws IOException {
        throw new UnsupportedOperationException();
    }

    @Override
    public Reader openReader(boolean ignoreEncodingErrors) throws IOException {
        CharSequence charContent = getCharContent(ignoreEncodingErrors);
        if (charContent == null)
            throw new UnsupportedOperationException();
        if (charContent instanceof CharBuffer) {
            CharBuffer buffer = (CharBuffer)charContent;
            if (buffer.hasArray())
                return new CharArrayReader(buffer.array());
        }
        return new StringReader(charContent.toString());
    }

    @Override
    public CharSequence getCharContent(boolean ignoreEncodingErrors) throws IOException {
        Path path = Paths.get(this.uri);

        Stream<String> lines = Files.lines(path);
        String data = lines.collect(Collectors.joining("\n"));
        lines.close();
        return data;
    }

    @Override
    public Writer openWriter() throws IOException {
        return new OutputStreamWriter(openOutputStream());
    }

    @Override
    public long getLastModified() {
        return 0;
    }

    @Override
    public boolean delete() {
        return false;
    }
}
``` 

接著就可以寫第二個測試，跟第一個只有差一行就不貼全部程式碼

``` java
JCTree.JCCompilationUnit compilationUnit = compiler.parse(new MySimpleJavaFileObject(URI.create("file:///D:/lab/learning-jvm/src/main/java/client/User.java")));
```

忽略把User.java的<code>JCTree</code>印出來，程式執行結果如下
```
CLASS
id: VARIABLE: Long
name: VARIABLE: String
```
