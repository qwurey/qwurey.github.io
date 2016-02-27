---
layout: post
title: "osx plist 定时任务"
date: 2015-09-07 09:56:18 +0800
comments: true
categories: osx
tags: [tips]
---

osx下定时任务是通过写plist脚本实现的，plist要比linux/osx下使用crontab更为简单、方便、细粒度。

可以通过两种方式来设置脚本的执行时间。一个是使用StartInterval，它指定脚本每间隔多长时间（单位：秒）执行一次；另外一个使用StartCalendarInterval，它可以指定脚本在多少分钟、小时、天、星期几、月时间上执行，类似如crontab的中的设置。

比如：

写一个名为“com.urey.test.plist”的脚本保存在目录“/Users/urey/data/”下面。

```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.urey.test.plist</string>
    <key>ProgramArguments</key>
    <array>
        <string>/Users/urey/data/test.sh</string>
    </array>
    <key>KeepAlive</key>
    <false/>
    <key>RunAtLoad</key>
    <true/>
    <key>StartInterval</key>
    <integer>5</integer>
</dict>
</plist>
```
其中，key是plist对应的属性名，相应的为该属性名对应的属性值（即key/value，plist为一种特殊的xml文件）。

比如，上面我们定于了该定时任务叫“com.urey.test.plist”，定时执行的脚本是“/Users/urey/data/test.sh”，载入时开始执行，执行间隔是每5s一次。

/Users/urey/data/test.sh是shell脚本，我写的测试脚本如下：

```
#!/bin/sh
cd /Users/urey/data/
LOG=`date +"%Y-%m-%d %H:%M:%S"`
LOGFILE=`date + "log-%Y%m%d-%H:%M:%S.log"`
echo $LOG > $LOGFILE
```

好，下面我们就可以通过命令：

```
sudo launchctl load com.urey.test.plist
sudo launchctl unload com.urey.test.plist
```
来载入或者取消载入对应的定时任务。

值得注意的是，我们还必须明确com.urey.test.plist的权限和所有者：

<img src="http://7xlhna.com1.z0.glb.clouddn.com/plist.png" width = "600" height = "200" alt="图片名称" align=center />

即需要操作：

```
sudo chmod 600 com.urey.test.plist
sudo chown root com.urey.test.plist
```

至此，一个简单的定时任务就做好了。

我们可以把plist脚本存放在路径为/Library/LaunchDaemons或/Library/LaunchAgents，其区别是后一个路径的脚本当用户登陆系统后才会被执行，前一个只要系统启动了，哪怕用户不登陆系统也会被执行。

当然，我们也可以存放在任意的其它路径上，这时就不会有上面的限制且会根据具体的属性值进行限制。