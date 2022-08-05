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
```go
r.GET("/problems", func(c *gin.Context) {
    c.String(http.StatusOK, "There are no problems set yet.")
})
```

## Day 4

定義Problem.go跟TestCase.go

```go
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
```go
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
```go
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

```go
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
```go
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
```go
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

```go
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
    2-2. functions: compile(), execute()
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

## Day 14

### 使用Dependency Injection這個pattern又生出了一些額外的問題...

先暴力解決了...

```go
// var submissionSource ISubmissionSource = &DatabaseSubmissionSource{}
var submissionSource = &DatabaseSubmissionSource{}
```

> 測試的submission的code跟題目的要求沒有對起來，導致多花了好多時間在debug以前就正確的功能:(

## Day 15

### Check CompileError

```
1. 編譯時吐出 Exception：回傳 CE。
2. 編譯後得不到執行檔檔名字串：回傳 CE。
3. 編譯後，從執行檔檔名字串找不到檔案：回傳 CE。

情況1的結果應該包含在情況2了
```

### Goroutine + Channel + Select

[Goroutine](https://www.bogotobogo.com/GoLang/Golang_GoRoutines.php)

light weight Thread

[Channel](https://www.bogotobogo.com/GoLang/GoLang_Channels.php)

Goroutines之間溝通的"管道"

```
data := <- my_channel // read from channel my_channel
my_channel <- data    // write to channel my_channel
```

[Select](https://www.bogotobogo.com/GoLang/GoLang_Channel_with_Select.php)

not like Switch, Select only use for Channel cases.

### Check RuntimeError

```
1. 如果 IExecutor.execute() 丟出 Exception：表示程式執行途中發生意想不到的錯誤，回傳 RE。
2. 如果執行完後沒有任何結果狀態：表示審核程式在執行程式時，在中途發生意外狀況而結束，回傳 RE。
3. 如果執行完後超時：回傳 TLE。
4. 如果執行途中程式壞掉：回傳 RE。

情況1的結果應該包含在情況2了
```

### 困難點

要確認command line執行的程式，是否有在timeOutSeconds內完成，這點還是蠻麻煩的...

[CommandContext + WithTimeout](https://stackoverflow.com/questions/67750520/golang-context-withtimeout-doesnt-work-with-exec-commandcontext-su-c-command)

* 看來只要<-ctx.Done()這個Channel有東西，而這個東西(ctx.Err())是context.DeadlineExceeded的話就代表超時了。

* if err := cmd.Wait(); err != nil 代表 RuntimeError

## Day 16

> [Big Misconceptions about Bare Metal, Virtual Machines, and Containers](https://www.youtube.com/watch?v=Jz8Gs4UHTO8&t=3s)

以下這段code真的把runner程式底下的a.txt刪掉了！雖然想對LeetCode實驗看看"rm -rf/"，不過怕被吉所以還是算了吧哈哈。

```
fun main() {
 val inputs = readLine()!!.split(' ')
 val a = inputs[0].toInt();
 val b = inputs[1].toInt();
 val c = inputs[2].toInt();
 ProcessBuilder("rm", "a.txt").start().waitFor()
 println("${a + b + c}")
}
```

看來之前的kotlin image架構不對的問題，是直接先放著了哈哈...

### Golang struct Constructor

沒有Constructor真的麻煩，每個struct現在都要有init()這個方法，所以連帶著interface也要增加init()這個方法了...

### Docker Compile Code

```
完整指令:
"docker",
"run",
"--rm",
"-v",
"${System.getProperty("user.dir").appendPath(workspace)}:/$workspace",
"zenika/kotlin",
"kotlinc",
"/$codeFilePath",
"-include-runtime",
"-d",
"/$executableFilePath"

分析:
docker run [映像檔名稱] [指令] -> 建立並進入container後，執行指令。
docker run [zenika/kotlin] [kotlinc /$codeFilePath -include-runtime -d /$executableFilePath]

--rm -> container執行完後自動刪除container

-v [主系統目錄位置]:[Docker 內容器檔案系統目錄位置] -> 兩個位址的內容同步
"-v ${System.getProperty("user.dir").appendPath(workspace)}:/$workspace",
```

* [redirect compile process error](https://pkg.go.dev/os/exec#Cmd.StderrPipe)


### Docker Execute Code

我這邊處理可執行檔的輸入輸出方式是用"StdinPipe/StdoutPipe"，不是再去處理file的東西。

```
完整指令:
"docker",
"run",
"--rm",
"--name",
DOCKER_CONTAINER_NAME,
"-v",
"${System.getProperty("user.dir").appendPath(workspace)}:/$workspace",
"zenika/kotlin",
"sh", 這兩個應該不用(？
"-c", 這兩個應該不用(？
"java -jar /$executableFilename < /$inputFilePath > /$outputFilePath" -> "java -jar /$executableFilename"

分析:
--name DOCKER_CONTAINER_NAME -> 當執行executableFilename產生TLE時，
就需要先kill該container(DOCKER_CONTAINER_NAME)，才能 kill 執行這個command的Process(executeProcess)。
```

### 困難點

之前遇到的問題現在大概知道是發生什麼事了，不就是container裡沒有kotlinc這個compiler嗎哈哈...

```
Command
[docker run --rm -v /Users/jimmy/Documents/side-project/online-judge-runner-go/workspace:/workspace dyninka/kotlin:dyninka kotlinc /workspace/_code.kt -include-runtime -d /workspace/_code.jar]
docker: Error response from daemon: OCI runtime create failed: container_linux.go:380: starting container process caused: exec: "kotlinc": executable file not found in $PATH: unknown.
```

## Day 17

### Multiprocess of Judge-Service

使用multiprocess的方式執行兩個Judge-Service(就是開兩個terminal執行go run . [runner] 而已)。

結果如下，兩個process都還是會去執行同一筆submission

runner 1先拿到submission -> runner 2拿到submission -> runner 1 serResult -> runner 2 setResult

大概是這樣，所以還是用個Task Queue比較保險啊！

```
go run . 1
compile file name _code1.kt
Submission 102: Accepted - Score: 100 (0.1)

go run . 2
compile file name _code2.kt
Submission 102: Accepted - Score: 100 (0.093)
```

### Redis

```
docker pull redis -> 拉Redis的image
docker run --name judge-redis -p 6379:6379 -d redis
-d 背景執行
-p 參數，去將 Docker 容器內的 port 與我們主機實際的 port 去做對應
```

查了task queue後找到[這個](https://taskq.uptrace.dev/guide/golang-task-queue.html#creating-queues-and-tasks)，研究了老半天發現其實也不用這麼複雜。

只要有rpush跟lpop即可...[參考](https://blog.csdn.net/pengpengzhou/article/details/108361064)

* [parse JSON string to struct](https://stackoverflow.com/questions/47270595/how-to-parse-json-string-to-struct)

* [defer](https://www.evanlin.com/golang-know-using-defer/)

A defer statement defers the execution of a function until the surrounding function returns.

* todo [context](https://blog.wu-boy.com/2020/05/understant-golang-context-in-10-minutes/)

### Redis出錯時的處理

#### Redis連線失效

使用rdb之前最多retry一次，用之前也要檢查連線是否存在。

```go
func getConnection(rdb *redis.Client) error {
	ctx := context.Background()
	pong, err := rdb.Ping(ctx).Result()

	if err != nil {
		fmt.Println("ping error, try reconnect", err.Error())
		rdb = redis.NewClient(&redis.Options{
			// Addr: ":6379",
		})
		pong, err = rdb.Ping(ctx).Result()
		return err
	}

	fmt.Println("ping result:", pong)
	return nil
}

if err = getConnection(rdb); err != nil {
    isOK = false
    c.JSON(http.StatusInternalServerError, gin.H{
        "redis": "disconnection",
    })
    return
}
```
#### POST /submissions/restart && POST /submissions/{id}/restart

* Q: 因為**Data-Management-Service**除了把submission記錄在Postgres，還需要額外往Redis放submission，好讓**Judge-Service**去消化掉那些submission。那如果現在Redis的submission跟Postgres的submission不一致怎麼辦(e.g. Redis斷線導致submission沒有被放進Redis，那當然也就沒有被judge)？
* A: 多了兩隻API，讓他們去Postgres找出尚未judge的submission，並且都放回Redis裡。

* restartSubmissionsHandler會遇到典型的[1 + N problem](https://segmentfault.com/a/1190000039421843)

submission table與testcase table透過problemId關聯起來。

如果今天要找出所有unjudged的submission，並將這些submission相關的testcases包成[]JudgerSubmissionData。
比較簡單的做法就需要1(找出所有result = "-"的submission) + N(找N個submission的testcase)個query。

這樣其實蠻耗資料庫的資源的(多個connection)，所以這邊的解法是
1. find all unjudged submissoins and its problemId
2. find it's related testCases
3. combine to JudgerSubmissionData and push to Redis

query數從1 + N -> 1 + 1

* 測試普通使用者使用 POST /submissions/restart這個API，status code應該為StatusUnauthorized -> 成功

* Multiprocess of Judge-Service -> 成功 

* redis: can't marshal main.JudgerSubmissionData (implement encoding.BinaryMarshaler)

```go
ctx := context.Background()
val, err := rdb.RPush(ctx, newSubmission.Language, judgerSubmissionData).Result()
if err != nil {
    panic(err)
}
```

Redis的操作還是用.Result()檢查一下比較好。Judge Service一直沒拿到資料，一開始以為是斷線問題，結果是JSON Serialize的問題... [serialize and desirialize of JSON](https://codewithyury.com/how-to-correctly-serialize-json-string-in-golang/)

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

