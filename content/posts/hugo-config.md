---
title: "Hugo Config"
date: 2022-06-19T15:36:07+08:00
draft: true
tags: [Hugo]
categories: []
author: "Jimmy_kiet"
isCJKLanguage: true
---

## 前言

終於從Hexo轉到Hugo啦。一方面是想跟過去的自己說掰掰，重新開始一段新的旅程。另一方面就是想透過文章記錄自己的學習，所謂output學習法嘛。最後就是Go這個語言真的給了我一種很潮的感覺（？所以就開始弄新的Blog了，一開始免不了就是要來一段配置，不過有了之前Hexo的經驗這次就沒有這麼迷茫了哈哈。

## 常用指令

> 在content/posts 新增文章
```
$ hugo new posts/my-first-post.md
```
> 啟動server並預覽草稿
```
$ hugo server -D
```

## CI/CD

不知道為什麼用方法一的時候，推新的commit到main branch，明明文章就有變化，build出來的檔案卻說是沒有改變所以不需要弄新的commit推到gh-pages branch上，算了換第二種方法。

## Reference

[posts template](https://jimmylin212.github.io/post/0004_hugo_other_and_cheat_sheet/)

[deploy to github page](https://www.zoeydc.com/zh/posts/2021-05-23-hugo-website_github-pages_custom-domain/) 我是用方法二

[LoveIt theme author config](https://hugoloveit.com/zh-cn/theme-documentation-basics/#21-%E5%88%9B%E5%BB%BA%E4%BD%A0%E7%9A%84%E9%A1%B9%E7%9B%AE)