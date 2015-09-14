---
layout: article
title: "INSTALL_FAILED_DUPLICATE_PERMISSION错误"
modified: 2015-09-11T15:09:50+08:00
categories: android
comments: true
date: 2015-09-10T21:25:44+08:00
---

##错误原因及解决方法
如果两个APP在manifest中定义了相同名字的`permission`，而两者使用不同的key来签名，那么同一个设备上只能安装其中的一个，在安装第二个的时候就会出现`INSTALL_FAILED_DUPLICATE_PERMISSION`的错误而无法安装。比如在使用google的推送服务GCM时，必须在manifest中定义一个格式为`applicationPackage + ".permission.C2D_MESSAGE"`的`permission`从能收到通知，比如：

	<permission android:name="com.example.gcm.permission.C2D_MESSAGE"
	        android:protectionLevel="signature" />
	<uses-permission android:name="com.example.gcm.permission.C2D_MESSAGE" />

上面代码只是google给出的一个例子，如果直接拷贝到代码中，就有可能遇到重复的问题，因为其他开发者也有可能直接复制这段代码。

如果使用Android Studio开发，可以使用gradle的变量`applicationId`去自动替换前缀。 如果不是gradle自带变量，可以自定义`manifestPlaceHolders`

	<permission android:name="${applicationId}.permission.C2D_MESSAGE"
        	android:protectionLevel="signature" />

##其他
项目中，我们会提供一个`aar`包给第三方开发者，`aar`中包含了一个`service`，核心功能都由`service`完成，在我们的APP中也需要调用这个`service`。我在Android Studio中创建了一个library的模块，我们的APP和第三方APP在项目中都引用这个library，所以`service`的`permission`是在这个library模块的manifest里面定义的，在引用这个库的项目中，gradle会自动把库里面的manifest与当前项目的manifest合并。也就是说，使用这个library的APP中都会包含一个相同的service，而且定义了一个相同的`permission`，这些APP通常也由不同的公司或者开发者开发，自然签名所使用的key也不一样，最终会导致`INSTALL_FAILED_DUPLICATE_PERMISSION`的问题。后面想到说，我们提供的service如果集成到第三方APP，这个service只是提供给他们调用，直接把`android:exported`设置为false就无需设置`permission`了。但问题是，我们自己开发的APP还是希望把这个service提供给其他APP调用，方便升级。总结一个使用场景就是，第三方APP引用我们的aar包，使用`service`提供的功能，第三方APP各自独立，但如果用户安装了我们的APP（以下称为主APP），那么所有的第三方APP会自动调用主APP中的service来调用对应的功能。所以在library模块的manifest中把service的experted设为false，而在主APP的manifest中覆盖这个属性，设置为true，然后加上一个自定义的`permission`。代码如下：

library的`AndroidManifest.xml`


    <manifest xmlns:android="http://schemas.android.com/apk/res/android"
    	package="com.example.aaa" >

	    <uses-permission android:name="com.example.bbb.permission.XXXXX"/>
	
	    <application>
	
	        <service
	            android:name=".service.MyService"
	            android:enabled="true"
	            android:exported="false" >
	        </service>
	
	    </application>

    </manifest>

主APP的`AndroidManifest.xml`

	<manifest xmlns:android="http://schemas.android.com/apk/res/android"
	    xmlns:tools="http://schemas.android.com/tools"
	    package="com.example.bbb" >
	
	    <permission android:name="com.example.bbb.permission.XXXXX" />
	    <uses-permission android:name="com.example.bbb.permission.XXXXX" />
	
	    <application
	        android:icon="@mipmap/ic_launcher"
	        android:label="@string/app_name"
	        android:theme="@style/AppTheme" >
	
	        <service
	            android:name="com.example.aaa.service.MyService"
	            android:enabled="true"
	            android:exported="true"
	            android:permission="com.example.bbb.permission.XXXXX"
	            tools:replace="android:exported">
	        </service>
	    </application>
	
	</manifest>
