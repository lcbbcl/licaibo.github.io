---
layout: post_layout
title: Jenkins发布项目SpringBoot+Gitee(二)
time: 2019年10月18日 星期五
location: 海南 海口
pulished: true
---

**在码云上新建一个SpringBoot项目，使用Jenkins从码云拉取代码至部署环境，使用maven进行打包和启动。由于这里只有一台虚拟机服务器，因此这里就在Jenkins本地打包，正常的话是使用Publish over SSH插件将包传送至远程发布服务器进行启动**

![cmd-markdown-logo](https://licaibo.github.io/assets/img/jenkins-build.jpg)


## (一) 安装Gitee插件
**系统管理 -> 插件管理，搜索Gitee插件，并且安装**

![cmd-markdown-logo](https://licaibo.github.io/assets/img/jenkins-pluns1.jpg)

## (二) 新建项目
> * **系统管理 -> 新建项目 -> 构建一个自由风格的软件项目；填写基本项目信息、项目网址和git地址、码云账号密码**

![cmd-markdown-logo](https://licaibo.github.io/assets/img/jenkins-gitee1.jpg)

![cmd-markdown-logo](https://licaibo.github.io/assets/img/jenkins-gitee2.jpg)

> * **编写shell脚本关闭项目、备份包、启动项目**
**注意：要求服务启动之前加上 BUILD_ID=dontKillMe，名字可以任意取。原理：jenkins默认会在构建完成后杀掉构建过程中的shell命令触发的衍生进程。jenkins根据BUILD_ID识别某个进程是否为构建过程的衍生进程，故修改BUILD_ID后，jenkins就无法识别是否为衍生进程，则此进程能在后台保留运行**

```shell
echo "开始构建"
pwd
mvn clean install -Dmaven.test.skip=true
APPNAME=licaibo-eureka
DATE=$(date +%Y%m%d)
echo $DATE

pid=`ps -ef | grep $APPNAME.jar | grep -v grep | awk '{print $2}'`
echo $pid
if [ -n "$pid" ];
	then
kill -9 $pid
fi
echo "kill done"

mv /home/parallels/push/$APPNAME.jar /home/parallels/backup/$APPNAME$DATE.jar

mv -f /home/parallels/.jenkins/workspace/licaibo-eureka/target/$APPNAME.jar /home/parallels/push/$APPNAME.jar
BUILD_ID=dontKillMe
nohup java -jar /home/parallels/push/$APPNAME.jar > /home/parallels/log/$APPNAME.log 2>1&
if [ $? = 0 ];
	then
sleep 10
cd /home/parallels/log
tail -n 50 $APPNAME.log
fi
echo "完成构建"
```

![cmd-markdown-logo](https://licaibo.github.io/assets/img/jenkins-gitee3.jpg)



## (三) 安装Git Parameter插件
**平时开发会存在多个分支和tag,如生产分支、测试分支、tag1.0。Git Parameter插件能够在构建的时候，允许我们选择git某个分支或tag进行构建发布**

![cmd-markdown-logo](https://licaibo.github.io/assets/img/jenkins-pluns2.jpg)


## (四) 选择分支自动构建

![cmd-markdown-logo](https://licaibo.github.io/assets/img/jenkin-bulid-1.jpg)

![cmd-markdown-logo](https://licaibo.github.io/assets/img/jenkin-bulid-2.jpg)

![cmd-markdown-logo](https://licaibo.github.io/assets/img/jenkin-bulid-3.jpg)




