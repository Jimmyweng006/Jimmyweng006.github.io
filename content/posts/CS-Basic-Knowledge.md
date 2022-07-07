---
title: "CS Basic Knowledge"
date: 2022-06-29T11:52:26+08:00
draft: false
tags: [面試]
categories: []
author: "Jimmy_kiet"
isCJKLanguage: true
---

## 前言

不想每次面試的時候像是背八股文般地去準備，所以還是花些時間好好吸收，整理成自己的知識吧...

這篇提及的內容主要依照此[文章](https://visonli.medium.com/%E6%89%BE%E5%B7%A5%E4%BD%9C%E5%BF%83%E5%BE%97-%E8%BB%9F%E9%AB%94%E5%B7%A5%E7%A8%8B%E5%B8%AB-cfa2db407f0a)提到的東西為主，秉持著貴精不貴多的精神來好好學習吧：）

## OS

### Program & Process & Thread & Process/Thread pool

* Program: 放在Secondary Storage的程式碼 -> 工廠的藍圖
* Process: 在記憶體中執行的Program -> 根據工廠藍圖製造的工廠
    * 資源(CPU、記憶體、檔案、I/O裝置等等)分配的最小單位
    * 由kernel執行Context Switch
    * Context Switch: 從執行A process轉為執行B process，需要先記錄A process當前的狀態，再去讀取B process之前的狀態，再開始執行B process。
    * logic cpu = cpu 個數 * cpu 核心數 -> 同一個時間點有多少計算單位能使用
* Thread: 程式碼片段實際的執行者，包含在Process內 -> 工廠內的工人
    * 運算排程的最小單位
    * 同一個Process內的threads，他們各自的stack是彼此獨立的 -> 無法共享
    * 同一個Process內的threads可以共享Process的記憶體空間 -> 資源共享競爭(Race Condition)
    * Race Condition:
        ```
        int count = 0;
        Thread A 執行 count++
        Thread B 也執行 count++
        預期count = 2，但實際上count = 1。
        ```
    * 多個Process互搶資源 -> 死結(Deadlock)
    * Deadlock:
        1. 互斥: 這個資源同時只能有一個process擁有
        2. 持有並等待: 某個process拿了某些資源，又想再拿別的資源
        3. 不可強奪: process擁有的資源只能由自己主動釋放
        4. 循環等待: A等待B持有的資源，B等待A持有的資源
* Concurrent vs Parallel
    * Concurrent: 透過Context Switch達成很多process能被cpu執行
    * Parallel: 很多process同時被很多cpu執行
* Process/Thread pool
    * 提前建立好Process/Thread，這樣可以減少建立和結束Process/Thread的資源消耗。

### I/O

* I/O: Input/OutPut
    1. 文件I/O
    2. 網路I/O
* 從硬碟讀取資料，並將資料傳送到網路的流程
    ```
    User Space    |    Kernel Space         |    Outside Space
            (1) ->     (2)            (4) ->
    user cache    |    kernel cache         |    disk
                   <- (3,6)                  <- (5) 
    user cache    |    socket cache         |    network
            (7) -> <- (8)             (9) ->
    (1) 應用程式發送 system call read 某個檔案，給作業系統的內核。(用戶切內核)
    (2) 內核看看內核緩衝區有沒有相關資料。
    (3) 有則將內核緩衝區的資料，拷貝到用戶緩衝區。copy1
    (4) 無則前往硬碟取得。
    (5) 將硬碟資料拷貝至內核緩存區。copy2
    (6) 將內核緩衝區資料，拷貝到用戶緩衝區。(內核切成用戶) copy1
    (7) 在從用戶緩衝區發送 system call write 將用戶記憶體資料拷貝到 socket 緩衝區，這緩衝區也是在內核中。(用戶切內核) copy3
    (8) 拷貝到 socket 緩衝區後，用戶端收到 ack。(內核切用戶)
    (9) data on fly!
    ```
* Zero Copy Concept:
    * 原本的方法需要用到3次copy，但這是因為需要把資料帶到User Space的情況。
    * 如果不需要把資料帶到User Space，那基本上可以省下(3 or 6)的操作，也就是直接將kernel read cache的資料copy至socket cache。
* Zero Copy Implement:
    1. mmap(Memory-Mapped) + write: (3, 6)原本會回傳資料本身，現在改回傳**資料在kernel cache的位置**，之後再用write把位置上的資料傳到網路。
    2. sendfile: 流程跟Concept描述的差不多。

### Stream

* 傳統資料傳輸問題: 如同上一個主題I/O所提到的，(3 or 6)會將kernel cache的資料直接copy到user cache。但如果現在資料太大呢(10GB)？一次複製就要把這麼大的資料放到cache裡肯定是要爆的。
* Stream: 那其實只要每次傳輸的量不要那麼大(100MB)，基本就能解決上述問題了。
* IPC(InterProcess Communication): process與process之間傳輸資料的方式
    1. Process A將自己cache的資料copy到kernel cache裡
    2. kernel再將資料copy到Process B的cache裡
    * 既然這邊也是在做copy資料的操作，自然也可以應用Stream的概念囉。

### Reactor

* 非阻塞I/O(應用層的I/O操作，不會把process給阻塞住)
* 傳統I/O
    * 方式: 一個client請求，就有一個process處理。
    * 問題: 大量的requests同時進來的話，沒辦法創造那麼多個process去處理。
    * 為何需要那麼多process:
    ```
    從網路讀取資料後存到memory的過程:
        User Space    |    Kernel Space         |    Outside Space
                (1) ->     (2)            (4) ->
        user cache    |    kernel cache         |       Network
                    <- (3,6)                  <- (5) 
    當process A執行到步驟(2)，如果read()的時候kernel cache是空的，那就沒有其他辦法，
    process A只能等了，那同時也會執行context switch去執行其他process。只有當kernel cache
    的資料到了後，才會context switch回process A繼續執行。
    ```
* 非阻塞I/O - Reactor
    * 使用到I/O多路復用(Multiplexing): 在linux是使用了epoll這個技術
    * 方式: 有事件發生時(有資料可以讀、新的連線等等)，就會有指定的handler處理。
    * 注意: 如果主要是CPU使用量大的requests，使用reactor沒有比較好，因為會把process阻塞住。
* 總結
    * 系統I/O使用量大 -> reactor
    * 系統CPU使用量大 -> multiprocess/multithread

## System Design

* 最近蠻紅的[資源](https://www.youtube.com/channel/UCZgt6AzoyjslHTC9dz0UoTw)，內容算好吸收，分享給大家：）

### What happens when you type a URL into your browser?

1. enter a URL into the browser
    * URL(Universal Resource Locator): Scheme(which protocol to use) + Domain + Path + Resource
    * http://example.com/product/electric/phone -> http: + example.com + product/electric + phone
2. get IP address by look up browser/OS cache or DNS(Domain Name System)
3. establish TCP connection
4. client sends HTTP requset to server
5. server sends HTTP response to client
6. browser renders content

### How does HTTPS work?

如果只是原本的HTTP protocol，client與server之間傳輸的資料都是明碼的，如果被攔截了東西就被看光光 -> 所以我們需要HTTPS

* Symmetric encryption: 加密解密的鑰匙相同 -> 用來大量資料加密解密
* Asymmetric encryption: 加密解密的鑰匙不同 -> 用來驗證

主要有四個階段

1. TCP Handshake
2. Certificate Check: 在這個階段，client會拿到由server產生的**非對稱加密公鑰**。
3. Key Exchange: client會使用上一個階段拿到的**非對稱加密公鑰**，來去加密由client產生的**對稱加密鑰匙(session key)**，最後再將 **加密過後的對稱加密鑰匙(encrypted session key)** 傳給server。
4. Data Transimission: 之後client與server之間傳輸的資料就能透過session key加密啦，可喜可賀~

## Reference

### Process related

[Process & Thread & Process/Thread pool](https://pjchender.dev/computer-science/cs-process-thread/)

[Multi-Threading vs Multi-Processing](https://medium.com/erens-tech-book/%E7%90%86%E8%A7%A3-process-thread-94a40721b492)

### I/O related

[Zero-Copy](https://mark-lin.com/posts/20190905/)

[Stream](https://mark-lin.com/posts/20190906/)

[Reactor](https://mark-lin.com/posts/20190907/)
