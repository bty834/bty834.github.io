---
title: IDEA插件开发：自动生成setter
categories: [编程,plugin]
tags: [idea plugin]
---

我的Intellij插件主页：[Bao Tingyu](https://plugins.jetbrains.com/vendor/1b0d1834-d798-409c-82c2-adc1d3324dac)

# 背景
在给Java局部变量的实体赋值时，往往有很多setter，一个一个写很麻烦，也会漏掉，因此开发一款插件，可以自动生成局部变量实体的所有setter。

插件效果如下：
![](/assets/2024/05/31/summonSetters.gif)

可以在plugin marketplace 搜索：`Summon Setters`
源码参考：[Summon-all-setters](https://github.com/bty834/Summon-all-setters)

# 开发前
IDEA plugin 通过 Java 或 Kotlin 语言编写，官方目前推荐Kotlin语言，依赖管理使用 Gradle。
插件框架初始化可以手动通过Gradle创建，也可以从[官方的Template](https://github.com/JetBrains/intellij-platform-plugin-template)下载，默认为Kotlin语言。

参考文档：
- [Developing a Plugin](https://plugins.jetbrains.com/docs/intellij/developing-plugins.html)
- [idea插件开发文档](https://www.ideaplugin.com/idea-docs/)

也可以参考开源的插件实现，在 [Intellij Plugin Marketplace](https://plugins.jetbrains.com/)搜索相关功能插件，点开`Source Code`栏（可能没有）

同时IDEA可安装插件开发插件：`Plugin DevKit`。

为了方便查看文件的PSI树形结构，设置IDEA安装目录下的`bin`目录的`idea.properties`文件中的`idea.is.internal=true`，通过主菜单的`Tools`->`View PSI Structure`即可查看。 

## 目录结构
这里使用Github上的intellij-platform-plugin-template，目录结构如下：
```text
.
├── CHANGELOG.md
├── CODE_OF_CONDUCT.md
├── LICENSE
├── README.md
├── build.gradle.kts
├── gradle
│         ├── libs.versions.toml 
│         └── wrapper
│             ├── gradle-wrapper.jar
│             └── gradle-wrapper.properties
├── gradle.properties
├── gradlew
├── gradlew.bat
├── qodana.yml
├── settings.gradle.kts
└── src
    └── main
        ├── kotlin
        │
        └── resources
            └── META-INF
                     ├── plugin.xml
                     └── pluginIcon.svg
```

## 注意事项
开发时，`gradle.properties` 中需要引入相关依赖：

```
...
platformPlugins = com.intellij.java
...
```

同时 `plugin.xml` 中也要设置:
```xml
<!-- Plugin Configuration File. Read more: https://plugins.jetbrains.com/docs/intellij/plugin-configuration-file.html -->
<idea-plugin>
    ...
    <depends>com.intellij.java</depends>

    <description><![CDATA[
        这里填写介绍，不能少于40个字符，同时README.md文件中也要写，不然无法提交到marketPlace
    ]]>
    </description>

    ...
</idea-plugin>

```
README.md:
```
<!-- Plugin description -->
这里填写介绍，不能少于40个字符
<!-- Plugin description end -->
```
同时默认的图标pluginIcon.svg需要替换掉，图标规范参考 [plugin-icon-file](https://plugins.jetbrains.com/docs/intellij/plugin-icon-file.html?from=jetbrains.org#plugin-logo-usages)

# Summon Setters 插件开发

在实施代码开发前，要考虑通过什么方式生成，自定义Action？自定义Extension？两种方式都能实现，参考了市面上的两种实现，发现第二种更直观简单写。

这里我们扩展Intention Extension。Intention Extension即为代码的提示扩展，快捷键`option/alt + enter`。

![img.png](/assets/2024/05/31/img.png)


我们在`plugin.xml`中注册Extension:
```xml
<idea-plugin>
    ...
    <extensions defaultExtensionNs="com.intellij">
        <intentionAction>
            <language>JAVA</language>
            <!-- 生成setters，不带参数 -->
            <className>io.github.bty834.SummonSettersIntentionAction</className>
        </intentionAction>
        <intentionAction>
            <language>JAVA</language>
            <!-- 生成setters，带默认参数 -->
            <className>io.github.bty834.SummonSettersWithDefaultsIntentionAction</className>
        </intentionAction>
    </extensions>
</idea-plugin>
```

`SummonSettersIntentionAction` 需要继承`com.intellij.codeInsight.intention.PsiElementBaseIntentionAction`

看一下需要实现的几个方法：
```kotlin

import com.intellij.codeInsight.intention.HighPriorityAction
import com.intellij.codeInsight.intention.PriorityAction
import com.intellij.codeInsight.intention.PsiElementBaseIntentionAction
import com.intellij.openapi.editor.Editor
import com.intellij.openapi.project.Project
import com.intellij.psi.PsiElement

class MyIntentionExtension : PsiElementBaseIntentionAction(), HighPriorityAction {

    override fun getFamilyName(): String {
        TODO("一组extension共用的名称，我们这里定义的两个extension使用同一个familyName")
    }
    override fun getText(): String {
        TODO("intention展示时的名称")
    }
    
    override fun getPriority(): PriorityAction.Priority {
        // intention的优先级排序，com.intellij.codeInsight.intention.HighPriorityAction接口的方法
        return PriorityAction.Priority.TOP
    }
    
    override fun isAvailable(project: Project, editor: Editor?, element: PsiElement): Boolean {
        TODO("判断当前光标处是否可以展示该intention")
    }
    override fun invoke(project: Project, editor: Editor?, element: PsiElement) {
        TODO("运行intention extension")
    }

}
```

我们要实现`isAvailable`方法，判断能否展示当前intention:

自动生成setter，需要判断当前光标指向的是不是局部变量，且局部变量有含有setter的类：
```kotlin
// 获取当前局部变量的类
fun getLocalVariableContainingClass(psiElement: PsiElement): PsiClass? {
    val psiLocalVar: PsiLocalVariable = PsiTreeUtil.getParentOfType(psiElement, PsiLocalVariable::class.java) ?: return null
    if (psiLocalVar.parent !is PsiDeclarationStatement) {
        return null
    }
    return PsiTypesUtil.getPsiClass(psiLocalVar.type)
}
// 判断当前类是否有setter
fun checkClazzHasValidSetters(psiClass: PsiClass?): Boolean {
    psiClass ?: return false
    if (psiClass.hasAnnotation("lombok.Setter") || psiClass.hasAnnotation("lombok.Data") {
        return true
    }
    val fields: Array<PsiField> = psiClass.allFields
    if (fields.any { it.hasAnnotation("lombok.Setter") }) {
        return true
    }
    if (psiClass.allMethods
                    .filter {
                        it.hasModifierProperty(PsiModifier.PUBLIC)
                                && it.name.startsWith("set")
                                && !it.hasModifierProperty(PsiModifier.STATIC)
                                && !it.hasModifierProperty(PsiModifier.ABSTRACT)
                                && !it.hasModifierProperty(PsiModifier.DEFAULT)
                                && !it.hasModifierProperty(PsiModifier.NATIVE)
                    }
                    .any { it.name.startsWith("set") }) {
                return true
            }
    return false
}
```

满足以上条件，我们开始生成setter代码，大致步骤如下：
1. 定位光标当前的局部变量；
2. 找到当前局部变量的类以及类中的setter，包含手写的setter和lombok的`@Data`和`@Setter`，lombok注解又分为注解在类上和注解在字段上，并且要忽略静态setter方法；
3. 生成代码（包含缩进）并插入当前代码编辑区。

先看一下一个局部变量该有的PSI树形结构：
![img_1.png](/assets/2024/05/31/img_1.png)


```kotlin
override fun invoke(project: Project, editor: Editor?, element: PsiElement) {
    // 先找到PsiLocalVariable类型的父级元素，必须为PsiDeclarationStatement
    val localVariable: PsiLocalVariable =
        PsiTreeUtil.getParentOfType(element, PsiLocalVariable::class.java) ?: return
    // 不是就返回
    if (localVariable.parent !is PsiDeclarationStatement) {
        return
    }
    // 获取局部变量的类
    val psiClass = PsiTypesUtil.getPsiClass(localVariable.type)
    // 获取该类的所有setter函数名
    val setterMethodNames: List<String> = CommonUtil.getSetterMethodNames(psiClass)
    
    // 局部变量的变量名
    val variableName: String = localVariable.name
    
    // 找到代码缩进量：
    val psiDocumentManager = PsiDocumentManager.getInstance(project)
    val containingFile: PsiFile = localVariable.containingFile
    val document = psiDocumentManager.getDocument(containingFile) ?: return
    val indentNum: Int = CommonUtil.getIndentSpaceNumsOfCurrentLine(document, localVariable.parent.textOffset)

    val insertSetterStr: StringBuilder = StringBuilder()
     setterMethodNames.forEach {
        // 缩进
        insertSetterStr.append(" ".repeat(indentNum))
        // setter
        insertSetterStr.append("$variableName.$it();\n")
     }
    
    // 写入编辑区
    document.insertString(localVariable.parent.textOffset + localVariable.parent.textLength + 1, insertSetterStr.toString())
    psiDocumentManager.doPostponedOperationsAndUnblockDocument(document)
    psiDocumentManager.commitDocument(document)
    FileDocumentManager.getInstance().saveDocument(document)
    
}
```







