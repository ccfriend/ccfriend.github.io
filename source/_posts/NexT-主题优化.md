---
title: NexT 主题优化
date: 2022-05-08 11:36:23
categories: blog
tags: [blog, hexo]
---

NexT可以做进一步的美化和易用性的改进使用。

- 字数统计和阅读时长
- 统计浏览次数
- 侧边栏社交


<!-- more -->

# 字数统计

字数统计需要安装npm的一个插件：

```
npm install hexo-symbols-count-time --save
```

然后在配置文件中增加：

```
# hexo-symbols-count-time
symbols_count_time:
  symbols: true
  time: true
  total_symbols: true
  total_time: true
```

# 统计浏览次数
在 themes/next/_config.xml中把enable设置成true：
```
# Show Views / Visitors of the website / page with busuanzi.
# Get more information on http://ibruce.info/2015/04/04/busuanzi
busuanzi_count:
  enable: true
  total_visitors: true
```

# 侧边栏社交
修改配置文件的social 字段，一行一个链接。参考如下：
```
social:
  GitHub: https://github.com/ccfriend || fab fa-github
  E-Mail: mailto:ccfriend@sina.cn || fa fa-envelope
```