---
title: 用Hexo建立Git Hub 博客
date: 2019-12-17 11:33:32
tags:
categories: 
    - 工具
    - 文字
---

## 主要链接
1. Hexo安装 https://hexo.io/docs/setup
2. Hexo发布到GitHub https://hexo.io/docs/github-pages
3. Next主题安装 https://github.com/theme-next/hexo-theme-next
4. Next主题使用教程 http://theme-next.iissnan.com/

## 添加插件
1. 站长工具 cnzz
目前v7.6.0版本安装cnzz特别简单，只需要将创建的cnzz的id写入_config.yml里cnzz_siteid即可，无需添加其他任何文件。
如何申请友盟，请参考https://www.jianshu.com/p/3025b0e221bf

## 问题解决
1. 当博客域名是子URL时，例如https://xxx.github.io/blog，则修改站点配置文件如下：
```yaml
# URL
## If your site is put in a subdirectory, set url as 'http://yoursite.com/child' and root as '/child/'
url: https://gaocher.github.io/blog
root: /blog/
permalink: :year/:month/:day/:title/
permalink_defaults:
pretty_urls:
  trailing_index: true # Set to false to remove trailing 'index.html' from permalinks
  trailing_html: true # Set to false to remove trailing '.html' from permalinks
```

## 参考文章：
https://www.ezlippi.com/blog/2016/02/jekyll-to-hexo.html
https://github.com/EZLippi/hexo-theme