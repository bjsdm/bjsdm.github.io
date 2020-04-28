---
layout: post
title: Could not resolve all artifacts for configuration ':classpath'.
author: 不近视的猫
date: 2020-04-28
categories: blog
tags: []
description: Could not resolve all artifacts for configuration ':classpath'.
---


给新电脑配置开发环境，遇到这个问题

> Could not resolve all artifacts for configuration ':classpath'.

把百度谷歌搜到的方法基本都试过：

- 搭梯子
- 使用阿里云的代理
- 使用mavenLocal()

等等

**都不行！！！**

在近乎绝望的状态下，看到这个网站[https://jfrog.com/jcenter-http/](https://jfrog.com/jcenter-http/)

![403](https://img-blog.csdnimg.cn/20200428140859800.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L20wXzQ2Mjc4OTE4,size_16,color_FFFFFF,t_70#pic_center)

嗯？有戏？

**改了一下，再点击`Sync Now`，还真行了！！！**

先把原先的配置文件发下：

```

buildscript {
    repositories {
        maven {
            url 'https://maven.google.com/'
            name 'Google'
        }
        mavenCentral()
        jcenter(){ url 'http://jcenter.bintray.com/'}
        google()
    }
    dependencies {
        classpath 'com.android.tools.build:gradle:3.5.0'
        classpath 'com.github.dcendents:android-maven-gradle-plugin:1.4.1'
        classpath 'com.novoda:bintray-release:0.9.1'
        classpath 'com.jakewharton:butterknife-gradle-plugin:10.1.0'
        // NOTE: Do not place your application dependencies here; they belong
        // in the individual module build.gradle files
    }
}

allprojects {
    repositories {
        jcenter()
        maven {
            url 'https://maven.google.com/'
            name 'Google'
        }
        mavenCentral()
        jcenter(){ url 'http://jcenter.bintray.com/'}
        maven { url 'https://jitpack.io' }
        google()
    }
}

task clean(type: Delete) {
    delete rootProject.buildDir
}

```

**将`jcenter()`里面的`http`换为`https`即可**

```

buildscript {
    repositories {
        maven {
            url 'https://maven.google.com/'
            name 'Google'
        }
        mavenCentral()
        jcenter(){ url 'https://jcenter.bintray.com/'}
        google()
    }
    dependencies {
        classpath 'com.android.tools.build:gradle:3.5.0'
        classpath 'com.github.dcendents:android-maven-gradle-plugin:1.4.1'
        classpath 'com.novoda:bintray-release:0.9.1'
        classpath 'com.jakewharton:butterknife-gradle-plugin:10.1.0'
        // NOTE: Do not place your application dependencies here; they belong
        // in the individual module build.gradle files
    }
}

allprojects {
    repositories {
        jcenter()
        maven {
            url 'https://maven.google.com/'
            name 'Google'
        }
        mavenCentral()
        jcenter(){ url 'https://jcenter.bintray.com/'}
        maven { url 'https://jitpack.io' }
        google()
    }
}

task clean(type: Delete) {
    delete rootProject.buildDir
}
```
