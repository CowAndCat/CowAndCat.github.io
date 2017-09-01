---
layout: post
title: Android学习
category: android
comments: false
---

1. 先看博客

	1. [http://stormzhang.com/](http://stormzhang.com/)
	2. [老罗的Android之旅](http://blog.csdn.net/luoshengyang)


2. Android调试  
	Android真机 优先于 Android模拟器, 模拟器首推Genymotion(最快的安卓模拟器).  	
	- [一个强大的Android模拟器Genymotion](http://www.stormzhang.com/android/2013/12/04/android-genymotion/)  
	- [Android Studio如何安装插件](http://blog.csdn.net/hyr83960944/article/details/35987721)
	- [Android Studio如何集成Genymotion](http://blog.csdn.net/hyr83960944/article/details/37900383)

3. Android studio的使用
	1. 调节字体：File->Settings->Editor->Colors&Fonts->Font 可以设置字体和大小（Courier New, 14)
	2. 快捷键：File->Settings->keymap(支持多种IDE，可以选择Eclipse）
	3. 自动导包：这是一个很有用的设置，我们只有每次引用一些类的时候必须要导包，而Studio可以通过设置自动导包.   
    File->Settings -> Editor -> General->Auto Import -> Java 把三个选项勾上就OK了。

4. Android项目结构
	- **/src**
	- **/gen** ：这个也是源代码目录，但是它只包含android平台自动生成的Java源代码文件。
	- **/Android {版本号}**  ：
	 这个目录包含了项目需要的库文件（Jar文件）。
	- **/res**：这个目录包含了android应用所需要的所有外部资源文件（图片、数据文件等）这些外部资源是要在android应用中引用的。它包含三个子目录  
		- /res/drawable：这个目录包含所有的图片。如果你打算在android应用中包含一个图片或者图标，就应该把它们放在这个目录。
		- /res/layout ：这个目录包含了项目中用到的UI layout，这些layout是以xml形式保存的
		- /res/Values：这个目录也包含了一些xml文件，但主要是应用中要引用的key-value对。这些XML文件声明了数组（Array）、颜色（color）、度量(Dimension)、字符串。之所以把这些东西分别放在单独的xml文件中主要是考虑到这些值能够在不更改源代码的情况下用于多语言环境。例如，根据用户语言的不同应用程序中的信息可以有多种语言版本。

	- **/assets**：这个目录和res包含的xml文件差不多，也是应用中引用到的一些外部资源。但主要区别在于这些资源是以原始格式保存，且只能用编程方式读取。
	- **AndroidManifest.xml**：这个XML文件包含了android应用中的元信息，是每个android项目中的重要文件。它包含了activity（行为）、view（视图）、service（服务）之类的信息，以及运行这个android应用程序需要的用户权限列表，同时也详细描述了android应用的项目结构。

5. 开源项目  
	- Github: [Collect and classify android open source projects](https://github.com/Trinea/android-open-project)
	- [Android 开源库获取途径整理](http://www.trinea.cn/android/android-open-project-summary/)
	- [http://p.codekk.com/](http://p.codekk.com/)
	- [完整开源APP](http://www.jianshu.com/p/31c0b42820fb)
	- [gank.io Android 客户端](http://gank.io/download)
	- [JuheNews](https://github.com/onlyloveyd/JuheNews)

10. [Gradle基础](http://stormzhang.com/devtools/2014/12/18/android-studio-tutorial4/)  
	Gradle是一种依赖管理工具，基于Groovy语言，面向Java应用为主，它抛弃了基于XML的各种繁琐配置，取而代之的是一种基于Groovy的内部领域特定（DSL）语言。

   一份完整的gradle 配置示例：

	apply plugin: 'com.android.application'

	def releaseTime() {
    	return new Date().format("yyyy-MM-dd", TimeZone.getTimeZone("UTC"))
	}

	android {
	    compileSdkVersion 21
	    buildToolsVersion '21.1.2'

	    defaultConfig {
	        applicationId "com.boohee.*"
	        minSdkVersion 14
	        targetSdkVersion 21
	        versionCode 1
	        versionName "1.0"

	        // dex突破65535的限制
	        multiDexEnabled true
	        // 默认是umeng的渠道
	        manifestPlaceholders = [UMENG_CHANNEL_VALUE: "umeng"]
	    }

	    lintOptions {
	        abortOnError false
	    }

	    signingConfigs {
	        debug {
	            // No debug config
	        }

	        release {
	            storeFile file("../yourapp.keystore")
	            storePassword "your password"
	            keyAlias "your alias"
	            keyPassword "your password"
	        }
	    }

	    buildTypes {
	        debug {
	            // 显示Log
	            buildConfigField "boolean", "LOG_DEBUG", "true"

	            versionNameSuffix "-debug"
	            minifyEnabled false
	            zipAlignEnabled false
	            shrinkResources false
	            signingConfig signingConfigs.debug
	        }

	        release {
	            // 不显示Log
	            buildConfigField "boolean", "LOG_DEBUG", "false"

	            minifyEnabled true
	            zipAlignEnabled true
	            // 移除无用的resource文件
	            shrinkResources true
	            proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
	            signingConfig signingConfigs.release

	            applicationVariants.all { variant ->
	                variant.outputs.each { output ->
	                    def outputFile = output.outputFile
	                    if (outputFile != null && outputFile.name.endsWith('.apk')) {
	                    	// 输出apk名称为boohee_v1.0_2015-01-15_wandoujia.apk
	                        def fileName = "boohee_v${defaultConfig.versionName}_${releaseTime()}_${variant.productFlavors[0].name}.apk"
	                        output.outputFile = new File(outputFile.parent, fileName)
	                    }
	                }
	            }
	        }
	    }

	    // 友盟多渠道打包
	    productFlavors {
	        wandoujia {}
	        _360 {}
	        baidu {}
	        xiaomi {}
	        tencent {}
	        taobao {}
	        ...
	    }

	    productFlavors.all { flavor ->
	        flavor.manifestPlaceholders = [UMENG_CHANNEL_VALUE: name]
	    }
	}

	dependencies {
	    compile fileTree(dir: 'libs', include: ['*.jar'])
	    compile 'com.android.support:support-v4:21.0.3'
	    compile 'com.jakewharton:butterknife:6.0.0'
	    ...
	}
