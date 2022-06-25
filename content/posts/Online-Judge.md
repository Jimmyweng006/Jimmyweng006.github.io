---
title: "Online-Judge"
date: 2022-06-25T18:41:08+08:00
draft: false
tags: [online-judge]
categories: [side-project]
author: "Jimmy_kiet"
isCJKLanguage: true
---

# [Online-Judge-Project](https://ithelp.ithome.com.tw/articles/10233368)

## 前言

紀錄一下學習強者的project過程，目前完成了前端以外的部分。
先把預計要做的事情寫下來，也算是給自己一個動力完成吧！

1. 把project的語言從kotlin轉成go
2. deploy到AWS-ec2上面
3. 做個壓力測試跟監控

雖然整體架構跟API、資料庫都已經開好了，但技術選型(go framework, orm)可能還是要斟酌一下，加油吧！

## Day 3

* routing: route requests by two thing
    1. Method
    2. URI(Uniform Resource Identifier)
        * URI 可以：
        1. 單獨表示 URL(locator), subset of URI
        2. 單獨表示 URN(name), subset of URI
        3. 兩者兼具 (identified by locator and name)
    4. request format: [Method] [URI] HTTP/1.1
    5. response format: HTTP/1.1 [HTTP Status Code] [該 HTTP Status Code 的英文名稱]
    6. header(parameters) + body(parameters value)

## Day 4

* data schema
* ktor-jackson: convert data structure to JSON format
* Gradle: 處理專案建置的輔助工具

## Day 5

* RESTful API: 題目資源被 /problems 來代表

## Day 6

* Http Request:
    1. Accept: which format client receive
    2. Content-Type: which format client send(body)

## Day 7

* Error Handling:
    1. no correspoding route to handle request -> 404 Not Found
    2. Exception -> 500 Internal Server Error
    3. create problem with invalid body -> 400 Bad Request

## Day 8

password: 秘密
port: 5432
* auto increment title id(SERIAL), date persistency -> database
* Database normalization: Testcases Table use problemId as foreign key

## Day 9

* object 這個關鍵字是用在如果你要定義的 class 只會生出一個物件的話，你可以直接用 object 簡寫即可，最後宣告出來的結果會類似於其他語言中的 static class 的使用方式，可以直接將該類別當成一個物件來使用。
* DTO(Data Transfer Object): 
* PUT /problems/{id}: 
    * testcases in body: {A, B, C}, testcases in database: {A, B, D}
    1. testcase exist{A, B} -> override it
    2. testcase not exist{C} -> create it
    3. testcase disapper{D} -> delete it
* DELETE /problems/{id}: A <- B, B depend on A, if you want to delete A, delete B first

```
TestCases Table裡的val problem
應該要改成val problemId
才能跟POST /problems裡的變數名稱對起來？
```

## Day 10

```
fun verifyPassword(password: String, passwordHash: String) =
        BCrypt.verifyer().verify(
            password.toCharArray(),
            passwordHash
        ).verified
```

少了最後面的.verified

* 驗證機制:
    1. Session-Based Authentication: store Session on browser, and put Session Id on cookie(http request header) to authenticate when doing http request
    2. Token-Based Authentication: similar to Session-Based Authentication, but server doesn't need to store Session
    * number of users can be served: Session-Based loss(server need space to store session)
    * frequency of users to re-login: Token-Based loss(token need to expire quickly to prevent from stealing)
* 會員系統:
    * UserIdAuthorityPrincipal: use userId and authority as session
    * 設定 Session 的值為 userId 和 authority 形成的 UserIdAuthorityPrincipal 物件值，同時設定完也會讓 HTTP response 加上一段告訴客戶端之後要帶什麼樣的 Cookie 值回來。
    * after doing logout, cookie on client will also be cleared

## Day 11

* Submission API
* Authority(change problem operations, submit operation)

## Day 12

* Execution of Program
    1. 透過編譯器將程式碼變成執行檔後，在該編譯出的執行擋可執行之平台執行
    2. 透過編譯器變成中間碼檔案，此中間碼檔案可利用各平台能夠執行該中間碼的程式進行執行。
    3. 直接用直譯器去執行程式碼。
* Build environment for Kotlin code to be compiled and executed

## Day 13

* Runner proceed submissions process:
* Architecture:
```
hierarchy:
        Application.kt(submissionSource, Judger)
        /                                       \
    ISubmissionSource                        Judger
1. ISubmissionSource(FileSubmissionSource, DatabaseSubmissionSource)
    1-1. functions: getNextSubmission(), setSubmissionResult()
2. Judger:
    2-1. components: ICompiler(KotlinCompiler), IExecutor(JVMExecutor)
```

## Day 14

* connect with postgreSQL, same as Day9
* implement "DatabaseSubmissionSource" class
* Runner will take unexecuted submission, judge it and then set result

## Day 15

* result of runner compile and execute:
    * Accepted (AC)：程式通過審核。
    * Wrong Answer (WA)：程式輸出的結果有誤。
    * Compile Error (CE)：程式在編譯的時候出現編譯錯誤。
    * Runtime Error (RE)：程式在執行時壞掉。
    * Time Limit Exceeded (TLE)：程式執行時間超過規定。

## Day 16

* 容器就是基於原本的作業系統，想辦法在主作業系統上，分配部分的資源去建立起一個與外部隔離的環境。
* Docker
    * Image: 環境設定檔
    * Container: Image的實體
    * Repository: 放Image的地方
* goal: 為了要讓容器裡面的環境能夠讀取到我們的程式碼，我們會需要讓主體作業系統分享一個資料夾位置給容器，讓容器可以透過自己的檔案系統讀取到分享的資料夾內的內容。
* compile:
```
docker run --rm -v ${System.getProperty("user.dir").appendPath(workspace)}:/$workspace 
zenika/kotlin kotlinc /$codeFilePath -include-runtime -d /$executableFilePath

docker run [映像檔名稱] [指令] -> 建立並進入container後執行指令
--rm -> container執行完後自動刪除container
-v [主系統目錄位置]:[Docker 內容器檔案系統目錄位置] -> 兩個位址的內容同步
compileProcess.redirectError(ProcessBuilder.Redirect.INHERIT) -> 執行指令時產生的錯誤會導到runner的console上
```
* execute:
```
docker run --rm --name DOCKER_CONTAINER_NAME -v ${System.getProperty("user.dir").appendPath(workspace)}:/$workspace 
zenika/kotlin sh -c 'java -jar /$executableFilename < /$inputFilePath > /$outputFilePath'

java -jar [執行檔於 Docker 容器內的路徑] < [輸入檔於 Docker 容器內的路徑] > [輸出檔於 Docker 容器內的路徑]
sh -c -> 我們想要導向的是 Docker 容器內執行的指令，而非執行 Docker 的指令
--name DOCKER_CONTAINER_NAME -> 當執行executableFilename產生TLE時，
就需要先kill該container(DOCKER_CONTAINER_NAME)，才能把執行這個command的Process(executeProcess) destroy。
```

1. 先是架構不對 amd64 -> arm64
2. image跟host同一個架構後(arm64) -> Unable to find image 'dyninka/kotlin:latest' locally, 一直想用最新版的image卻不用現有的image？
3. 改成用image id取代image名稱&tag -> docker: Error response from daemon: OCI runtime create failed: container_linux.go:380: starting container process caused: exec: "kotlinc": executable file not found in $PATH: unknown. ???

## Day 17

* -p -> 這裡是將主機的 6379 號碼的 port 與 Docker 容器內的 6379 號碼的 port 連結在了一起，這樣我們才能在 Docker 容器外連進去 Docker 容器內的 Redis 資料庫所預設監聽的 6379 號的 port。
* JudgerSubmissionData(for redis)跟SubmissionData結構相同 -> runner可以直接取用redis的資料不用再做轉換
* 由於 submissionId 和 testCaseData 這兩個變數是在 transaction 裡面的 Lambda Function 被修改，導致後面使用 if 語句去判斷是否為 null 時，編譯器不知道該 Lambda Function 實際會在什麼時候執行，故無法保證這兩個值是否真的不為 null -> use two other variables to store
* Jedis.rpush([Queue 的名稱], [要丟進 Queue 內的內容]) -> redis裡面可以有很多個task queue，這裡使用code lauguage做為queue的種類。
* 推進去的 JudgerSubmissionData 資料要是 JSON 的格式字串 -> writeValueAsString([資料物件])
* readValue炸裂??? None of the following functions can be called with the arguments supplied.
```
return jacksonObjectMapper().readValue(data, SubmissionData::class.java)
```
* POST /submissions/restart: super user, unjudged submission will be judge
* POST /submissions/{id}/restart: submission belongs to that user, unjudged/judged submission will be judge
* 因為剛遞交的程式碼是在 Redis 資料庫斷線的時候遞交的，所以 Redis 資料庫並沒有收到，也就讓審核程式沒辦法拿到那些程式碼資料去進行批改。 -> call submissions/restart api -> 會把postgreSQL未judge的submission丟到redis裡

## Day 18

* support multiple languages
    1. image for compile & execute
    2. implement ICompiler & IExecutor for multiple languages
```
雖然沒有講到c++的作法，不過因為image可以跟c的共用，
所以只要在跟language相關的地方加上c++的東西即可。
```