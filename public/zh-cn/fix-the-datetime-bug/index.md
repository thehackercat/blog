# 使用Octopress搭建静态博客


![Spiderman](http://7xse6j.com1.z0.glb.clouddn.com/spiderman.jpg)

最近把个人博客搭好了，用了[Octopress](https://github.com/octopress/octopress),一个基于 Jekyll 的集成开发工具。

原来 CSDN 的那个『骇客猫』弃坑了。
## 安装和配置.
Octopress 的安装配置比较简单，是需要按照官网或者网上一些[教程](http://shengmingzhiqing.com/blog/setup-octopress-with-github-pages.html/)一步步走即可。
<!--more-->

由于我在2015年10月1日更新了 OS X EI Capitan，新系统在权限设置上增加了 [System Integrity Protection](http://www.macworld.com/article/2948140/os-x/private-i-el-capitans-system-integrity-protection-will-shift-utilities-functions.html) (SIP) 来提高系统安全性并且在 System Library 的路径上作了修改，导致了一些安装 Jekyll 时出现的异常，罗列如下：

1. 如果你使用命令行 `$ gem install jekyll` 安装 Jekyll 时 遇到了如下问题：

   ``` bash
   ERROR:  While executing gem ... (Errno::EPERM)
   Operation not permitted - /usr/bin/jekyll
   ```

   辣么尝试使用 `$ sudo gem install -n /usr/local/bin/ jekyll` 从而有效地避开 EI Captian 中 rootless 用户的权限问题。
   或者有更彻底的办法，在终端输入

   ```shell
   $ export PATH=/usr/local/bin:$PATH
   ```

   这样会将原来 /usr/bin 的路径更改为 /usr/local/bin ，然后再进行安装，一劳永逸，但我不建议这么做。

2. 如果你在进行上述操作时遇见了如下问题：

   ```shell
   在命令行中输入 ```$ xcode-select —install```就可以安装了。
   ```

   辣么你应该没有安装 OS X developer tools ，安装后才能编译一些 ruby 的原生的拓展插件。
   在命令行中输入 ```$ xcode-select —install```就可以安装了。
3. 如果你遇到了任何 Permission denied 的问题：

   ```shell
   ERROR:  While executing gem ... (Errno::EACCES)
   Permission denied
   ```

   辣么在命令行之前加上 `$ sudo` 。

## 个性化修改
对于我的博客的个性化修改我主要做了以下三个：

- 第三方主题：Octopress有很多[第三方主题](https://github.com/imathis/octopress/wiki/3rd-Party-Octopress-Themes)安装也很便捷。
- 插件安装：可以在 `/plugins` 目录下安装一些第三方插件，诸如 Disqus 评论系统、 Twitter 的时间线等。
- 样式修改：我在 `/sass/custom/_styles.scss` 中修改了字体、 blog 的行间距以及一些边边角角的地方。

我使用了**第三方主题** cleanpress 她极简的风格很吸引我，但是这个主题有蛮多 bug 的。

比如，在首页会遇见一个博客的 post 时间无法显示导致日历图标和目录图标重合的问题，如下所示：

![Date_format_bug](https://cloud.githubusercontent.com/assets/190438/8529829/37c7bc80-23d2-11e5-896b-61a6cd0fc590.png)

经过一上午的 debug ，我发现了在 `/source/_includes/post/date.html` 第11行
` date_formatted ` 是没有声明 formatted 的格式的从而导致了无法显示。
故我将其替换成了 ` date | date: "%b %e, %Y" ` 然后就可以显示出 format 后的时间了。

![Date_format_bug_fixed](https://cloud.githubusercontent.com/assets/10649416/11036546/3078799a-8734-11e5-80c6-8460962bd945.png)

还有在发布超过20字的标题的博客时，首页的相应博客处会出现样式错误， date_line 会与标题重叠在一起。
这两个 bug 我都已修复并提交了，在[这里](https://github.com/macjasp/cleanpress/pull/23)可以查看并修改。

我使用了**第三方插件** [Disqus](https://disqus.com/) 非常棒的评论系统，以及 [Twitter Timeline](https://dev.twitter.com/web/embedded-timelines) 也是非常棒的时光机插件。根据官网的教程很容易安装并使用。

接着我还修改了**样式**，其中把全局字体改成了谷歌和 Adobe 联合发布的 **思源黑体** ，漂亮得不像实力派。修改过程主要参考了[这篇文章](http://www.uisdc.com/source-han-sans-webfont)。其中每个不同的 Adobe 账户需要插入的是不同的 Typekit 代码( Adobe 会帮你自动生成代码)。但需要注意的是 Adobe Typekit 虽然不是免费服务，但也有免费方案可以选择，注册后有每月 25,000 次的浏览次数额度，对于一般个人 blog 或小型网站来说其实还算充裕（当然你也可以考虑付费升级，价格并不高 ：P ）。

## 为什么是蜘蛛侠 ？
嗷，其实是这样的，熟悉我的人就知道，我个人是漫威巨粉，而先前我看到 cleanpress 的 demo 页是酱的图片我很喜欢，并且想起来[蜘蛛侠要回归漫威了](http://marvel.com/news/movies/24062/sony_pictures_entertainment_brings_marvel_studios_into_the_amazing_world_of_spider-man), ~~再看绿箭我就节食5分钟！~~ 这就像苯宝宝又要回归已经弃坑的绿箭侠一样鸡冻。

![苯宝宝只想安静地装醇](http://1.im.guokr.com/DQq0wz6RNDAcEszKLQI4xJXatRcJXp-327H-__RyrwToAQAAlQEAAEpQ.jpg)

