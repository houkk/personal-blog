---
title: Hello World
date: 2017-12-23 16:15:40
categories: hexo
tags: [hexo, next]
---
## 使用 hexo - next 搭建 github pages 注意事项
`next version`： `5.1.3`

文件内容只是一些比较碎的注意事项， 详细流程请参照： [next 官网](http://theme-next.iissnan.com/)
<!-- more -->

### 1. 更换 next 主题

```
$ hexo init site_name # 创建你的 hexo 项目目录
$ cd site_name
$ git clone https://github.com/iissnan/hexo-theme-next themes/next # 下载主题

```
        
此时，项目目录下有两个 _config.yml 配置文件：
1. ./_config.yml                       # `站点配置文件`
2. ./themes/next/_config.yml # `主题配置文件`
然后修改`站点配置文件` **theme: next** 即可

### 2.  部署项目至 your_user_name.github.io
首先， 在 `github` 创建名为 `your_user_name.github.io` 的项目
然后， 在 `站点配置文件` 加入或配置

```
deploy:
  type: git
  branch: master
  repo: https://github.com/your_user_name/your_user_name.github.io.git
```
最后， 开始将 hexo 项目推至 github pages

```
$ hexo clean
$ hexo generate
$ hexo deploy
```

### 3. 域名配置
两种方案：
>方案一： 在 your_user_name.github.io 项目中点击 `setting`, 配置 custom domain （缺点： 每次 deploy 需要重新配置）
>方案二： 在 hexo 项目 source 目录下， 创建 CNAME 文件存储域名

### 4. 静态文件存放
由于每次 deploy 都会生成新的 ./.deploy 文件夹并推送至远端，所以类似 images、CNAME、README 文件都要放在 source 文件夹下， 否则将不会推送至远端。

### 5. 文件`内容头`说明
在每篇文章的头部都会有说明信息供解析， 不多说， 看示例
```
---
title: Hello World
date: 2017-12-16
categories: test
tags: [test]
---
```

### 6. 首页 `read more` 设置
在 *.md 文件中加入 `<!-- more -->`， 自由控制显示内容， 其他方式不做推荐，不做介绍

### 7. 语言设置
语言设置最好设置固定语言（个人推荐）， 实测如果不设置，可能会出现乱七八槽的语言；
`站点配置文件`
```
language: en
```

### 8. gitment
首先， 在 github [developer setting](https://github.com/settings/applications/new)  `Register a new OAuth application`
```
application name # 随意
Homepage URL # https://your_user_name.github.io || custom domain
Authorization callback URL # https://your_user_name.github.io || custom domain
```
>其中， 两个 URL 视需求而定， 暂不支持多URL；
>创建后会生成 client_id、client_secret 供使用

然后， 创建 一个项目 `gitment-comments` ， 项目名随意， 下面配置会用到，用来存储回复内容；
最后， 在 `主题配置文件`中配置：
```
gitment:
  enable: true
  mint: true
  count: true
  lazy: false
  cleanly: false
  language: # Force language, or auto switch by theme
  github_user: # your user name
  github_repo: gitment-comments # 存储回复内容的项目名称
  client_id: # client_id
  client_secret: # client_secet
  proxy_gateway: # Address of api proxy, See: https://github.com/aimingoo/intersect
  redirect_protocol: # Protocol of redirect_uri with force_redirect_protocol when mint enabled
```

### 9. 搜索功能推荐
本文章推荐使用 local search

```
$ npm install hexo-generator-searchdb --save # 安装插件
```
`站点配置文件`（不做解释）：
```
search:
  path: search.xml
  field: post
  format: html
  limit: 10000
```
`主题配置文件`：
```
local_search:
  enable: true
  # if auto, trigger search by changing input
  # if manual, trigger search by pressing enter key or search button
  trigger: auto
  # show top n results per article, show all results by setting to -1
  top_n_per_article: 1
```






