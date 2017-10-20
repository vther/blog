---
title: 如何使用Hexo在Github上搭建个人博客
date: 2017年10月15日18:11:59
thumbnail: https://hexo.io/logo.svg
tags: 
- Hexo
categories: 
- Hexo
---

本文主要介绍了在Windows下利用Git Pages和Hexo免费搭建个人博客的方法。

写博客的原因主要有感于[为什么要写博客](https://zhuanlan.zhihu.com/cnfeat/19743861);选择Hexo主要参考了[关于博客技术的选型](https://www.zhihu.com/question/21981094)

千里之行，始于足下。与其感慨路难行，不如马上出发！
<!--more-->

----------
# **本地安装Hexo**

### 1，安装GIT && 安装Nodejs
[安装步骤参考](https://hexo.io/zh-cn/docs/index.html)
[解决国内NPM安装依赖速度慢问题](http://blog.csdn.net/rongbo_j/article/details/52106580)

```bash
npm install -g hexo-cli
```
### 2，初始化博客
在E盘新建目录Blog，并执行命令：
```bash
hexo init
```
可能比较耗时，执行完毕后提示：
> INFO  Start blogging with Hexo!

### 3，生成博客
```bash
hexo generate (OR hexo g)
```
可能比较耗时，执行完毕后提示： 
> INFO  Start processing 
> INFO  Files loaded in 197 ms
> INFO  Generated: index.html INFO  
> INFO  Generated: xxxxxxxxxxxxxx ...略去 
> INFO  28 files generated in 545 ms

### 4，本地启动博客
```bash
hexo server  (OR hexo s)
```
执行完毕后提示： 
> INFO  Start processing
> INFO  Hexo is running at http://localhost:4000/. Press Ctrl+C to stop.

更多命令，例如如何启动指定端口，如果发布博客，可以参考[Hexo官方文档](https://hexo.io/zh-cn/docs/index.html)

### 5，定制化Hexo

> 可以参考[Hexo官方文档](https://hexo.io/zh-cn/docs/themes.html)
> 更换主题：[hexo-theme-icarus](https://github.com/ppoffice/hexo-theme-icarus) , [hexo-theme-spfk](https://github.com/luuman/hexo-theme-spfk)

----------
# **发布博客到Github等网站**

由于没有代理，访问Github比较卡，所以使用了coding.net作为博客的托管
### 1，在coding.net注册并创建项目

在这里我创建了两个项目

项目名 | 用途                    | 是否开启Pages服务   
----   | ------                  |----             
vther  | 博客发布项目            | 是
blog   | 用来存放Hexo的配置和源码| 否

创建完成后，可以本地clone下自己的项目，并且随便修改下再提交，让git保存好账号密码。或者配置SSH公钥。

### 2，修改Hexo的deploy配置
如果使用Github，可以使用注释的配置
```yaml
deploy:
  type: git
  repo:
      coding: https://git.coding.net/vther/vther.git,master
      #github: https://github.com/vther/vther.github.io.git,master
```
修改完成后，使用以下命令来将
```bash
hexo deploy (OR hexo d)
```
最后可以使用 **https://vther.coding.me** 来访问自己的博客了！
**注意：**使用自己的用户名[vther]作为博客的发布项目可以使访问域名更简洁。*如果不这么做，需要用这个URL才能访问：https://{用户名}.coding.me/{项目名}*

----------



# **其他**

### **常见的HEXO配置错误**

-   搭建 hexo，在执行 hexo deploy 后,出现 [**ERROR** Deployer not found] 的错误。 打开package.json发现，没有安装hexo-deployer-git模块，使用以下[命令](https://hexo.io/zh-cn/docs/deployment.html)安装：

```bash
npm install hexo-deployer-git --save 
```

### **关于Markdown **
- [语法介绍](http://www.appinn.com/markdown/)
- [跨平台免费编辑器typora-功能很强](https://www.typora.io/)
- [在线编辑-作业部落](https://www.zybuluo.com/mdeditor)
- [在线编辑-小书匠 - 有很多快捷按钮很方便入门](http://markdown.xiaoshujiang.com/)
- [在线编辑-Cmd Markdown - 有很多主题](http://marxi.co/)


### **参考**
- [Github Pages + Hexo搭建博客](http://fanzhenyu.me/categories/Hexo/)
- [如何搭建一个独立博客——简明 GitHub Pages与 jekyll 教程](http://www.cnfeat.com/blog/2014/05/10/how-to-build-a-blog/)