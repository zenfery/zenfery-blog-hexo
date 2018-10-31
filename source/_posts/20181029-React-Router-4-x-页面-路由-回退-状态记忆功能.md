---
title: 'React Router 4.x 页面(路由)回退, 状态记忆功能'
permalink: react-router-4-goback
categories:
  - 前端
  - React-Router
date: 2018-12-29 20:22:22
updated: 2018-12-29 20:46:00
author: Zenfery
tags:
  - 单页面
  - 组件化
thumbnail:
blogexcerpt: 使用 React 进行前端单页面应用开发，常用的页面路由组件为 React Router。那么在使用 React Router 时，如何实现路由（页面）回退时，状态记忆功能呢？
---

使用 React 进行前端单页面应用开发，常用的页面路由组件为 React Router。那么在使用 React Router 时，如何实现路由（页面）回退时，状态记忆功能呢？

首先，了解一下，何为页面状态记忆？在此，以一个示例来说明，看一下，以下两张阿里云管理控制台的截图，A 图为云主机列表页面，B 图为点击 A 右侧的“管理”按钮后的跳转页面; 在 B 页面操作后，点击左上角的“返回”，页面会再返回 A 页面；此时，A页面中的条件选择框中还能记忆住之前选择的时间条件，这种交互，称之为状态记忆。

- 图A:
  {% asset_img react-router-4-gobakc-1.png %}

- 图B:
  {% asset_img react-router-4-gobakc-2.png %}

了解了何为状态记忆后，下面就来实现此功能。

### 传统解决办法（非单页面开发）
在传统的多页面开发时，一般会采用在 A 页面的 URL 携带参数的办法，打开 B 页面时，采用新页面打开（新 Tab 页或新窗口），点击返回，直接将 B 页面关闭。

### React Router 中提供的办法
在使用 React 开发单页面应用时，一般不会打开新页面。B 页面返回，一般采用以下方法：
``` javascript
this.props.history.goBack();
```

而使用此方法返回，A 页面的状态是不能记忆的；需要在 A 页面，在修改条件（或提交查询时），将之前的查询条件保存到 React Router 的路由历史中，使用以下方法：
``` javascript
this.props.history.replace(this.props.location.pathname, state);
```
以上 repalce 方法，第一个参数为当前页面的 url；第二个参数为要保存的状态。在 A 页面中，可以使用以下方法获取到之前保存的状态，来处理相应的逻辑：
``` javascript
let state = this.props.location.state;
```
