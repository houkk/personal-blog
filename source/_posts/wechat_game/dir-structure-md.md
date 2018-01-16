---
title: 微信小游戏目录结构
date: 2018-01-14 19:52:10
categories: wechat
tags: [wechat_game]
---

本文主要讲述， 如何快速创建一个小游戏项目并且针对其目录结构进行简单讲解。
<!-- more -->

### 1. 创建项目
首先新建项目，目前小游戏不提供公开注册，可点击体验小游戏使用无 AppID 模式（创建时如果建立快速启动模板，需要选择一个空的目录）。![](../../../../../images/wechat_game/createProject.jpg)
### 2. 微信预览
选择建立快速启动模板后，将会生成一个小游戏项目，并在开发工具中打开，这时点击开发工具右上角的预览按钮，会生成对应二维码，微信扫码即可。
![](../../../../../images/wechat_game/yulan.jpg) 
### 3. 目录结构

`./audio`: 音频目录文件 <br>
`./images`: 图片文件目录 <br>
`./js`: 主要源代码目录 <br>
`./game.js`: 主入口文件 <br>
`./game.json`: 配置文件

```
./js
├── base                                   // 定义游戏开发基础类
│   ├── animatoin.js                       // 帧动画的简易实现
│   ├── pool.js                            // 对象池的简易实现
│   └── sprite.js                          // 游戏基本元素精灵类
├── libs
│   ├── symbol.js                          // ES6 Symbol简易兼容
│   └── weapp-adapter.js                   // 小游戏适配器
├── npc
│   └── enemy.js                           // 敌机类
├── player
│   ├── bullet.js                          // 子弹类
│   └── index.js                           // 玩家类
├── runtime
│   ├── background.js                      // 背景类
│   ├── gameinfo.js                        // 用于展示分数和结算界面
│   └── music.js                           // 全局音效管理器
├── databus.js                             // 管控游戏状态
└── main.js                                // 游戏入口主函数

```
> 源码分析：参考 [segmentfault的一篇文章](https://segmentfault.com/a/1190000012646888) 
