---
title: Tinker调研
tags: Tinker,热修复,Android
grammar_cjkRuby: true
---

## Tinker与其他热修复框架对比
![enter description here][1]

**总结**：

 1. **阿里的AndFix**作为native解决方案，首先面临的是稳定性与兼容性问题，更重要的是它无法实现类替换，它是需要大量额外的开发成本的；
 2. **美团的Robust**兼容性与成功率最高，但是它与AndFix一样，无法新增变量与类只能用做的bugFix方案，并且尚未开源，不过有参照Robust的install run原理实现的[开源方案][2]，不过较少人关注，实际效果未知。
 3. **百度金融的RocooFix**是Nuwa方案的改良版，增加了lib替换和即时生效支持，但是不支持在windows平台生成补丁，兼容性还有待测试。
 4. **饿了么的Amigo**是非常强大的一个方案，不仅是类替换，lib替换，资源替换都支持，同时也支持新增四大组件，缺点是不支持Android 3.0 ，notification & widget中RemoteViews的自定义布局不支持修改,只支持内容修复。

Amigo官方wiki介绍

> Amigo 原理与 QQZone
> 的方案有些类似，QQZone,Tinker,Nuwa这类方案是通过修改PathClassLoader中的dex实现的，Amigo则是釜底抽薪直接替换ClassLoader。同时进一步实现了
> so 文件、资源文件、四大组件的修复，可以对APP全面进行修复

 5. **微信的Tinker**是各方面都比较优秀的方案，毕竟经过了几亿微信用户的验证。Tinker的优点上图已经很明确了，而存在的缺陷有以下几方面：
 - Tinker不支持修改AndroidManifest.xml，Tinker不支持新增四大组件；  
 - 由于Google Play的开发者条款限制，不建议在GP渠道动态更新代码；
 -  在Android N上，补丁对应用启动时间有轻微的影响；
 -  不支持部分三星android-21机型，加载补丁时会主动抛出"TinkerRuntimeException:checkDexInstall failed"；
 -  由于各个厂商的加固实现并不一致，在1.7.6以及之后的版本，tinker不再支持加固的动态更新；（由于360电子市场必须经过加固应用才能上架，因此可以说tinker无法在360渠道上的apk实现热更新）
 -  对于资源替换，不支持修改remoteView。例如transition动画，notification icon以及桌面图标。
 -   与超级补丁技术一样，不支持即时生效，必须通过重启应用的方式才能生效。
 -    需要给应用开启新的进程才能进行合并，并且很容易因为内存消耗等原因合并失败。
 -    合并时占用额外磁盘空间，对于多DEX的应用来说，如果修改了多个DEX文件，就需要下发多个patch.dex与对应的classes.dex进行合并操作时这种情况会更严重，因此合并过程的失败率也会更高。
 -    接入tinker sdk略微复杂

## Tinker接入

### 1. 添加gradle依赖

 1. 首先在项目的**gradle.properties**文件指定Tinker版本，这样只需修改此处版本号就能更改Tinker版本。加入以下属性：

``` ini
TINKER_VERSION=1.7.7
```

 2. 在项目的**build.gradle**中，添加*tinker-patch-gradle-plugin*的依赖

``` nginx
  classpath "com.tencent.tinker:tinker-patch-gradle-plugin:${TINKER_VERSION}"
```

 3. 然后在app的gradle文件**app/build.gradle**，我们需要添加tinker的库依赖以及apply tinker的gradle插件.

``` nix
    compile("com.tencent.tinker:tinker-android-lib:${TINKER_VERSION}") { changing = true }
    provided("com.tencent.tinker:tinker-android-anno:${TINKER_VERSION}") { changing = true }
```
其中，**tinker-android-anno**用于注解生成application类 
**tinker-android-lib**为tinker的核心库
 4. 在app的gradle文件**app/build.gradle**配置**tinkerPatch task**，下面给出简单的示例：
 //全局信息相关的配置项

``` groovy
tinkerPatch {
    //有问题的apk的地址  准apk包的路径，必须输入，否则会报错
    oldApk = "/Users/littlebyte/AndroidStudioProjects/TInkerTest/app/oldApk/app-debug.apk"
    //
    ignoreWarning = false
    //在运行过程中，需要验证基准apk包与补丁包的签名是否一致，我们是否需要为你签名
    useSign = true
    //编译相关的配置项
    buildConfig {
        //在运行过程中，我们需要验证基准apk包的tinkerId是否等于补丁包的tinkerId。
        // 这个是决定补丁包能运行在哪些基准包上面，一般来说我们可以使用git版本号、versionName等等。
        tinkerId = "1.0"
    }
    //用于生成补丁包中的'package_meta.txt'文件
    packageConfig {
        //onfigField("key", "value"), 默认我们自动从基准安装包与新安装包的Manifest中读取tinkerId,并自动写入configField。
        // 在这里，你可以定义其他的信息，在运行时可以通过TinkerLoadResult.getPackageConfigByName得到相应的数值。
        // 但是建议直接通过修改代码来实现，例如BuildConfig。
//        configField("TINKER_ID", "1.0")
    }
    //dex相关的配置项
    dex {
        //只能是'raw'或者'jar'。
        //对于'raw'模式，我们将会保持输入dex的格式。
        //对于'jar'模式，我们将会把输入dex重新压缩封装到jar。
        // 如果你的minSdkVersion小于14，你必须选择‘jar’模式，而且它更省存储空间，但是验证md5时比'raw'模式耗时()
        dexMode = "jar"
        //需要处理dex路径，支持*、?通配符，必须使用'/'分割。路径是相对安装包的，例如/assets/...
        pattern = ["classes*.dex", "assets/secondary-dex-?.jar"]
        //它定义了哪些类在加载补丁包的时候会用到。这些类是通过Tinker无法修改的类，也是一定要放在main dex的类。
        loader = ["com.tencent.tinker.loader.*", "com.cn21.tinkertest.MyApplication"]
        /**
         * 这里需要定义的类有：
         1. 你自己定义的Application类；
         2. Tinker库中用于加载补丁包的部分类，即com.tencent.tinker.loader.*；
         3. 如果你自定义了TinkerLoader，需要将它以及它引用的所有类也加入loader中；
         4. 其他一些你不希望被更改的类，例如Sample中的BaseBuildInfo类。这里需要注意的是，
         这些类的直接引用类也需要加入到loader中。或者你需要将这个类变成非preverify。
         */
    }
    //lib相关的配置项
    lib {
        //需要处理lib路径，支持*、?通配符，必须使用'/'分割。与dex.pattern一致, 路径是相对安装包的，例如/assets/...
        pattern = ["lib/armeabi/*.so", "lib/arm64-v8a/*.so", "lib/armeabi-v7a/*.so", "lib/mips/*.so", "lib/mips64/*.so", "lib/x86/*.so", "lib/x86_64/*.so"]
    }
    //res相关的配置项
    res {
        pattern = ["res/*", "assets/*", "resources.arsc", "AndroidManifest.xml"]
        //对于修改的资源，如果大于largeModSize，我们将使用bsdiff算法。
        // 这可以降低补丁包的大小，但是会增加合成时的复杂度。默认大小为100kb
        largeModSize = 100
    }
    //7zip路径配置项，执行前提是useSign为true
    sevenZip {
        //例如"com.tencent.mm:SevenZip:1.1.10"，将自动根据机器属性获得对应的7za运行文件，推荐使用。
        zipArtifact = "com.tencent.mm:SevenZip:1.1.10"
    }
}
```
上面只是使用了部分tinker参数，全部参数及含义可参考[tinkerPatch gradle参数官方wiki][3]

### 2. 修改Application类
 1. 修改工程的Application类，使其继承自**DefaultApplicationLike**，然后生成默认的构造方法，并覆盖**onBaseContextAttached**方法，然后添加一个**registerActivityLifecycleCallbacks**方法，同时在自己的Application类上加上以下注解：

``` nix
 @DefaultLifeCycle(application = "com.cn21.tinkertest.MyApplication",
        flags = ShareConstants.TINKER_ENABLE_ALL,
        loadVerifyFlag = false)
```
其中，

 - **application**属性指定的是tinker为我们生成的真正的Application，一般是**包名＋自定义的Application名称**作为名字，其中application属性指定的是tinker为我们生成的真正的Application类，需要注意两点，一是AndroidManifest.xml 中的application节点下的name 属性必须是这个application属性的值。As找不到这个Application报错但不会影响编译成功；二是在**app/build.gradle**文件中的tinkerPatch-dex-loader节点中添加application属性的值（见tinkerPatch gradle配置）。
 - **flags**属性指定tinker可以修复的范围，*TINKER_ENABLE_ALL*是全部都可以修复，还有*TINKER_DEX_AND_LIBRARY*，*TINKER_RESOURCE_MASK*，*TINKER_DEX_MASK*等等，根据名字就可以知道所代表的含义。


以下是完整的自定义Application代码：

``` java
@DefaultLifeCycle(application = "com.cn21.tinkertest.MyApplication",
        flags = ShareConstants.TINKER_ENABLE_ALL,
        loadVerifyFlag = false)
public class TinkerTestApplicarion extends DefaultApplicationLike {

    public TinkerTestApplicarion(Application application, int tinkerFlags, boolean tinkerLoadVerifyFlag, long applicationStartElapsedTime, long applicationStartMillisTime, Intent tinkerResultIntent) {
        super(application, tinkerFlags, tinkerLoadVerifyFlag, applicationStartElapsedTime, applicationStartMillisTime, tinkerResultIntent);
    }

    /**
     *  install tinker
     * @param base
     */
    @TargetApi(Build.VERSION_CODES.ICE_CREAM_SANDWICH)
    @Override
    public void onBaseContextAttached(Context base) {
        super.onBaseContextAttached(base);
        TinkerInstaller.install(this);
    }

    @TargetApi(Build.VERSION_CODES.ICE_CREAM_SANDWICH)
    public void registerActivityLifecycleCallbacks(Application.ActivityLifecycleCallbacks callback) {
        getApplication().registerActivityLifecycleCallbacks(callback);
    }
}
```
### 3. 使用tinker生成补丁
到此，配置已经基本完成了。下面开始使用。
 1. 首先编译运行一次工程，将生成的apk保存备份在除了build/output/apk以外的文件夹，tinker会读取这个旧的apk与新的apk进行比较生成补丁，同时需要修改**app/build.gradle**文件中oldApk的路径。
 2. 修改工程中代码或者资源，然后打开As gradle任务栏，找到**tinker任务**那一项，选择对应的tinker任务运行

 ![tinker gradle task][4]
然后在build/outputs/tinkerPatch目录下会生成补丁包与相关日志。将补丁包**patch_signed_7zip.apk**push到手机的sdcard目录，此时就可以在工程需要的地方调用tinker 的补丁加载方法了

``` java
TinkerInstaller.onReceiveUpgradePatch(getApplicationContext(), Environment.getExternalStorageDirectory().getAbsolutePath() + "/patch_signed_7zip.apk");
```
因为需要读取sdcard中的文件，因此读写权限必须要配置。
如果补丁加载成功，可以在logcat中看到以下信息

![补丁成功][5]
需要注意的是，tinker默认补丁成功后会杀死应用，因此如果有需要则自定义ResultService继承自**DefaultTinkerResultService**，修改补丁成功后的行为
 3. 重启应用则可以看到打补丁后的效果。

更详尽的tinker知识请参考tinker [Github主页][6]，包括tinker源码与使用示例都可以看到

## Tinker接入其他问题
### 1. 开启multidex支持
如果项目需要用到multidex则需要在gradle中添加multidex依赖，

``` gradle
 compile "com.android.support:multidex:1.0.1"
```
在android-defaultConfig节点中添加

``` nginx
 multiDexEnabled true
```
在Application初始化tinker之前加入

``` cmake
 MultiDex.install(base);
```
### 2. 多渠道打包
tinker默认是每个渠道生成一个对应的补丁包，这样子会造成空间浪费和发布的时候容易出错。因此官方推荐使用[packer-ng-plugin][7]工具进行多渠道打包。

### 3. 资源混淆
如果应用使用了[AndResGuard][8]混淆资源文件，编译流程需要做特殊处理，具体请参考[这篇文章][9]

### 4. 应用加固
tinker1.7.6之后不再支持加固

### 5.tinker与instant run的兼容问题
事实上，若编译时都使用assemble*, tinker与instant run是可以兼容的。但是不少用户基础包与补丁包混用两种模式导致补丁过大，所以tinker编译时禁用instant run，我们可以在设置中禁用instant run或使用assemble方式编译。

大家日常debug时若想开启instant run功能，可以将tinker暂时关闭：

``` objectivec
ext {
    //for some reason, you may want to ignore tinkerBuild, such as instant run debug build?
    tinkerEnabled = false
}
```
更多常见问题请参见[官方wiki][10]


  [1]: ./images/1487586483057.jpg "对比"
  [2]: https://github.com/fourbrother/Robust
  [3]: https://github.com/Tencent/tinker/wiki/Tinker-%E6%8E%A5%E5%85%A5%E6%8C%87%E5%8D%97
  [4]: ./images/1487576984119.jpg "tinker gradle task"
  [5]: ./images/1487578023565.jpg "补丁成功"
  [6]: https://github.com/Tencent/tinker
  [7]: https://github.com/mcxiaoke/packer-ng-plugin
  [8]: https://github.com/shwenzhang/AndResGuard
  [9]: http://www.cnblogs.com/yyangblog/p/6268818.html
  [10]: https://github.com/Tencent/tinker/wiki/Tinker-%E5%B8%B8%E8%A7%81%E9%97%AE%E9%A2%98