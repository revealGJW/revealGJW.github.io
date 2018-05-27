---
title: 使用Hexo+Github搭建自己的博客
date: 2017-04-29 12:12:44
tags:
  - Hexo
  - NexT
categories:
  - 技术杂文
---
最近搭了自己的博客，把过程记录一下，避免以后再次踩坑。
<!--more-->
## Hexo
跟着[Hexo官网](https://hexo.io/zh-cn/docs/index.html)步骤来，十分钟就装好了吧，一个提醒：装好nvm后，**重启终端**才能安装Node.js。
配置文件要注意：
1. 冒号后面必须有一个半角空格；
2. 时区的值一定要填对，北京时间是 Asia/Shanghai，填错的话会无法加载tags等页面；
3. 如果要使用disqus评论，url需要填写正确。
4. 可以使用[cmd markdown编辑器](https://www.zybuluo.com/mdeditor)即时编辑自己的文章，写完文章之后上传至Hexo对应文件夹中即可。

## NexT主题
看到NexT主题比较方便看起来也比较清爽，就用了这个主题。
没什么好说的，按照[NexT官方文档](http://theme-next.iissnan.com/)来一步步配置即可。
注意的地方：
1. 生成tags/categories等界面时，要保证上文说的 `timezone`属性设置正确，否则会无法正确识别tags。
2. 集成第三方服务按照说明来即可，本站只添加了disqus评论和不蒜子统计第三方服务。

## 部署至Github
1. 首先要有一个github账号。
2. 之后建立一个名称为 username.github.io 的静态仓库（此处的username替换为自己的名字）。
3. 还要在本地安装hexo-deployer-git，具体指令为： `npm install hexo-deployer-git --save`。
4. 安装完成之后，在本地的`_config.yml`文件中进行配置：
```
    deploy:
    type: git
    repo: https://github.com/revealGJW/revealGJW.github.io.git
    branch: master
```
repo值填入自己的仓库提交地址。
配置好之后，使用`hexo d -g`命令按照提示即上传完成。
这一步完成之后，直接访问 username.github.io 就可以看到自己的博客了。
## 域名绑定(可选）
如果你还有一个域名，可以将域名与git pages绑定。
1. 首先在`source`文件夹下，新建一个`CNAME`文件（无后缀）。
2. `CNAME`文件中写入自己的域名，不要加任何前缀。例如：`revealing.cn`
3. 前往到腾讯云，添加自己的域名（如果本来就是腾讯云的域名可忽略本步和下一步）。
4. 添加完成后配置DNS，修改DNS为`f1g1ns1.dnspod.net`和`f1g1ns2.dnspod.net`。
5. 最后添加两条记录，记录类型为`CNAME`，记录值为`username.github.io`，主机记录分别为`@`和`www`。
生效之后过一段时间，即可通过自己的域名访问到博客。

## 常用指令
```
hexo new "article name" #创建新文章
hexo generate #生成静态文件
hexo server #启动服务器
hexo deploy #部署网站

hexo d -g #生成静态文件并部署
hexo s -g #生成静态文件并启动服务器
```
## 总结
列出了一些搭建博客时关键的步骤，以及自己在搭建时碰到的不少坑。权当为自己下次安装列个提纲。一些细节问题不要忽略掉，否则会有奇怪的bug出现。