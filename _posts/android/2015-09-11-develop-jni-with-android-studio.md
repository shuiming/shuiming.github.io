---
layout: article
title: "使用Android Studio开发JNI程序"
modified:
categories: android
excerpt:
tags: []
image:
  feature:
  teaser:
  thumb:
comments: true
date: 2015-09-11T19:36:50+08:00
---

##JNI简介
JNI是Java Native Interface的缩写，实现Java和C/C++代码之间的通信，互相调用。在Android应用开发中，使用JNI开发主要有一下几种情况：

  - 提高程序运行的效率，比如游戏开发。
  - 增加程序被反编译的难度，防止代码泄漏。
  - 提高代码的重用率，因为iOS和Android都支持C/C++代码，减少重复劳动。

##使用Android Studio开发JNI
Android Studio是google官方建议使用的IDE工具，功能强大，目前最新版1.3.2对C/C++代码的支持也有了很大进步。

###1.声明native方法
只需要在方法名前面加上关键字`native`即可，比如在文件`com/example/myapp/MainActivity.java`文件中：

	native int add(int x, int y);

这样在Java代码中就可以直接调用这个方法了，不过只是声明了，还没有实现。

###2.实现native方法
创建jni目录，路径一般为src/main/jni，因为studio默认从这个路径查找C/C++代码。然后使用`javah`生成JNI风格的C/C++头文件，在命令行下先进入到`module`根目录，默认是`app`，再执行如下命令：

	javah -classpath build/intermediates/classes/debug;<sdk-path>/android.jar -d src/main/jni -force com.example.myapp.MainActivity

就会在jni目录下生成一个头文件，实现头文件中的函数即可。上面的命令中，-classpath指定java class的路径，如果有多个路径，在windows系统下用`;`分隔，linux下面则用`:`分隔。`build/intermediates/classes/debug`是编译生成的目录，所以需要先编译java代码，如果编译的是`release`版本，则把`debug`调换成`release`即可。引用`<sdk-path>/android.jar`是因为MainActivity中import了`Activity`的类，如果import的是AppCompatActivity，那么路径换成`<sdk-path>/extras/android/support...`，由于每个人电脑中的路径不一样，实际的命令也不同。`-d`指定头文件生成的目录，`-force`是强制替换，如果指定路径下已经有同名的头文件，则会被替换掉。最后是java文件的路径，比如`com.example.myapp.MainActivity`就对应`com/example/myapp/MainActivity.java`文件，路径从`package`名称开始，即`com.xxx.xx`

##3.编译so
实现头文件的函数之后，就可以开始编译so库了。首先需要下载[NDK](https://developer.android.com/ndk/downloads/index.html)。对于1.2.3及以下版本的gradle，需要在工程根目录的`local.properties`中指定NDK的路径：

	sdk.dir=D\:\\Android\\sdk
	ndk.dir=D\:\\Android\\android-ndk-r10e

配置好NDK之后，就可以编译了，方法和编译普通的Android APP一样，C/C++文件会自动编译成libapp.so并打包到最终的apk中。程序要运行，还差最后一步，就是在java文件中加载so库，对于本文演示的工程，在MainActivity中增加以下代码：

	static {
        System.loadLibrary("app");
    }

可以看到，加载的时候需要去掉`lib`前缀以及文件后缀。
##4.自定义so名称
有时候我们可能需要编译多个so,而且不想命名为libapp.so，那么就需要用到`Android.mk`和`Application.mk`了，在`jni`目录下创建这两个文件，内容如下：

`Application.mk`:

	APP_ABI := armeabi armeabi-v7a arm64-v8a x86
	APP_PLATFORM := android-15

`Android.mk`:

	LOCAL_PATH := $(call my-dir)

	include $(CLEAR_VARS)
	
	LOCAL_MODULE := mymodule
	LOCAL_LDLIBS := -llog
	
	LOCAL_SRC_FILES := \
		com_example_myapp_MainActivity.cpp \

	include $(BUILD_SHARED_LIBRARY)

以上参数含义可参考[Application.mk](https://developer.android.com/ndk/guides/application_mk.html)和[Android.mk](https://developer.android.com/ndk/guides/android_mk.html)，然后根据实际情况自定义。

然后在app的`build.gradle`文件中，加入以下代码：

	sourceSets.main {
        jniLibs.srcDir 'src/main/libs' //set libs as .so's location instead of jniLibs
        jni.srcDirs = [] //disable automatic ndk-build call with auto-generated Android.mk
    }

之后就可以在命令行中执行`ndk-build`命令，so文件会编译到`src/main/libs`目录。需要注意的是，如果修改了C/C++代码，需要再次手动执行`ndk-build`，否则apk打包的so可能是旧的。

当然，也可以定义一个gradle的task：

	task ndkBuild(type: Exec) {
        String osName = System.getProperty("os.name").toLowerCase();
        if (osName.contains("windows")) {
            commandLine 'ndk-build.cmd', '-C', file('src/main').absolutePath
        } else {
            commandLine 'ndk-build', '-C', file('src/main').absolutePath
        }
    }

