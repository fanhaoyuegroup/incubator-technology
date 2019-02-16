# interest-group 简介
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;为了更好的促进团队的学习技术氛围，更好的同步记录更新学习成果，为新同学提供优质的学习资料，我们决定维护一份优质的预售学习资料。推动大家共同进步。

# 工程模块
**interest-group** 兴趣小组学习总学习模块<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;| <br> 
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;|—— **document** : 存储每个兴趣小组总结的 md 文档<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;| <br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;|—— **hystrix**[module]: hystrix 学习小 demo <br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;| <br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;|—— **zk**[module]: zk api 操作的小 demo <br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;| <br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;|—— **sharding-jdbc**[module]: 分表分库操作的小 demo <br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;| <br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;|—— **algorithm** : 源码兴趣小组相关总结学习代码，包括一些较好的算法代码<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;| <br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;|—— **canal**[module]: 增量开发canal的相关学习。可与redis以及mq、zk联合学习。都有相关涉及<br>

# 工程代码文档要求与规范
由于该工程每个人都可以分享自己的学习心得和资料，每个人的学习和代码风格都不一样，所以某些模块我们需要一个规范，也希望大家严格按照要求来创建自己的工程。规范如下

1.新工程创建要求<br>

新工程创建一律从 interest-group 下创建一个子 module

![image.png](https://upload-images.jianshu.io/upload_images/10204326-1ea83f4df08ee47a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

工程创建只能创建 maven 或 springboot 工程。方便全局 pom 加载

![image.png](https://upload-images.jianshu.io/upload_images/10204326-a656e9b371ae21fd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

例如 sharding-jdbc 工程创建：

![image.png](https://upload-images.jianshu.io/upload_images/10204326-f50e295651c2975d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![image.png](https://upload-images.jianshu.io/upload_images/10204326-e63dcc303b87e771.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

通过上图我们会发现，一旦 sharding-jdbc 创建成功，父级 pom 文件会马上关联上 sharding-jdbc 的 module，然后我们只需要刷新一下 父级的 pom 文件就可以把 sharding-jdbc 加载进来，然后 sharding-jdbc 也可以使用父级里面的工具包，例如 lombok，guava 等。如果不创建 maven 文件，我们加载完父类的 pom 文件之后，还需要到指定工程将指定文件夹标记为源目录才可以允许相应的java类，这样很麻烦，所以麻烦大家统一创建 maven 或 springboot 项目。

![image.png](https://upload-images.jianshu.io/upload_images/10204326-c3b03f4dc69c0e47.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我们看到这个项目我们需要手动标记 sources root 目录才可以允许，项目一多非常麻烦。
![image.png](https://upload-images.jianshu.io/upload_images/10204326-fb4778b9ff092dcb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

2. module 要求<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;同类框架代码学习我们统一只创建一个 module，pom 统一继承 interest-group 父pom，artifactId 一定要言简意赅，命名如下：

![image.png](https://upload-images.jianshu.io/upload_images/10204326-78892b8b847393a9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

3. pom 包引入要求<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;如果有公用的 jar 包文件统一抽取到父类 pom 文件中，每个 module 中的 jar 包依赖，希望大家可以将一些冲突的 jar 包排除掉（升级需谨慎，要考虑兼容性），尽可能保证 pom 引入的 jar 包干净

4. 项目包名要求<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;一级包名统一使用 main.java.fan.xxx（xxx module 名称）如图：

![image.png](https://upload-images.jianshu.io/upload_images/10204326-453899e9c9217b6f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

5. 代码书写要求<br> 
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;代码分享的主要目的是为了给其他同学提供学习资料，因此我们务必保证代码的质量，至少代码是可以运行的，代码是给机器看的注释是给同学看的，因此我们代码注释必须写详细了。如图：

![image.png](https://upload-images.jianshu.io/upload_images/10204326-7f0d2594987a0bf9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
6. document 文档编写要求<br> 
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;为了保证团队学习成功质量，给后续新同学提供优质的学习资料，我们会对新上传的文档进行质量检查，
对于文档太过于简单且质量不合格的文档会被打回，所以务必请各个兴趣小组的组长保证自己组内文章的质量。

7.文档中图片上传要求

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;目前有很多同学文章里面的图片丢失，因此对图片上传做如下要求：

- 在自己文件夹下面创建一个 img 目录，里面存放你需要上传的图片，图片名称一定要和文章里面的图片名称对应上，使用相对路径。如下图：

![image.png](https://upload-images.jianshu.io/upload_images/10204326-3ba652a4fb9bcc87.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- 大家也可以将自己的图片上传到公司图片服务器获取第三方云存储平台，然后拿到访问路径即可，如下图，上传到简书中。

![image.png](https://upload-images.jianshu.io/upload_images/10204326-dceba9211d3f9d4f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

公司图片服务器上传地址：http://upload.2dfire.net/utils/page/upload.html

**建议第二种方式，采用第一种方式会使我们的项目越来越大，加载很费劲。**

8.附件上传要求

大家如果需要上传附件，那么直接在自己文件夹下创建一个 attachment 文件夹，大家可以把附件放在该文件夹下，命名一定要明确。然后在自己的文章里面链接过来即可。

![image.png](https://upload-images.jianshu.io/upload_images/10204326-e2e0b25ee3508299.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

gitlab 成效图：直接下载即可

![image.png](https://upload-images.jianshu.io/upload_images/10204326-041efdc2d9817bb2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
