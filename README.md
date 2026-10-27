## 前言

千古前端图文教程，超详细的 Web 前端入门到进阶知识库。从零开始学前端，做一名精致优雅的前端工程师。

通俗易懂，不懂技术也能学。此前端教程不玩猫腻，不会设置任何套路，因为我**相信启蒙的重要性**。

> **2026 年更新说明**
>
> 本项目原始内容写于 2017–2022 年，其中**基础部分依然扎实有效**，但**框架与工程化部分已明显滞后**（原 React 章仍在讲 `getDefaultProps`、`getInitialState` 等 2015 年 API；Vue 章基于已停止维护的 Vue 2 + vue-cli）。
>
> 本次更新**保留全部原内容**（确保历史积累不丢失），并**新增 5 个现代章节**补齐缺失层级，同时在各过时章节顶部加上「时效提示」，明确标注哪些能读、哪些代码不要照抄。详见下方 [学习路径](#学习路径2026) 与 [内容时效说明](#内容时效说明)。


## 项目介绍

### 项目地址

- 官网地址：<https://web.qianguyihao.com>
- GitHub 地址：<https://github.com/qianguyihao/Web>（上游原项目）

如果你发现本项目有内容上的错误，欢迎提交 issues 或者 pull requests 进行指正，方便归档。

### 项目作用

- 网上的大部分入门教程，都不太适合初学者，本项目争取照顾到每一位前端入门者的同理心。即使你完全不懂前端，甚至不懂编程，通过这个教程，也能让小白入门。

- 帮助前端同学提供一个精品学习资源和路线，提高学习效率，少走很多弯路。

- 可以当做前端字典，随时翻阅，查漏补缺。


## 学习路径（2026）

### 一、不可跳过的地基

这些内容**不受技术栈更迭影响**，值得完整读完：

| 章节 | 说明 |
|---|---|
| [01-HTML](01-HTML) | 语义化标签、表单、音视频 |
| [02-CSS基础](02-CSS基础) / [03-CSS进阶](03-CSS进阶) | 盒模型、选择器、层叠、定位 |
| [04-JavaScript基础](04-JavaScript基础) | 39 万字，类型、函数、对象、数组、DOM/BOM |
| [07-JavaScript进阶](07-JavaScript进阶) | 原型链、闭包、执行上下文 |
| [06-JavaScript基础：异步编程](06-JavaScript基础：异步编程) | **本项目最强的一章**：Promise 8 篇 + async + 事件循环 + 面试题 |

### 二、必须补齐的现代内容（本次新增）

| 章节 | 为什么必读 |
|---|---|
| [18-TypeScript](18-TypeScript) | 2026 年新项目的默认选项，原教程 0 覆盖 |
| [22-现代CSS](22-现代CSS) | Grid、`:has()`、容器查询、`clamp()`、CSS 变量 |
| [21-现代工程化](21-现代工程化) | Vite、pnpm、ESLint、测试、CI/CD、部署 |
| [19-Vue3](19-Vue3) | **替换**「12-Vue基础」，组合式 API + Pinia |
| [20-React18+](20-React18plus) | **替换**「13-React基础」，函数组件 + Hooks |

### 三、按需查阅

| 章节 | 说明 |
|---|---|
| [08-前端基本功：CSS和DOM练习](08-前端基本功：CSS和DOM练习) | 手写练习，巩固 DOM 操作 |
| [09-移动Web开发](09-移动Web开发) | 适配、视口 |
| [14-前端性能优化](14-前端性能优化) | 渲染机制、静态资源、防抖节流，**依然有效** |
| [00-前端工具](00-前端工具) | Git、VS Code、抓包 |
| [10-MySQL数据库](10-MySQL数据库) / [11-Node.js](11-Node.js) | 后端基础，了解即可 |


## 内容时效说明

本项目内容按"知识半衰期"分三类，读之前请先看清楚：

### ✅ 依然完全有效

HTML、CSS 基础布局与选择器、JavaScript 语言核心、DOM/BOM、浏览器渲染机制、HTTP 与跨域、Promise 与事件循环、性能优化原理。

**原因**：这些是平台级知识，浏览器十年一升级，概念不变。

### ⚠️ 概念可读，代码勿抄

| 章节 | 问题 |
|---|---|
| [12-Vue基础](12-Vue基础) | 基于 **Vue 2 + vue-cli**。`filters` 已删除、`beforeDestroy` 已改名、Vuex 已被 Pinia 取代。Vue 2 于 2023-12-31 停止维护 |
| [13-React基础](13-React基础) | 基于 **React 15**。`getDefaultProps` / `getInitialState` / `componentWillMount` 已全部移除；`bind(this)` 不再需要。今天只需函数组件 + Hooks |
| [05-ES6语法](05-JavaScript基础：ES6语法) | 止于 ES2015。缺 `?.`、`??`、`?.` 等高频新语法 |
| [15-前端工程化](15-前端工程化) | 仅名词罗列，无任何工具实操 |

### ❌ 已被取代

| 旧内容 | 替代 |
|---|---|
| 浮动 / `inline-block` 布局 | Flex / Grid（见 22 章） |
| `float` 清除浮动 | `gap` + Flex/Grid |
| Vue 2 选项式 API | Vue 3 组合式 API（见 19 章） |
| React class 组件 | 函数组件 + Hooks（见 20 章） |
| vue-cli | `npm create vue@latest` |
| webpack（新项目场景） | Vite（见 21 章） |
| Vuex | Pinia |


## 快速开始（新项目脚手架）

```bash
# Vue 3 + TS + Vite + Pinia + Router
npm create vue@latest

# React + TS + Vite
npm create vite@latest my-app -- --template react-ts

# Next.js（带 App Router / RSC）
npx create-next-app@latest
```

包管理建议统一使用 pnpm：

```bash
npm i -g pnpm
pnpm install
```


## 相关链接

- 维护这个项目的初衷：[《裸辞两个月，海投一个月，从Android转战Web前端的求职之路》](https://www.cnblogs.com/qianguyihao/p/8732781.html)
- 前端入门路线和推荐学习资源：[《Web前端开发流程和学习路线（详尽版）》](https://www.cnblogs.com/qianguyihao/p/16370961.html)


## 学习交流

在公众号「千古壹号」里回复“**前端学习**”，拉你进微信交流群：

- 进群暗号：前端学习
- 进群要求：少提问、少闲聊、多分享（不适合长期潜水）。

![](https://img.smyhvae.com/20210329_1930.png)


## LICENSE

![](http://img.smyhvae.com/20210331_CC-BY-NC-SA.png)

本作品采用[知识共享署名-非商业性使用-相同方式共享 4.0 国际许可协议](https://creativecommons.org/licenses/by-nc-sa/4.0/)进行许可。