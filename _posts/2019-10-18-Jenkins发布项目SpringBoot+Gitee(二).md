---
layout: post_layout
title: 2019-10-18-Jenkins发布项目SpringBoot+Gitee
time: 2019年10月18日 星期五
location: 海南 海口
pulished: true
---

**在码云上新建一个SpringBoot项目，使用Jenkins从码云拉取代码至部署环境，使用maven进行打包和启动。由于这里只有一台虚拟机服务器，因此这里就在Jenkins本地打包，正常的话是使用Publish over SSH插件将包传送至远程发布服务器进行启动**



## (一) 安装Gitee插件
**系统管理 -> 插件管理，搜索Gitee插件，并且安装**



## (二) 新建项目
> * **系统管理 -> 新建项目。填写基本项目信息、项目git地址、码云账号密码**
> * **编写shell脚本关闭项目、备份包、启动项目**
> * **shell脚本踩的坑**


## (三) 安装Git Parameter插件
**平时开发会存在多个分支和tag,如生产分支、测试分支、tag1.0。Git Parameter插件能够在构建的时候，允许我们选择git某个分支或tag进行构建发布**



## (四) 自动构建

