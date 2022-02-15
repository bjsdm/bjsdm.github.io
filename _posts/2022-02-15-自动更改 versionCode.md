---
layout: post
title: 自动更改 versionCode
author: 不近视的猫
date: 2022-02-15
categories: blog
tags: []
description: 自动更改 versionCode
---

## 前言

在我们每次发布新包的时候，总是需要更改 versionName 和 versionCode，versionName 是展示给用户看的版本名字，所以，每次发布都需要更改，这个可以理解，但是 versionCode 基本都是自增，那有没有什么方式可以让其在发布的时候自增？

答案当然是可以的，不然这篇文章就不会发布出去了。

<img src="https://img-blog.csdnimg.cn/20210326215055271.jpg" width = "100" >

## 正文

首先，我们在 `gradle.properties` 中定义一个 `APP_VERSION_CODE` ：

<img src="https://img-blog.csdnimg.cn/347fbb1c38ab4c9e81e376097473409e.png" width = "500" >

然后在 `build.gradle` 更改 versionCode 为：

```
versionCode APP_VERSION_CODE as int
```

这样的话，只要我们更改 `APP_VERSION_CODE` 的值，打包出来的 versionCode 就会改变了。但是，这样还需要手动去改，想要自动的话，我们还需要一个 task（示例是在 `build.gradle` 中编写）：

```
task updateVersionCode(){
    doFirst{
        // 获取 gradle.properties 文件
        def gradleProperties = file('../gradle.properties')
        // 读取里面的键值对
        def properties = new Properties()
        properties.load(new FileInputStream(gradleProperties))
        // 获取里面的 APP_VERSION_CODE 进行 +1
        def codeBumped = properties['APP_VERSION_CODE'].toInteger() + 1
        // 重新赋值
        properties['APP_VERSION_CODE'] = codeBumped.toString()
        // 写入
        properties.store(gradleProperties.newWriter(), null)
    }
}
```

至于要把代码放入 doFirst 中，是因为 task 的代码每次编译的时候都会执行的，但是 doFirst 和 doLast 是要等到这个 task 执行的时候才会执行，所以，这里也可以使用 doLast。

这时，我们可以运行这个 task 看下：

为了避免部分同学没有配置 gradle 命令，所以直接使用 Android Studio 图形化的进行演示：

<img src="https://img-blog.csdnimg.cn/c7325018fa9d40fdbc2426234c2fa027.png" width = "300" >
<img src="https://img-blog.csdnimg.cn/5403bdf7b2194739b942989a3415cfab.png" width = "300" >

双击运行。

我们再来查看下 `gradle.properties` 文件：

<img src="https://img-blog.csdnimg.cn/b60a2047741e46c6af2e31844b565e80.png" width = "350" >

<img src="https://img-blog.csdnimg.cn/20210326215055356.jpg" width = "150" >

可以看到 `APP_VERSION_CODE` 是更改成功了！！！

但是，好像有什么不妥？

<img src="https://img-blog.csdnimg.cn/2021042310500417.png" width = "150" >

哦！没错！它直接把 `gradle.properties` 文件内容给全改了，之前的那些注释和顺序都被改动了！

所以，我们不能把 `APP_VERSION_CODE` 放到  `gradle.properties` 中，我们新建一个 `version.properties` 进行存放：

<img src="https://img-blog.csdnimg.cn/c9745d83369d4aa8a0ece5b815ec9468.png" width = "300" >

由于 `build.gradle` 无法直接读取 `version.properties` 文件里面的 `APP_VERSION_CODE`，所以，我们要改成闭包进行读取：

```
versionCode getMyAppVersionCode()
```

```
def getMyAppVersionCode() {  //获取APP版本号
    def gradleProperties = file('../version.properties')
    def properties = new Properties()
    properties.load(new FileInputStream(gradleProperties))
    def codeBumped = properties['APP_VERSION_CODE'].toInteger()
    return codeBumped as int
}
```

`updateVersionCode` task 无需改动：

```
task updateVersionCode(){
    doFirst{
        // 获取 gradle.properties 文件
        def gradleProperties = file('../version.properties')
        // 读取里面的键值对
        def properties = new Properties()
        properties.load(new FileInputStream(gradleProperties))
        // 获取里面的 APP_VERSION_CODE 进行 +1
        def codeBumped = properties['APP_VERSION_CODE'].toInteger() + 1
        // 重新赋值
        properties['APP_VERSION_CODE'] = codeBumped.toString()
        // 写入
        properties.store(gradleProperties.newWriter(), null)
    }
}
```

不过，我希望它能够在每次编译时都自动调用 `updateVersionCode` task，所以，需要添加这行：

```
preBuild.dependsOn(updateVersionCode)
```

好了，万事俱备了，我们把 `APP_VERSION_CODE` 改回为 1，并且双击 `assembleRelease` 试试，看看打出的 apk 的 versionCode 为多少。

至于如何查看 versionCode，可以使用 aapt，当然，还有最简单的方式，直接在 Android Studio 中打开即可：

<img src="https://img-blog.csdnimg.cn/d79a9aeb94fe4d61b6df8465bc0c34b2.png" width = "550" >

em... 怎么 versionCode 还是 1....

我们来看看 `version.properties` ：

<img src="https://img-blog.csdnimg.cn/92e4aaaf83bd4664bb019db9539e17ea.png" width = "350" >

没错啊，确实是有改了。

我们添加日志输出看看，是不是执行顺序出错了：

```
task updateVersionCode(){
    doFirst{
        println("---------->updateVersionCode")
        ...
    }
}

preBuild.dependsOn(updateVersionCode)

def getMyAppVersionCode() {  //获取APP版本号
    println("---------->getMyAppVersionCode")
    ...
}
```

双击 `assembleRelease` ：

<img src="https://img-blog.csdnimg.cn/f6528afc8c0f442bafa0deb3ee06c14a.png" width = "370" >

好吧，原来 getMyAppVersionCode 比 task 还早运行...

至于这个方案的解决方式，我想了很多，也请教了一些大佬，后来决定以一种取巧的方式进行解决，也就是在读取出来的 `APP_VERSION_CODE` 基础上 +1 ：

```
def getMyAppVersionCode() {  //获取APP版本号
    def gradleProperties = file('../version.properties')
    def properties = new Properties()
    properties.load(new FileInputStream(gradleProperties))
    def codeBumped = properties['APP_VERSION_CODE'].toInteger() + 1
    return codeBumped as int
}
```

由此，之后构建出来的包，其 versionCode 就跟 `APP_VERSION_CODE` 保持一致了！

<img src="https://img-blog.csdnimg.cn/20210326215055356.jpg" width = "150" >

不过，现在还是有一个问题，那就是每次构建的时候都会调用 updateVersionCode，但其实，我们更加需要的是每次调用 `assembleRelease` 才会调用 updateVersionCode。

所以，我们需要把这个行进行删除：

```
preBuild.dependsOn(updateVersionCode)
```

添加这段：

```
project.tasks.whenTaskAdded { Task theTask ->
    if (theTask.name == 'assembleRelease') {
        theTask.dependsOn(updateVersionCode)
        theTask.mustRunAfter(updateVersionCode) // updateVersionCode在assembleRelease之前执行
    }
}
```

至此，该功能就全部完成了，以下为 `build.gradle` 的全部源码：

```
plugins {
    id 'com.android.application'
    id 'kotlin-android'
}

android {
    compileSdkVersion 31
    buildToolsVersion "30.0.3"

    defaultConfig {
        applicationId "com.bjsdm.testproject"
        minSdkVersion 16
        targetSdkVersion 30
        versionCode getMyAppVersionCode()
        versionName "1.0"

        testInstrumentationRunner "androidx.test.runner.AndroidJUnitRunner"
    }

    buildTypes {
        release {
            minifyEnabled false
            proguardFiles getDefaultProguardFile('proguard-android-optimize.txt'), 'proguard-rules.pro'
        }
    }
    compileOptions {
        sourceCompatibility JavaVersion.VERSION_1_8
        targetCompatibility JavaVersion.VERSION_1_8
    }
    kotlinOptions {
        jvmTarget = '1.8'
    }
}

dependencies {

    implementation "org.jetbrains.kotlin:kotlin-stdlib:$kotlin_version"
    implementation 'androidx.core:core-ktx:1.3.1'
    implementation 'androidx.appcompat:appcompat:1.2.0'
    implementation 'com.google.android.material:material:1.2.1'
    implementation 'androidx.constraintlayout:constraintlayout:2.0.1'
    testImplementation 'junit:junit:4.+'
    androidTestImplementation 'androidx.test.ext:junit:1.1.2'
    androidTestImplementation 'androidx.test.espresso:espresso-core:3.3.0'
}

task updateVersionCode(){
    doFirst{
        // 获取 gradle.properties 文件
        def gradleProperties = file('../version.properties')
        // 读取里面的键值对
        def properties = new Properties()
        properties.load(new FileInputStream(gradleProperties))
        // 获取里面的 APP_VERSION_CODE 进行 +1
        def codeBumped = properties['APP_VERSION_CODE'].toInteger() + 1
        // 重新赋值
        properties['APP_VERSION_CODE'] = codeBumped.toString()
        // 写入
        properties.store(gradleProperties.newWriter(), null)
    }
}

project.tasks.whenTaskAdded { Task theTask ->
    if (theTask.name == 'assembleRelease') {
        theTask.dependsOn(updateVersionCode)
        theTask.mustRunAfter(updateVersionCode)				// updateVersionCode在assembleRelease之前执行
    }
}

//preBuild.dependsOn(updateVersionCode)


def getMyAppVersionCode() {  //获取APP版本号
    def gradleProperties = file('../version.properties')
    def properties = new Properties()
    properties.load(new FileInputStream(gradleProperties))
    def codeBumped = properties['APP_VERSION_CODE'].toInteger() + 1
    return codeBumped as int
}
```


































