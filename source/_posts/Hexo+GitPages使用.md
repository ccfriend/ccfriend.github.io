---
title: Hexo+GitPages使用
date: 2022-03-30 16:37:09
categories: blog
tags: [blog, hexo]
---
使用Hexo+Next+github pages搭建最简单的博客。

里面有一些小坑，需要注意，主要项目名字，以及Deploy的配置。

还有个小技巧，可以在github全站，搜索 github.io，有不少基于Hexo的个人博客，可以参考下别人的实现。这里就要感谢一下：https://github.com/Neveryu/Neveryu.github.io, 从这里找到了一些灵感，才解决了之前遇到的错误。

总共分几步：
1. GitHub创建
2. Hexo使用
3. 自由配置主题

## GitHub创建项目
创建的项目必须是 ccfriend.github.io，ccfriend 替换为自己的用户名。这样最终由github deploy的时候网址就是：https://ccfriend.github.io. GitHub默认会开启https访问。

这里建议涉及两个分支，默认分支 main 可以用来做deploy，建立一个source分支用于存放博客源码，方便溯源和查找。

## Hexo 使用
hexo 安装使用依赖npm，网上教程很多，实际操作也比较简单。

### 安装npm
网上下载npm。

### 安装hexo
``` bash
npm install -g hexo-cli
``` 
### hexo 初始化
建议先建立一个空目录，然后把初始化的内容拷贝到source分支下，用于方便本地进行git 提交。
在空目录下初始化：
``` bash
hexo init
``` 
hexo初始化后最重要的文件就是_config.yml，后面会频繁修改。

初始化成功后，把所有文件拷贝到自己本地的git 仓（source分支）下，可以用 git push 命令进行源码提交。source分支提交可在.gitignore中忽略不必要的代码。如：
``` bash
.DS_Store
Thumbs.db
db.json
*.log
node_modules/
public/
.deploy*/
_multiconfig.yml
```
### 获取next主题
next主题官网：
https://github.com/theme-next


直接 git clone 就可以：
``` bash
git clone https://github.com/theme-next/hexo-theme-next themes/next
```
下载后，在_config.yml修改主题为 next，hexo初始化后默认主题是landscape，
```
theme: next
```
themes\next 主题目录下可以选择next的主题，有四种，自由选择即可。


### hexo 生成预览
``` bash
hexo g
hexo s
```

### hexo 发布
发布前需要在本地的_config.yml中配置生成地址，这部有坑，需要注意：
这里要写最终发布的地址，不要写工程，不要写重了，或者写多了：
```
url: https://ccfriend.github.io
```
deploy 字段可以写成这样：
``` bash
deploy:
  type: git
  repo: https://github.com/ccfriend/ccfriend.github.io
  branch: main
```
或者：
``` bash
deploy:
  type: git
  repo: git@github.com:ccfriend/ccfriend.github.io.git
  branch: main
```
hexo的一键发布依赖这个插件，需要安装一下：
```
npm install hexo-deployer-git --save
```
当然每更新一次插件和模块，都要重新清除并生成，否则node modules不会更新到推送的分支上，可以合并执行：
```
hexo clean && hexo g && hexo s
```
然后执行发布命令：
``` bash
hexo d
```
一般要等一会，看到git pages成功后，就能访问自己的主页了。

## 主题优化
这部分仁者见仁了，大家自由发挥。