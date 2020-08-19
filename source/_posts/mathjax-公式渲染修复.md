---
title: mathjax 公式渲染修复
top: false
cover: false
toc: true
mathjax: false
reprintPolicy: cc_by
date: 2020-05-19 13:52:59
author:
password:
img:
summary:
categories: hexo
keywords: mathjax渲染 hexo
tags:
	- mathjax渲染
	- hexo
---

## 问题

hexo默认使用**hexo-renderer-marked**引擎去渲染网页，它会把利用**Markdown语法**写的文本去转换为相应的**html标签**，使得我们写的MathJax公式被错误渲染，没法正确显示出来。

## 解决

```bash
npm uninstall hexo-renderer-marked --save
npm install hexo-renderer-kramed --save
```

[hexo-renderer-kramed](https://github.com/sun11/hexo-renderer-kramed)有人写好的一个修复库

重新部署后如果发现内联公式仍有渲染错误的，找到**../node_modules/kramed/lib/rules/inline.js**文件

```js
//escape: /^\\([\\`*{}\[\]()#$+\-.!_>])/,      第11行，将其修改为
escape: /^\\([`*\[\]()#$+\-.!_>])/,
//em: /^\b_((?:__|[\s\S])+?)_\b|^\*((?:\*\*|[\s\S])+?)\*(?!\*)/,    第20行，将其修改为
em: /^\*((?:\*\*|[\s\S])+?)\*(?!\*)/,
```

它取消了该渲染引擎对`\,{,}`的转义，然后再`hexo clean、hexo g`重新部署，即可解决问题

更多[参考](https://vitaheng.com/2017/08/03/hexo博客MathJax公式渲染问题/)