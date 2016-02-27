---
layout: post
title: "BUG:No Such File or Directory - Jekyll"
date: 2015-10-26 21:34:17 +0800
comments: true
categories: life
tags: [tips]
styles: [data-table]
---

> “版本管理是一个艰巨的任务” —— 无名


#### **当Octopress遇上OSX EI Captain**
<br>

前一阶段升级了最新的OSX系统，谁知写完日志好端端的`rake generate`失效了，下意识的`rake preview`，报错：

```
Errno::ENOENT: No Such File or Directory - Jekyll 
```

这下慌乱了，本来对`ruby`就不熟，这怎么调错，还好前人早已总结：

[Errno::ENOENT: No Such File or Directory - Jekyll ~ Octopress and El Capitan](http://schalkneethling.github.io/blog/2015/10/16/errno-enoent-no-such-file-or-directory-jekyll-octopress-el-capitan/)

我小结一下：是新系统下，需要新的依赖库，而这些库需要ruby2.2.3的版本安装，而系统默认是2.2.0。

我在安装完ruby2.2.3后，`ruby -v`后仍是2.2.0，也没有折腾，直接改了`~/.bash_profile`里面的`PATH`路径，加上了新安装好的ruby所在的`bin`目录，之后升级完+安装完相应的包后就可以`rake generate && deploy`了。

在CSDN上写文章很方便，移到这里好多细节需要自己处理，图片啊，表格啊，markdown语言啊，着实不容易，但是能增加一点点成就感呢。😄


