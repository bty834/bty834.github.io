---
title: 手写Maven插件生成MyBatis代码
categories: [编程,plugin]
tags: [maven plugin]
---


代码链接：[codegen-maven-plugin](https://gitee.com/bao-tingyu/codegen-maven-plugin)

在使用MyBatis时，需要根据库表结构编写一些通用的Mapper interface、XML、Entity，这些重复操作可以通过代码生成器自动生成，大大提高开发效率。
目前，代码生成分为两种方式：
- 模版引擎：如 `velocity`
- 代码直接生成：如 `javapoet`

本文使用 [javapoet](https://github.com/square/javapoet)直接生成Java代码，使用[jdom2](http://jdom.org/)生成XML文件。

## Maven 插件
自定义Maven插件
可参考[Plugin Developers Centre](https://maven.apache.org/plugin-developers/index.html)

Maven内置以下3个生命周期，每个生命周期（Lifecycle）有一个或多个阶段（Phase），每个阶段可以注册多个Plugin。
- default : handles your project deployment
- clean : handles project cleaning
- site : handles the creation of your project's web site

其中，default生成周期常用，包含以下阶段，按顺序执行：
1. `validate` - validate the project is correct and all necessary information is available
2. `compile` - compile the source code of the project
3. `test` - test the compiled source code using a suitable unit testing framework. These tests should not require the code be packaged or deployed
4. `package` - take the compiled code and package it in its distributable format, such as a JAR.
5. `verify` - run any checks on results of integration tests to ensure quality criteria are met
6. `install` - install the package into the local repository, for use as a dependency in other projects locally
7. `deploy` - done in the build environment, copies the final package to the remote repository for sharing with other developers and projects.

如果单独执行某个Phase，会执行改Phase之前所有的Phases。

自定义插件套路如下：

```java
// 选定默认Phase，可在<build>中覆盖
@Mojo(name = "codegen", defaultPhase = LifecyclePhase.GENERATE_SOURCES)
public class SQLTableGenMojo extends AbstractMojo {

    @Parameter(defaultValue = "${project}", required = true, readonly = true)
    private MavenProject project;

    @Parameter(property = "skip", defaultValue = "false")
    private boolean skip;

    // 自定义xml项
    @Parameter(property = "absoluteFilePath", required = true, readonly = true)
    private String absoluteFilePath;


    @Override
    public void execute() throws MojoExecutionException, MojoFailureException {
        if (skip) {
            getLog().info("codegen is skipped!");
            return;
        }
        if (StringUtils.isBlank(absoluteFilePath) || !Files.exists(Paths.get(absoluteFilePath))) {
        	// 异常
            throw new MojoExecutionException("Failed to generate code: config file does not exist");
        }
        // 日志通过getLog()获取
        getLog().info("codegen-maven-plugin for " + project.getName() + " starting!");
        doExecute(project.getBasedir().getAbsolutePath(),absoluteFilePath);
    }

    public void doExecute(String baseAbsoluteDir,String filePath) throws MojoExecutionException {
       // ...

    }

   
}
```

## javapoet生成Java源代码
javapoet通过Fluent api的方式构建代码：

```java
        ClassName criterion = ClassName.get(configProperties.getMapperInterfaceGenPkg(),"Criterion");


TypeSpec criterion = TypeSpec.classBuilder("Criterion")
				// 加annotation(也可以使用AnnotationSpec构建)
                .addAnnotation(Data.class)
              
              	// List生成方式使用ParameterizedTypeName
                // private List<Criterion> criteria  
                .addField(FieldSpec.builder(ParameterizedTypeName.get(ClassName.get(List.class),criterion),"criteria",Modifier.PRIVATE).build())
                // private Object value;
                .addField(FieldSpec.builder(Object.class, "value", Modifier.PRIVATE).build())
               // private boolean noValue;
                .addField(FieldSpec.builder(TypeName.BOOLEAN, "noValue", Modifier.PRIVATE).build())
        		// 添加构造器
                .addMethod(MethodSpec.constructorBuilder()
                        .addModifiers(Modifier.PUBLIC)
                        .addParameter(ParameterSpec.builder(String.class, "condition").build())
                        .addParameter(ParameterSpec.builder(Object.class, "value").build())
                        // addStatement最后不加分号
                        .addStatement("System.out.println(123);\n this(condition, value, null)")
                        .build()
                )
                // private void addCriterion(String property){...}
               .addMethod(MethodSpec.methodBuilder("addCriterion")
                        .addModifiers(Modifier.PRIVATE)
                        .returns(TypeName.VOID)
                        .addParameter(String.class,"property")
                        // $T表示类型占位符
                        // $L for Literals
                        // $S for Strings
                        // $T for Types
                        // $N for Names
                        .addStatement("    if (value == null) {\n"
                                + "    throw new $T(\"Value for condition cannot be null \");\n"
                                + "}\n"
                                + "criteria.add(new $T(condition,value));",RuntimeException.class,criterion)
                        .build())
                ).build();
```

## JDom2生成XML

```java
Document xml = new Document();

        DocType mybatisDocType = new DocType("mapper", "-//mybatis.org//DTD Mapper 3.0//EN",
                "http://mybatis.org/dtd/mybatis-3-mapper.dtd");
        xml.setDocType(mybatisDocType);

        Comment comment = new Comment("Auto generated by codegen-maven-plugin @author:baotingyu " + LocalDateTime.now());
        xml.addContent(comment);

        Element mapper = new Element("mapper");
        // <mapper namespace="xxx">
        mapper.setAttribute("namespace", configProperties.getMapperInterfaceGenPkg()+"."+xmlName);
        xml.addContent(mapper);
```

