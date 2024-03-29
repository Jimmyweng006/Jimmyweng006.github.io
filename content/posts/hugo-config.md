---
title: "Hugo Config"
date: 2022-06-19T15:36:07+08:00
draft: false
tags: [Hugo]
categories: []
author: "Jimmy_kiet"
isCJKLanguage: true
---

## 前言

終於從Hexo轉到Hugo啦。一方面是想跟過去的自己說掰掰，重新開始一段新的旅程。
另一方面就是想透過文章記錄自己的學習，所謂output學習法嘛。最後就是Go這個語言真的給了我一種很潮的感覺（？
所以就開始弄新的Blog了，一開始免不了就是要來一段配置，不過有了之前Hexo的經驗這次就沒有這麼迷茫了哈哈。

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

不知道為什麼照別人的文章步驟去走總是有一些問題，
直到我照著這個[影片](https://www.youtube.com/watch?v=psyz4UPnGAA)的說明走完後CD才會正常，分享給大家參考。

我的作法跟影片中有一點不同的地方在於，我的repo是username.github.io，所以會自動執行pages build and deployment這個action。

## Reference

[posts template](https://jimmylin212.github.io/post/0004_hugo_other_and_cheat_sheet/)

[LoveIt theme author config](https://hugoloveit.com/zh-cn/theme-documentation-basics/#21-%E5%88%9B%E5%BB%BA%E4%BD%A0%E7%9A%84%E9%A1%B9%E7%9B%AE)