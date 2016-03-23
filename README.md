## Jenkins+Git+Gradle+Fir 自动Build并上传Fir

### 项目结构
基于Android studio的Gradle项目,代码的版本管理使用Git。项目中有Beta(测试环境)和Online(正式版本)两个productFlavors。内部测试都是发布到Fir
平台上。
### 实现目的
* Beta,Online包发布到不同的Fir链接
* 代码build、Fir发布由Jenkins完成
* Git push后触发Jenkins自动构建
* Fir发布时更新日志从本地的日志更新记录中读取

### 实现步骤
* #### Fir准备
我们要发布Beta和Online两个包到Fir，但是我们这里的packName一样(Gradle可以不同的Flavors设置不同的packName，但是实际项目中不同的
packName会影响我们的第三方SDK的申请)，直接申请两个Fir帐号。
* #### Gradle配置
    * 添加Fir插件 详情参考:[使用 Gradle Plugin 发布应用到 fir.im](http://blog.fir.im/gradle/)
    * 新建一个util.gradle，在里面添加方法`getFirTokenAndFirLog()`,从更新日志记录的Verion文件中获取最新的更新日志(每次升级都
    把最新的的更新日志写在最上面并用========分割)
    * app的build.gradle 添加util.gradle的引用
        > apply from: rootProject.getRootDir().getAbsolutePath() + "/utils.gradle"
    * app 的bulid.gradle 中fir块中，注释掉apiToken和changeLog。apiToken是需要根据Beta和Online环境动态传入，changeLog是用上面
    定义的方法读取。需要注意Gradle Property的优先级: **<font color="red">ext块> -P参数传入 > 在gradle.properties 文件</font>**
    * 给app project 添加一个hook，让在执行Fir的上传任务前赋值apiToken和changeLog。这里用到project.afterEvaluate，在gradle解析完整个任务之后
    找到指定的任务，给任务添加一个前置任务。
        ```
          project.afterEvaluate{
               tasks.getByName("publishApkBetaRelease") {
                 it.doFirst {
                    getFirTokenAndFirLog()
                 }
               }

               tasks.getByName("publishApOnlineRelease") {
                 it.doFirst {
                    getFirTokenAndFirLog()
                 }
               }
          }
        ```
    这里的publishApkBetaRelease 和 publishApOnlineRelease task 就是Fir的上传BetaRelease，OnlineRelease的俩个task，task name是根据gradle的配置变化而
    变化。
    
* #### Jenkins安装
省略
* #### Git 设置
    * 设置Dev、Beta、Online三个branch（发布Beta版本的时候切换到Beat并merge Dev然后Push 到远程的Beta分支，
    发布Online版本的时候切换到Online并merge Dev然后Push 到远程的Online分支，）
    * 在Git服务器上中的pre-receive hooks 中设置 http://jenkinsurl/git/notifyCommit?url=<URL of the Git repository>
    让提交代码后就通知Jenkins有代码更新 [参考](https://zoakerc.com/archives/jenkins-series-trigger-build-through-git-hooks/)
    
* #### Jenkins配置
    * 在Jenkins 的系统管理--插件管理中安装Gradle插件
    * 在Jenkins 的系统管理--系统设置 安装JDK 和 Gradle(最好和本地的版本一致)
    * 新建两个Project（BetaProject，OnlineProject）
        * 源码管理
        ![源码管理](1.png)
        设置好Git仓库和认证，以及对应的branch(BetaProject 对应Beta，OnlineProject 对应 Online)。
        * 构建触发器
        ![构建触发器](2.png)
        选择Poll SCM
        * 构建配置
        ![构建配置](3.png)
            * Invoke Gradle Verison 选择安装的Gradle的版本
            * Build File 选择app下的build.gradle
            * Tasks 填写 clean publishApkBetaRelease -PFirToken="Beta/Online fir帐号对应token"
            其中Beta 对应 `publishApkBetaRelease`，Online 对应`publishApkOnlineRelease`

##### 以后再push Beta/Online branch的时候会分别触发BetaProject/OnlineProject 自动构建，最后会上传到Fir上



参考资料:

+ [Jenkins 持续集成之 Git Hooks 触发构建任务](https://zoakerc.com/archives/jenkins-series-trigger-build-through-git-hooks/)
+ [深入理解Android（一）：Gradle详解](http://www.infoq.com/cn/articles/android-in-depth-gradle)

