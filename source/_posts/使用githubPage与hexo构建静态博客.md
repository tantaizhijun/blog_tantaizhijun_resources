---
title: 使用github pages + hexo构建自己的博客网站
categories: 其他
date: 2018-11-06 15:48:43
tags: 博客
---

**前言**
    本文章只对github pages + hexo构建博客网站做简要指导, 请做好以下软件安装准备工作...

### 一.准备工作

#### 1.安装git
[下载地址](https://git-scm.com/downloads)
#### 2.安装node
[下载地址](http://nodejs.cn/download/)
#### 3.注册一个github帐号
[注册地址](https://github.com), 至于github怎么连通使用, 请自行百度, 本文章不做指导.

<!--more-->
### 二.使用hexo构建博客网站

#### 1.安装hexo
```
$ npm install -g hexo-cli
```
#### 2.初始化hexo
打开git bash, 执行如下代码:
```
$ hexo init [folder]
```
**说明:** 最好新建一个空文件夹, 切换到该目录执行hexo init 命令, 也可以使用 [folder] 参数指定一个路径作为你的博客目录.

#### 3.本地服务启动
hexo init完成之后, 一个简单的博客网站就完成了, ,并且自带了主题, 可以在本地启动hexo服务进行预览博客了.
执行如下代码:
```
$ hexo server
```
**说明:** 如果报如下错误, 应是hexo server模块没有安装, 
```
$ hexo server
Usage: hexo <command>

```
执行如下代码安装hexo server:
```
$ npm install hexo-server --save
或者:npm i hexo-server
```
重新执行hexo server即可, 启动之后, 默认运行在4000端口, 打开浏览器,输入: http://localhost:4000 即可预览。。。
如果端口被占用,执行: ** hexo -p 5000 server** ,换成5000端口重新运行即可.

### 三.发布到github
#### 1.新建仓库
在你的github新建一个仓库, 仓库名格式必须是: **username.github.io**, username就是你的github账号名称.
#### 2.建立关联
创建完成之后, 复制你的仓库地址, 
然后打开你博客目录下的_config.yml文件，拉倒最下面，修改deploy为以下代码:
```
deploy: 
  type: git
  repository: git@github.com:peterzhen40/peterzhen40.github.io.git
  branch: master
```
**注:** repository后面就是你刚才复制的仓库地址, 每个分号(":")后面有一个英文空格.
#### 3.部署
hexo部署到git我们需要安装hexo-deployer-git插件, 执行如下命令:
```
npm install hexo-deployer-git --save
```
执行如下命令进行部署:
```
$ hexo d        ##部署,该命令等同于: hexo deploy
```

部署完成之后, 就可以在 http://username.github.io 地址进行访问了...
以后的部署就是这三步了:
```
$ hexo clean    ##清除已经生成的静态页面, 
$ hexo g        ##重新生成静态页面, 该命令等同于: hexo generate
$ hexo d        ##部署,该命令等同于: hexo deploy
```

** <font color="red">说明:</font>**
发布到github只需使用上面的hexo d 命令即可, 不要把本地源码发布到github, (如:IDEA本地local change的变动不用管,hexo会帮我们把生成的博客自动提交的)


### 四.其他相关

#### 1.更换主题
hexo初始化默认自带一个主题, 名为: landscape, 如果不喜欢, 可以更换.
1. 可以去 [主题](https://hexo.io/themes/) 找一个自己喜欢的主题, 找到他的仓库地址, 复制仓库地址.
2. 切换到你的博客目录下的/themes/目录下, 执行如下代码:
```
$ git clone "你复制的仓库地址"        ##即把主题下载下来
```
3. 找到博客目录下的 _config.yml 文件, 修改theme, theme后面就是你下载的主题名称, 
(注意分号后面有个英文空格, 比如我的是: **theme: hexo-theme-next**)


#### 2. hexo常用命令
##### ①.新建一个博客文件
```
hexo new [layout] "name"
```
会在source/_posts/下生成一个名叫"name"的md文件, 就是你新建的博客.
说明: [layout]用来指定文章的布局，默认是post, hexo默认有三种布局，每种布局的路径不同：

|布局|路径|
|---|---|
|post|source/_posts|
|page|source|
|draft|source/_drafts|

##### ②.新建一个分类模块
```
$ hexo new page categories
```
-你会发现你的source文件夹下有了categorcies/index.md，打开index.md文件将title设置为title: 分类, type设置为"categories"
-打开**主题配置文件**_config.yml 找到menu，将categorcies取消注释
-要把文章归入分类只需在写文章时,在的顶部标题下方添加categories字段，即可自动创建分类名并加入对应的分类中,如: 
```
title: 分类测试文章标题
categories: 分类名
```
##### ③.添加标签模块
```
$ hexo new page tags
```
-同上, 你会发现source文件夹下有了tags/index.md，打开index.md文件将title设置为title: 标签, type设置为"tags"
-打开**主题配置文件**_config.yml, 找到menu，将tags取消注释
-给文章添加标签, 只需在文章的顶部标题下方添加tags字段，即可自动创建标签名并归入对应的标签中

##### ④.添加关于模块
```
$ hexo new page about
```
-同上, 你会发现source文件夹下有了about/index.md，打开index.md文件即可编辑关于你的信息。
-打开 主题配置文件 找到menu，将about取消注释

##### ⑤.添加搜索功能
安装 hexo-generator-searchdb 插件
```
$ npm install hexo-generator-searchdb --save
```
-打开 站点配置文件 找到Extensions在下面添加下面信息: 
```
# 搜索
search:
  path: search.xml
  field: post
  format: html
  limit: 10000
```
##### ⑥添加阅读全文按钮
这个很简单, 只需在文章结尾添加: 
```
<!--more-->
```
这样, 更多的文章内容就不会显示了，只能点击阅读全文才能看...
##### ⑦修改文章内超链接样式
打开文件 themes/next/source/css/_common/components/post/post.styl，在末尾添加
```
.post-body p a {
  color: #0593d3;
  border-bottom: none;
  border-bottom: 1px solid #0593d3;
  &:hover {
    color: #fc6423;
    border-bottom: none;
    border-bottom: 1px solid #fc6423;
  }
}
```

##### ⑧设置网站缩略图标
把你的图标文件（png或jpg格式，不是ico文件）放在主题目录themes/next/source/images里，然后打开**主题配置文件**_config.yml, 找到favicon，将small、medium、apple_touch_icon三个字段的值都设置成/images/图片名.jpg就可以了，其他字段都注释掉。

##### ⑨去掉文章目录标题的自动编号
我们自己写文章的时候一般都会自己带上标题编号，但是默认的主题会给我们带上编号，很是别扭，如何去掉呢？
打开主题配置文件，找到number, 设置为false即可, 如下:
```
# Table Of Contents in the Sidebar
toc:
  enable: true

  # Automatically add list number to toc.
  number: false

  # If true, all words will placed on next lines if header width longer then sidebar width.
  wrap: false
```

更多hexo知识请查看 [hexo官方文档](https://hexo.io/zh-cn/docs/)
更多主题配置, 请查看[主题设置](http://theme-next.iissnan.com/theme-settings.html)
更多hexo插件, 请查看[hexo插件](https://hexo.io/plugins/)

#### 3.markdown语法
自己百度去学习吧...




