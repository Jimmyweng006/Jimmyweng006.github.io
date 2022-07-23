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

## Day 6

今天主要就是透過Postman去測試Day 5寫的那些api是否正確。

for loop range的時候拿到的值不是直接reference的，所以update的時候需要透過index去改。

```
code:
nums := []int{10, 20, 30, 40}
for i, num := range nums {
    if i == 0 {
        num = 50
    }
    fmt.Println(i, num)
}
fmt.Println(nums)

output:
0 50
1 20
2 30
3 40
[10 20 30 40]
```

## Day 7

Q: 處理create problem帶的參數錯誤時，應該要回傳400 Bad Request的情況。

A: 前面的error handling處理好這部分了，不過要記得提早return不然後面的code還會被執行到。
```
if err != nil {
    c.String(http.StatusBadRequest, fmt.Sprintf("create problem err: %s", err.Error()))
    return
}
```

## Day 8

主要在弄Database的設定。

* auto increment title id(SERIAL), date persistency -> database
* Database normalization: Testcases Table use problemId as foreign key
```
資料表裡面的值，不需要再另外拿去做別的事情才能知道它代表的意義是什麼。
在這種一對多的關係裡面，它會傾向於在多的那方增加欄位去說明它是屬於一的那方的哪筆資料。
```

## Day 9

> 今天主要在處理RESTful apis如何使用orm來去跟資料庫互動。

* ORM: 程式跟資料庫之間的橋樑
* 優點:
    1. 防止SQL Injection
    2. 簡單化重複性SQL
    3. 通用性(轉移到不同資料庫也沒問題)
* 缺點:
    1. 效能稍差(多了從orm轉換成SQL這個處理)
    2. 學習曲線高
    3. 複雜查詢支援度低

這邊選擇GORM，老樣子初入門的話先從資源多的工具下手吧！

* install:
```
$ go get -u gorm.io/gorm
$ go get -u gorm.io/driver/postgres
```
* initDatabase & create Table:
```
// init Database
func initDatabase() (db *gorm.DB, err error) {
	dsn := "host=localhost user=postgres password=123456789 " +
		"dbname=onlinejudge-go port=5432 sslmode=disable"
	db, err = gorm.Open(postgres.Open(dsn), &gorm.Config{})

	return db, err
}

// create Table
db, err := initDatabase()
if err != nil {
    fmt.Println(err)
    return
}

// no transaction yet
db.AutoMigrate(&ProblemTable{}, &TestCaseTable{})
```
### Foreign Key

天阿有點麻煩，不知道tag應該擺在哪裡...

### API to Database CRUD

~~目前看來只要struct裡有ID就可以拿來做DB相關的操作，所以不太需要特別弄一個PostDTO的struct。~~

還是需要的，因為不希望Post帶過來的body裡有ID這個欄位。

前三個API測試完成，主要就是熟悉transaction的寫法以及orm如何做query。

* Update & Delete Concept

```
* PUT /problems/{id}: 
    * testcases in body: {A, B, C}, testcases in database: {A, B, D}
    1. testcase exist{A, B} -> override it
    2. testcase not exist{C} -> create it
    3. testcase disapper{D} -> delete it
* DELETE /problems/{id}: A <- B, B depend on A, if you want to delete A, delete B first
```

* Update

~~原本的update有點麻煩啊...試試看把有requested Id的testcase都先從資料庫刪掉，再把新的testcase資料加回去？~~

還是別亂搞吧，刪了再加感覺對B+ tree負擔很大...

* driver: bad connection

不太確定具體的問題是什麼，不過根據查到的資料，目前的猜測是Next()跟Delete(), Update()同時弄在一起就會有問題。

猜對囉，果然不自己動手寫寫看都不知道會遇到什麼問題哈哈...

```
for rows.Next() {
    var testcase TestCaseTable
    tx.ScanRows(rows, &testcase)

    updatedTestcase, ok := newTestcasesMap[strconv.Itoa(testcase.Id)]
    if !ok {
        tx.Delete(&TestCaseTable{Id: testcase.Id})
    } else {
        tx.Model(&TestCaseTable{Id: testcase.Id}).Updates(
            TestCaseTable{
                Input:          updatedTestcase.Input,
                ExpectedOutput: updatedTestcase.ExpectedOutput,
                Comment:        updatedTestcase.Comment,
                Score:          updatedTestcase.Score,
                TimeOutSeconds: updatedTestcase.TimeOutSeconds,
            })
    }
}
```

## Day 10

休息了一個禮拜要回來動工啦><

> 建立會員系統與登入登出機制(Session-Based)

### 會員系統

雜湊演算法有著不可逆的特性，很適合拿來加密使用者的密碼，這裡選擇使用SHA-256。

### 登入登出機制

登入成功 -> server紀錄這個登入狀態，~~也就是session {userId, authority}~~ -> 之後的request會在http header裡的cookie
帶上session id給server驗證。

上面記錄登入狀態的部分有點理解錯誤，server只會紀錄 {userKey(固定的string), userId}這樣而已，也就是對應到這段code
```
session.Set(userKey, userId)。
```
authority的部分就等下個章節來處理。

* Wanted to override status code 400 with 200

在transaction裡面使用Abort，也只會停止執行接下來的middleware，不會讓整個handler結束，就會導致上述的問題。

最後用了個dbError的變數暴力解決這個問題了哈哈。

## Day 11

> submission API

來還之前的技術債囉，session現在要存放{userKry, UserIdAuthorityPrincipal}

* securecookie: error - caused by: securecookie: error - caused by: gob: type not registered for interface:

看來只是一些[encode問題](https://github.com/gin-contrib/sessions/issues/39)

* cannot use session.Get(userKey) (type interface {}) as type UserIdAuthorityPrincipal in assignment: need type assertion

用到了[type assertion](https://go.dev/tour/methods/15)

## Day 12

內容跟[這裡](https://jimmyweng006.github.io/posts/online-judge/#day-12)相同。

## Day 13

> 定義ISubmissionSource, ICompiler, IExecutor

> Dependency Injection，利用依賴 interface 來輕鬆替換掉裡面的實作改變程式行為，但是卻不用動到整個程式的架構程式碼。

> Architecture

```
            main.go(submissionSource, Judger)
        /                                       \
    ISubmissionSource                        Judger
1. ISubmissionSource(FileSubmissionSource, DatabaseSubmissionSource)
    1-1. functions: getNextSubmission(), setSubmissionResult()
2. Judger:
    2-1. components: ICompiler(KotlinCompiler), IExecutor(JVMExecutor)
```

### [Interface](https://blog.kennycoder.io/2020/02/03/Golang-%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3interface%E5%B8%B8%E8%A6%8B%E7%94%A8%E6%B3%95/)

### Enum

[iota](https://www.educative.io/answers/what-is-an-enum-in-golang)

[implement func (r Result) String() string {}](https://stackoverflow.com/questions/41480543/how-to-make-go-print-enum-fields-as-string)

printf("%v")的結果怪怪的，原來是enum還要implement func (r Result) String() string {}這個方法

### [File](https://zetcode.com/golang/writefile/)

### [在程式內執行Command Line](https://zetcode.com/golang/exec-command/)

### 將執行Command Line時會有的輸入輸出重新導向到文件

> 如果直接執行的話，它就會等待我們使用鍵盤輸入內容進去，並且將結果印在螢幕上

[StdinPipe](https://pkg.go.dev/os/exec#Cmd.StdinPipe)

[read from StdoutPipe](https://stackoverflow.com/questions/46723308/streaming-exec-command-stdoutpipe)

### cannot use tempCompiler (variable of type *KotlinCompiler) as *ICompiler value in struct literal

怪了KotlinCompiler這個struct已經implement ICompiler這個interface了啊...

看了[說明](https://stackoverflow.com/questions/13511203/why-cant-i-assign-a-struct-to-an-interface)但還是有點confuse...



## Reference

[go-module](https://geektutu.com/post/quick-golang.html)

[go-gin](https://geektutu.com/post/quick-go-gin.html#%E7%AC%AC%E4%B8%80%E4%B8%AAGin%E7%A8%8B%E5%BA%8F)

[go-gin-JSON](https://zhuanlan.zhihu.com/p/108112752)

[go-request body JSON to struct](https://matthung0807.blogspot.com/2021/08/go-gin-receive-request-json-as-struct.html)

[orm](https://ithelp.ithome.com.tw/articles/10207752)

[gorm-tags](https://www.cnblogs.com/zisefeizhu/p/12788017.html)

[iterate on query result](https://gorm.io/docs/advanced_query.html#Iteration)

[go-sha256](https://stackoverflow.com/questions/30956244/golang-base64-encoded-sha256-digest-of-the-user-s-password)

[gin-session](https://github.com/Depado/gin-auth-example/blob/master/main.go)

