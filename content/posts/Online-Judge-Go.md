---
title: "Online-Judge-Golang"
date: 2022-06-28T09:55:37+08:00
draft: false
tags: [online-judge]
categories: [side-project]
author: "Jimmy_kiet"
isCJKLanguage: true
---

## 前言

離上一篇畫大餅的文章後過了三天就開始動工了，還算不錯：）繼續保持！

## Day 3

今天主要就是選擇go的web框架，這邊選的是gin，單純就是該框架的網路資源目前看起來是最多的。

簡單弄一個router就可以收工了。
```
r.GET("/problems", func(c *gin.Context) {
    c.String(http.StatusOK, "There are no problems set yet.")
})
```

## Day 4

定義Problem.go跟TestCase.go

```
type Problem struct {
	Id          string     `json:"id"`
	Title       string     `json:"title"`
	Description string     `json:"description"`
	TestCases   []TestCase `json:"testCases"`
}
```

裡面的field要用成大寫的，不然在c.Json的時候會有些問題...
> Go 语言也有 Public 和 Private 的概念，粒度是包。如果类型/接口/方法/函数/字段的首字母大写，则是 Public 的，对其他 package 可见，如果首字母小写，则是 Private 的，对其他 package 不可见。

## Day 5

目標: 處理api structure

這邊用api分組的方式來避免寫很多重複的uri prefix
```
problems := r.Group("/problems")
{
    problems.GET("/", getProblemsHandler)
    problems.POST("/", createProblemsHandler)

    problems.GET("/:id", getProblemByIDHandler)
    problems.PUT("/:id", updateProblemByIDHandler)
    problems.DELETE("/:id", deleteProblemByIDHandler)
}
```

而傳入的JSON body轉為struct的方式則是如下
```
err := c.Bind(&newProblem)
if err != nil {
    c.String(http.StatusBadRequest, fmt.Sprintf("create problem err: %s", err.Error()))
}
```

> 每次更新source files就要把process關掉再重啟才能看到變化 -> 太麻煩了，所以我用Hot Reload。

```
$ go get github.com/codegangsta/gin
$ gin --appPort 8080 run .
```

## Reference

[go-module](https://geektutu.com/post/quick-golang.html)

[go-gin](https://geektutu.com/post/quick-go-gin.html#%E7%AC%AC%E4%B8%80%E4%B8%AAGin%E7%A8%8B%E5%BA%8F)

[go-gin-JSON](https://zhuanlan.zhihu.com/p/108112752)

[go-request body JSON to struct](https://matthung0807.blogspot.com/2021/08/go-gin-receive-request-json-as-struct.html)


