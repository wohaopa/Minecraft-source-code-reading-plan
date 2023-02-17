## 结构：
```
net.minecraft.launchwrapper
├─injector
  ├─AlphaVanillaTweakInjector
  ├─IndevVanillaTweakInjector
  └─VanillaTweakInjector
├─ITweaker
├─IClassNameTransformer
├─IClassTransformer
├─AlphaVanillaTweaker
├─IndevVanillaTweaker
├─VanillaTweaker
├─Launch
├─LaunchClassLoader
└─LogWrapper
```
>version: 1.2

本包结构简单，功能单一。同时也是启动Minecraft、修改Minecraft字节码的核心。  是CoreMod的核心框架，也是实现动态反混淆的核心框架。
核心类：`Launch LaunchClassLoader`
## 类 Launch
本类含有`main`方法，是安装Forge后的入口。用于修改启动Minecraft，Tweaker将在这里执行。
### 字段
1. `DEFAULT_TWEAK` 默认tweak的类名，当游戏参数`--tweakClass`缺失时作为默认参数使用。
2. `minecraftHome` 游戏的运行路径，运行时产生的各种文件均存放于此。由`--gameDir`赋值。
3. `assetsDir` 游戏的assets路径，由`--assetsDir`赋值。
4. `blackboard` 用于存放一堆全局变量。
5. `classLoader` 详见 `LaunchClassLoader`。
### 方法
1. `main` 略
2. `<init>` 初始化、替换类加载器。
3. `launch` 解析游游戏参数，具有一个ITweaker队列，调用`ITweaker.injectIntoClassLoader`方法，允许动态添加ITweaker到队列。并保证仅调用一次。然后调用所有`ITweaker.getLaunchArguments`方法并将结果放入argumentList，最后调用第一个找到的ITweak `primaryTweaker`的`getLaunchTarget`方法获得并运行启动类`main`方法，传入argumentList。接下来游戏正式启动。
### 总结
本类主要替换ClassLoader和执行ITweaker。非常面向过程的写法。
## 类 LaunchClassLoader
继承URLClassLoader，是Forge修改Minecraft字节码的核心类。在加载Minecraft类时对其修改。也是实现CoreMod的核心类。
### 字段
1. `BUFFER_SIZE` 缓冲大小，初始缓冲区大小，每次扩容也会扩大这些（笔者认为4096字节太小了，mc中大部分类超出这个大小，每次扩容会浪费大量时间。建议改为8192。
2. `sources` 用于存储原始URL的列表。
3. `parent` jvm原有的ClassLoader。用于加载不需要特殊处理的类。
4. `transformers` 所有的transformer均会注册在此，在`findClass`方法中被遍历。
5. `cachedClasses` 缓存所有被加载的类。
6. `invalidClasses` 各种原因加载失败的类
7. `classLoaderExceptions` 不被LaunchClassLoader处理的类，会用`parent`正常处理，不进入`cachedClasses`。
8. `transformerExceptions` 不接受transformer的类，仍然接受LaunchClassLoader处理，会进入`cachedClasses`。CoreMod的Tweaker会在此处。
9. `packageManifests` Jar包内的manifest文件集合
10. `resourceCache` 
11. `negativeResourceCache` 加载失败的
12. `renameTransformer` 映射转换器，用于运行时反混淆。只允许存在一个。
13. `EMPTY` 指代缺失的manifest
14. `loadBuffer` 线程私有的缓冲区，用于存放加载缓冲数组。
15. `RESERVED_NAMES`
16. `DEBUG` 调试，`-Dlegacy.debugClassLoading=true`打开调试，可以输出更多信息。
17. `DEBUG_FINER` 在执行`runFransformers`方法时输出更多信息，`-Dlegacy.debugClassLoadingFiner=true`打开
18. `DEBUG_SAVE` 保存所有被LaunchClassLoader处理过的类至`tempFolder`路径下。
19. `tempFolder` 在开启`DEBUG_SAVE`时用于保存被处理过类的文件夹。
### 方法
1. `<init>` 初始化，排除一些包、保护一些包（不接受transform的包） 。
2. `registerTransformer` 注册transformer至`transformers`，特别的，会找出唯一的`renameTransformer`。
3. `findClass` 复写父类方法，返回类，由jvm自动调用。用于查找加载jvm所需要的class，类核心方法，如果需要transform则调用`runTransformers`方法。
4. `saveTransformedClass` 在开启`DEBUG_SAVE`时用于保存transform后的类文件。由`findClass`调用。
5. `untransformName` 返回transform前类名，也就是未被反混淆的notch名。由`findClass`调用。
6. `transformName` 返回transform后的类名，也就是srg名。由`findClass`调用。
7. `isSealed` 
8. `findCodeSourceConnectionFor` 
9. `runTransformers` 调用transformers中所有`transform`方法。transform执行的地方，由`findClass`调用。
10. `addURL` 重写父类方法，为`sources`添加URL
11. `getSources` get sources
12. `readFully` 用于读入InputStream流中的字节的工具方法，缓冲数组将在这里扩容，由`getClassBytes`调用。
13. `getOrCreateBuffer` 管理`loadBuffer`的工具方法。由`readFully`调用。
14. `getTransformers` get transformers（返回不可变列表
15. `addClassLoaderExclusion` 注册不受处理的class
16. `addTransformerExclusion` 注册不受transform的class
17. `getClassBytes` 读取class文件，返回字节数组。
18. `clearNegativeEntries` 批量删`negativeResourceCache`
19. 静态`closeSilently` 用于关闭流的工具方法。
### 总结
forge自定义了一个ClassLoader来在类加载完成前对字节码执行修改，并为这一套体系保留大量接口供自定义。
## 类LogWrapper
用于输出调试信息的工具类。使用的是log4j的logger。仅在进入`LaunchTarget`之前被使用。具有Log工具类常见方法（性能不咋地就是了，不过使用的时间很短。
## 接口 ITweaker
所有tweaker均需实现此接口。
### 方法
1. `acceptOptions` 传入启动参数
2. `injectIntoClassLoader` 执行操作，注册transformer，设置不接受处理类啥的
3. `getLaunchTarget` 传出启动目标，仅`primaryTweaker`被调用此方法。
4. `getLaunchArguments` 传出启动参数，为Minecraft main方法的args添加参数。
## 接口 IClassTransformer
所有transformer均需实现此接口。
### 方法
1. `transform` 传入字节码、类名等信息，传出修改后的字节码。coremod修改字节码的操作需在此进行。
## 接口 IClassNameTransformer
类名重命名接口，目前只有`DeobfuscationTransformer`实现
### 方法
1. `unmapClassName` 返回未映射的类名
2. `remapClassName` 返回映射后的类名
## 类 VanillaTweaker
实现`ITweaker`接口，无特殊行为。仅注册`injector.VanillaTweakInjector`。
## 类 injector.VanillaTweakInjector
实现`IClassTransformer`接口，无特殊行为。
### 方法
1. `transform` 仅注入一个回调至`net.minecraft.client.Minecraft.main`方法
2. `inject` 被注入的回调。取消图片缓存，修改游戏目录。
3. `loadIconsOnFrames` 设置icon至窗口。由于`inject`调用。
4. `loadIcon` 加载icon的工具方法。
## 类 IndevVanillaTweaker AlphaVanillaTweaker
行为与`VanillaTweaker`一致。
## 类 injector.IndevVanillaTweakInjector
行为与`injector.VanillaTweakInjector`一直，`transform`是将回调注入到了`run`方法。
## 类 injector.AlphaVanillaTweakInjector
疑是早期版本的启动类，未深究。
***
至此，`net.minecraft.launchwrapper`包已经全部介绍完毕。本包入口`Launch.main`也是Minecraft Forge客户端的入口。在本包中，minecraft forge会提供修改minecraft字节码的方法，也是CoreMod实现的框架。  
最后，本人能力有限，也非计算机专业，对代码理解仍有缺陷。