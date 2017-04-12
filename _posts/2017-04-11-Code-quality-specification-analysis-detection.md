---
layout: post
title: 视频直播项目总结之代码质量分析与检测
category: 技术
comments: false
---

<br/>
## Java&Android code style guide
<br/>

<br/>
## AndroidStudio-Inspect Code

<br/>

<br/>
## Error-prone ##

####参考[Error-prone,Google出品的Java和Android Bug分析利器](http://droidyue.com/blog/2017/04/09/error-prone-tool-for-java-and-android/). 

对参考博文的几点补充：<br/>
1. “开启/关闭部分检查”中，类似如下的配置Error-prone参数的代码段，应是位于项目根目录--待编译Moudle--build.gradle文件-->buildscript{}内

```java
tasks.withType(JavaCompile) {
      options.compilerArgs += ['-XepAllErrorsAsWarnings' ]
}
```

2.如果同时设置了“分条件开启error-prone插件”，也即在项目根目录的build.gradle文件中做如下代码处理时

```java
allprojects {
    repositories {
        mavenCentral()
    }
    //如果接受的参数有enableErrorProne则应用插件，否则不应用
    if (project.hasProperty("enableErrorProne")) {
        apply plugin: "net.ltgt.errorprone"
    }
}
```

那么对于注意点1中的代码应修改为

```java
tasks.withType(JavaCompile) {
  if (project.hasProperty("enableErrorProne")) {
      options.compilerArgs += ['-XepAllErrorsAsWarnings' ]
  }
}
```

这样在利用AS的图形化界面执行编译操作时才不会报XepAllErrorsAsWarnings指令找不到的错误。
同时，AS的Terminal窗口中，在项目根目录路径下（作者乃wingdow 7系统），开启插件时执行的完整命令应类似如下

```java
E:\ASWorkSpace\Android>gradlew assembleDirectRelease -PenableErrorProne
```

错误示例中

```java
/.../ErrorProneSample/app/src/main/java/com/example/jishuxiaoheiwu/errorpronesample/MainActivity.java:16:
error: [ArrayToString] Calling toString on an array does not provide useful information
        "hello World".getBytes().toString();
                                         ^
    (see http://errorprone.info/bugpattern/ArrayToString)
```

ArrayToString指的错误类型，对该类错误的详细解释可查看http://errorprone.info/bugpattern/ArrayToString .

在类似该错误的解释页面，ErrorProne给出了如下解释内容：

####错误类型及描述

```java
ArrayToString

Calling toString on an array does not provide useful information
```

####产生的原因


```java
The problem

The toString method on an array will print its identity, such as [I@4488aabb. This is almost never needed. Use Arrays.toString to print a human-readable summary
```


####绕过检查该类型错误时所对应的注解


```java
Suppression

Suppress false positives by adding an @SuppressWarnings("ArrayToString") annotation to the enclosing element.
```

####规避该类错误的正确/错误的代码示范


```java
Positive examples
...

Negative examples
...
```
