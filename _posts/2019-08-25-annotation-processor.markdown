---
layout: post
title:  "Creating Annotation Processor To Generate Getters And Setters"
date:   2019-08-25 10:54:08 +0530
categories: java
---

**Generate Your Java Code Don’t Repeat !**

I’m writing this blog because of some scenarios in my work experience that I had some trouble doing, and also i got fascinated by the idea of generating code rather than copy and pasting the same stuff over and over. I will explain some scenarios that we do need to generate our code based on some resources or some criteria to make it productive and time saving,

Use Case #1

If any of you (reading this blog) , have any experience developing rest based api’s with [swagger](https://swagger.io/) documentation, in order to provide examples and descriptions about it you will need to add some annotations to your model classes most like ‘@ApiModeProperty’ to describe elements in your api, so if you have 100 of model classes you need to fill every class with above annotations to make your api more readable and understanding, so it is not an easy task, one way of making this easy is you can create a resource file (excel) and feed it to a program which will do modifications based on a criteria you defined. 

Use Case #2

I think you’ve heard of [lombok](https://projectlombok.org/), When you writing your java code you will need to add getters and setters, and other pretty basic things to your model classes and lombok is a great tool to make it easier to generate vanilla code and save your time. But if you need to generate something that pretty much custom then you are back to square one, so if you know the way to do this you can generate it easily,

So enough of the chat, I think now you know the importance of this and we will start generating some code, I will try to make this short and easy to understand,

First of all you will need to learn some basics of the annotation processing, to do that you can refer to the following articles based on annotation processing,



*   [Java Annotation Processors](https://www.javacodegeeks.com/2015/09/java-annotation-processors.html)

So i’m guessing now you have an idea of annotation processors.

Let’s start with a processor that will generate getters setters on a java pojo. I will create two projects, one is for the annotation processor and other one is for the demo project.

**Sample Processor**

pom.xml

{% highlight xml %}

<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <groupId>com.maxexplode</groupId>
    <artifactId>annotation-processor</artifactId>
    <version>1.0</version>
    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <configuration>
                    <source>8</source>
                    <target>8</target>
                </configuration>
            </plugin>
        </plugins>
    </build>
    <properties>
        <auto-service.version>1.0-rc4</auto-service.version>
    </properties>
    <dependencies>
        <dependency>
            <groupId>com.google.auto.service</groupId>
            <artifactId>auto-service</artifactId>
            <version>${auto-service.version}</version>
            <optional>true</optional>
        </dependency>
        <dependency>
            <groupId>com.github.olivergondza</groupId>
            <artifactId>maven-jdk-tools-wrapper</artifactId>
            <version>0.1</version>
        </dependency>
    </dependencies>
</project>

{% endhighlight %}

In case if you are wondering [auto-service](https://www.baeldung.com/google-autoservice) dependency it is used to generate META-INF/service file automatically for our processor.

And our annotation for the processor will be @SimplePojo

{% highlight java %}

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;
@Target({ElementType.METHOD, ElementType.TYPE, ElementType.CONSTRUCTOR})
@Retention(RetentionPolicy.SOURCE)
public @interface SimplePojo {
}

{% endhighlight %}

Sample processor for our generation

{% highlight java %}

import com.google.auto.service.AutoService;
import com.maxexplode.processor.stereotype.SimplePojo;
import com.sun.source.tree.ClassTree;
import com.sun.source.tree.CompilationUnitTree;
import com.sun.source.util.TreePath;
import com.sun.source.util.TreePathScanner;
import com.sun.source.util.Trees;
import com.sun.tools.javac.code.Symbol;
import com.sun.tools.javac.code.Symtab;
import com.sun.tools.javac.code.TypeTag;
import com.sun.tools.javac.processing.JavacProcessingEnvironment;
import com.sun.tools.javac.tree.JCTree;
import com.sun.tools.javac.tree.JCTree.JCExpression;
import com.sun.tools.javac.tree.JCTree.JCMethodDecl;
import com.sun.tools.javac.tree.JCTree.JCVariableDecl;
import com.sun.tools.javac.tree.TreeMaker;
import com.sun.tools.javac.tree.TreeTranslator;
import com.sun.tools.javac.util.Context;
import com.sun.tools.javac.util.List;
import com.sun.tools.javac.util.Names;
import javax.annotation.processing.AbstractProcessor;
import javax.annotation.processing.Messager;
import javax.annotation.processing.ProcessingEnvironment;
import javax.annotation.processing.Processor;
import javax.annotation.processing.RoundEnvironment;
import javax.annotation.processing.SupportedAnnotationTypes;
import javax.annotation.processing.SupportedSourceVersion;
import javax.lang.model.SourceVersion;
import javax.lang.model.element.Element;
import javax.lang.model.element.TypeElement;
import javax.tools.Diagnostic;
import javax.tools.JavaFileObject;
import java.util.Set;

@SupportedAnnotationTypes(
    "com.maxexplode.processor.stereotype.SimplePojo")
@SupportedSourceVersion(SourceVersion.RELEASE_8)
@AutoService(Processor.class)
public class ResourceProcessor extends AbstractProcessor {

  private ProcessingEnvironment processingEnvironment;
  private Messager MESSENGER = null;
  private Trees trees;
  private TreeMaker treeMaker;
  private Names names;
  private Context context;

  @Override
  public synchronized void init(ProcessingEnvironment processingEnv) {
    super.init(processingEnv);
    this.processingEnvironment = processingEnv;
    MESSENGER = processingEnvironment.getMessager();
    JavacProcessingEnvironment javacProcessingEnvironment = (JavacProcessingEnvironment) processingEnv;
    trees = Trees.instance(processingEnv);
    context = javacProcessingEnvironment.getContext();
    treeMaker = TreeMaker.instance(context);
    names = Names.instance(context);
  }

  @Override
  public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment roundEnv) {
    MESSENGER.printMessage(Diagnostic.Kind.OTHER, "Starting instrumentation ...");
    final TreePathScanner<Object, CompilationUnitTree> scanner =
        new TreePathScanner<Object, CompilationUnitTree>() {
          @Override
          public Trees visitClass(
              final ClassTree classTree,
              final CompilationUnitTree unitTree) {
            if (unitTree instanceof JCTree.JCCompilationUnit) {
              final JCTree.JCCompilationUnit compilationUnit = (JCTree.JCCompilationUnit) unitTree;
              if (compilationUnit.sourcefile.getKind() == JavaFileObject.Kind.SOURCE) {
                compilationUnit.accept(new TreeTranslator() {
                  @Override
                  public void visitClassDef(JCTree.JCClassDecl jcClassDecl) {
                    super.visitClassDef(jcClassDecl);
                    List<JCTree> members = jcClassDecl.getMembers();
                    for (JCTree member : members) {
                      if (member instanceof JCVariableDecl) {
                        JCVariableDecl field = (JCVariableDecl) member;
                        List<JCTree.JCMethodDecl> methods = createMethods(field);
                        for (JCMethodDecl jcMethodDecl : methods) {
                          jcClassDecl.defs = jcClassDecl.defs.prepend(jcMethodDecl);
                        }
                      }
                    }
                  }
                });
              }
            }
            return trees;
          }
        };

    for (final Element element : roundEnv.getElementsAnnotatedWith(SimplePojo.class)) {
      final TreePath path = trees.getPath(element);
      scanner.scan(path, path.getCompilationUnit());
    }

    MESSENGER.printMessage(Diagnostic.Kind.OTHER, String
        .format(
            "Finished instrumenting classes : processed classes",
            roundEnv.getRootElements().size()));

    return true;
  }

  public List<JCTree.JCMethodDecl> createMethods(JCTree.JCVariableDecl field) {
    JCVariableDecl param = treeMaker.Param(field.getName(), field.vartype.type, null);
    return List.of(
        treeMaker.MethodDef(
            treeMaker.Modifiers(1),
            names.fromString("get".concat(field.getName().toString())),
            (JCExpression) field.getType(),
            List.nil(),
            List.nil(),
            List.nil(),
            treeMaker.Block(1, createGetterForField(field)),
            null),
        treeMaker.MethodDef(
            treeMaker.Modifiers(1),
            names.fromString("set".concat(field.getName().toString())),
            treeMaker.TypeIdent(TypeTag.VOID),
            List.nil(),
            List.of(param),
            List.nil(),
            treeMaker.Block(1, createSetterForField(
                field,
                param)),
            null)
    );
  }

  public List<JCTree.JCStatement> createGetterForField(JCTree.JCVariableDecl field) {
    return List.of(treeMaker.Return((treeMaker.Ident(field.getName()))));
  }
  
  public List<JCTree.JCStatement> createSetterForField(
      JCTree.JCVariableDecl field, JCTree.JCVariableDecl field1) {
    return List.of(treeMaker.Assignment(field.sym, treeMaker.Assign(
        treeMaker.Ident(field),
        treeMaker.Ident(field1))), treeMaker.Return(null));
  }
}

{% endhighlight %}
