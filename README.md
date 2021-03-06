## 多渠道打包之动态修改App名称，图标，applicationId，版本号，添加资源
近来公司有需求，同一套代码，要打包N套APP，而且这些APP的软件名称，软件图标，applicationId，版本号，甚至主页都不一样。之前都是单次修改，单次打包，可随着需求越来越多，需要打的包也会越来越多，单次打包费时费力，很明显已经不再适合，于是研究了一下，使用gradle成功实现了需要的功能，打包过程也变的更为简单。

gradle是一个基于Apache Ant和Apache Maven概念的项目自动化建构工具。他可以帮助我们轻松实现多渠道打包的功能。

- **效果图**

![多渠道打包.gif](http://upload-images.jianshu.io/upload_images/2761423-45b3ea86b630ad7e.gif?imageMogr2/auto-orient/strip)

- **项目结构图**

![项目结构.png](http://upload-images.jianshu.io/upload_images/2761423-0ac8db9394a40b44.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- **项目结构中build.gradle的具体内容**

```java
apply plugin: 'com.android.application'

//打包时间
def releaseTime() {
    return new Date().format("yyyy-MM-dd", TimeZone.getTimeZone("UTC"))
}

//获取local.properties的内容
Properties properties = new Properties()
properties.load(project.rootProject.file('local.properties').newDataInputStream())

android {
    compileSdkVersion 23
    buildToolsVersion "23.0.3"

// 使用签名文件进行签名的两种方式

//    //第一种：使用gradle直接签名打包
//    signingConfigs {
//        config {
//            storeFile file('keyTest.jks')
//            storePassword '123456'
//            keyAlias 'HomeKey'
//            keyPassword '123456'
//        }
//    }
    //第二种：为了保护签名文件，把它放在local.properties中并在版本库中排除
    // ，不把这些信息写入到版本库中（注意，此种方式签名文件中不能有中文）
    signingConfigs {
        config {
            storeFile file(properties.getProperty("keystroe_storeFile"))
            storePassword properties.getProperty("keystroe_storePassword")
            keyAlias properties.getProperty("keystroe_keyAlias")
            keyPassword properties.getProperty("keystroe_keyPassword")
        }
    }

    // 默认配置
    defaultConfig {
        minSdkVersion 16
        targetSdkVersion 23
        versionCode 1
        versionName "1.0.1"
    }

    // 多渠道 的不同配置
    productFlavors {
        baidu{
            // 每个环境包名不同
            applicationId "com.shi.androidstudio.multichannel.baidu"
            // 动态添加 string.xml 字段；
            // 注意，这里是添加，在 string.xml 不能有这个字段，会重名！！！
            resValue "string", "app_name", "百度"
            resValue "bool", "auto_updates", 'false'
            // 动态修改 常量 字段
            buildConfigField "String", "ENVIRONMENT", '"我是百度首页"'
            // 修改 AndroidManifest.xml 里渠道变量
            manifestPlaceholders = [UMENG_CHANNEL_VALUE: "baidu"]
        }
        qq{
            applicationId "com.shi.androidstudio.multichannel.qq"

            resValue "string", "app_name", "腾讯"
            resValue "bool", "auto_updates", 'true'

            buildConfigField "String", "ENVIRONMENT", '"我是腾讯首页"'

            manifestPlaceholders = [UMENG_CHANNEL_VALUE: "qq"]
        }
        xiaomi{
            applicationId "com.shi.androidstudio.multichannel.xiaomi"

            resValue "string", "app_name", "小米"
            resValue "bool", "auto_updates", 'true'
            resValue "drawable", "isrRank", 'true'

            buildConfigField "String", "ENVIRONMENT", '"我是小米首页"'

            manifestPlaceholders = [UMENG_CHANNEL_VALUE: "xiaomi"]
        }
    }

    //移除lint检测的error
    lintOptions {
        abortOnError false
    }

    buildTypes {
        debug {
            // debug模式下，显示log
            buildConfigField("boolean", "LOG_DEBUG", "true")

            //为已经存在的applicationId添加后缀
            applicationIdSuffix ".debug"
            // 为版本名添加后缀
            versionNameSuffix "-debug"
            // 不开启混淆
            minifyEnabled false
            // 不开启ZipAlign优化
            zipAlignEnabled false
            // 不移除无用的resource文件
            shrinkResources false
            // 使用config签名
            signingConfig signingConfigs.config

        }
        release {
            // release模式下，不显示log
            buildConfigField("boolean", "LOG_DEBUG", "false")
            // 为版本名添加后缀
            versionNameSuffix "-relase"
            // 不开启混淆
            minifyEnabled false
            // 开启ZipAlign优化
            zipAlignEnabled true
            // 移除无用的resource文件
            shrinkResources true
            // 使用config签名
//            signingConfig signingConfigs.config
            // 混淆文件位置
            proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'

//            // 批量打包
            applicationVariants.all { variant ->
                variant.outputs.each { output ->
                    def outputFile = output.outputFile
                    if (outputFile != null && outputFile.name.endsWith('.apk')) {
                        //输出apk名称为：渠道名_版本名_时间.apk
                        def fileName = "${variant.productFlavors[0].name}_v${defaultConfig.versionName}_${releaseTime()}.apk"
                        output.outputFile = new File(outputFile.parent, fileName)
                    }
                }
            }

        }
    }
}

//项目依赖
dependencies {
    compile fileTree(dir: 'libs', include: ['*.jar'])
    testCompile 'junit:junit:4.12'
    compile 'com.android.support:appcompat-v7:23.3.0'
}
```

- **项目结构中local.properties的具体内容**

```java
## This file is automatically generated by Android Studio.
# Do not modify this file -- YOUR CHANGES WILL BE ERASED!
#
# This file should *NOT* be checked into Version Control Systems,
# as it contains information specific to your local configuration.
#
# Location of the SDK. This is only used by Gradle.
# For customization when using a Version Control System, please read the
# header note.

sdk.dir=D\:\\Android_Studio\\SDK

#对应自己实际的证书路径和名字，在这里由于签名文件是放在app目录下，因为没有写绝对路径。
keystroe_storeFile=keyTest.jks
keystroe_storePassword=123456
keystroe_keyAlias=HomeKey
keystroe_keyPassword=123456
```
看完build.gradle和local.properties的具体内容之后，我们再来挑选几个地方来具体说一下。
### 一. signingConfigs
在signingConfigs中主要是为打包配置签名文件具体信息的，这里我使用了两种方式，第一种方式把签名文件的位置，storePassword ，keyAlias，keyPassword 等具体内容都直接写在其中，然后使用gradle进行打包，第二种是通过通过使用local.properties文件来间接加载签名文件的具体信息。一般我们更倾向于第二种方法，这样有助于保护我们的签名文件（在local.properties中不能有中文）。

### 二. productFlavors
不同渠道的设置基本都是在 productFlavors 里设置的，在里面想要添加多少个渠道都可以。
##### 修改app名称
```java
resValue "string", "app_name", "腾讯"
resValue "bool", "auto_updates", 'true'
```
通过resValue 我们可以在在 string.xml 里面添加了一个新的字段app_name，由于是添加，所以原来的string.xml 文件中不能存在app_name字段，否则会报错。
当然我们还可以添加布尔类型，还可以为color.xml、dimen.xml添加一些我们需要的字段。
##### 修改app图标
当我们在productFlavors 中添加了不同渠道环境名称之后，我们还可以mian文件夹同层级中建立和baidu，qq，xiaomi名称对应的文件夹，并放入特定的图标文件，当然我们还可以放入其他资源文件，甚至AndroidManifest.xml都可以放入，Gradle在构建应用时，会优先使用flavor所属dataSet中的同名资源。所以，在flavor的dataSet中添加同名的资源文件，会覆盖默认的资源文件，这样就能达到不同环境不同软件图标的功能。
![修改软件图标.png](http://upload-images.jianshu.io/upload_images/2761423-3aa0b6c4326db3a7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

简单介绍就到这里了，做个笔记方便以后使用，如果"不小心"帮到别人了当然也是极好的了。
[最后附上github上的项目地址](https://github.com/AFinalStone/MultiChannel-master)以及[整个demo下载地址](http://download.csdn.net/detail/abc6368765/9650523)