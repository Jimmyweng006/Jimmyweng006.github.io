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

因為不想每次面試的時候像是背八股文般地去準備，所以還是花些時間好好吸收，整理成自己的知識吧...

這篇提及的內容主要依照此[文章](https://visonli.medium.com/%E6%89%BE%E5%B7%A5%E4%BD%9C%E5%BF%83%E5%BE%97-%E8%BB%9F%E9%AB%94%E5%B7%A5%E7%A8%8B%E5%B8%AB-cfa2db407f0a)提到的東西為主，秉持著貴精不貴多的精神來好好學習吧：）

## Operating System

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

* Blocking I/O
    * 定義: OS提供的read()是阻塞的，也就是需要真的讀到資料後thread才會繼續往下執行。
    * 應用層的處理方式: 一個client請求，就給它一個thread去處理。
    * 問題: 大量的requests同時進來的話，沒辦法創造那麼多個thread去處理。
    * 為何需要那麼多thread:
    ```
    從網路讀取資料後存到memory的過程:
        User Space    |    Kernel Space         |    Outside Space
                (1) ->     (2)            (4) ->
        user cache    |    kernel cache         |       Network
                    <- (3,6)                  <- (5) 
    當thread A執行到步驟(2)，如果read()的時候kernel cache是空的，那就沒有其他辦法，
    thread A只能等了，那同時也會執行context switch去執行其他thread。
    只有當資料到了kernel cache後，才會context switch回thread A繼續執行。
    ```
* Non-Blocking I/O
    * 定義: OS提供的read()不是阻塞的，當資料還沒到達kernel cache時，會直接返回-1。
    * 問題: 資料從kernel cache到user cache這段還是阻塞的。
* Multiplxing I/O
    * naive的做法: 一個client connection對應到一個fd(file descriptor)，把fd都放到array裡面。
    然後輪詢這些fd，如果read()返回的不是-1，就代表資料就緒了。
    * 問題: 但是這樣會有很多無謂的system call(read()返回-1)，user space跟kernel space的context switch也很頻繁。
    * select: OS提供的system call，由OS在kernel space檢查fd的狀態。再返回就緒的fd給user space做read()。
    這樣的開銷是select(1 system call) + nready fd(n system call)
    * poll: select的進階版，監聽的fd數量變多。
    * epoll: 針對select的三個地方改進。
        1. 每次需要傳入fd array的copy -> kernel有一份fd的集合，每次只需告訴kernel有變更的fd即可。
        2. 輪詢(同步)找到就緒的fd -> 異步喚醒就緒的fd
        3. user space遍歷整個fd array找到就緒的fd -> kernel只返回就緒的fd給user
    * 一個thread監聽多個fd: 這個特性在naive的作法也能達成，但是會多很多無謂的system call。
    * 結論: 從原本的一個connection就有一個thread；到用一個thread監聽多個fd，等到有fd就緒後再另外開thread去執行。
    這樣就能應付一般的高併發場景了吧（？
* 非阻塞I/O - Reactor
    * 使用到I/O多路復用(Multiplexing): 在linux是使用了epoll這個技術
    * 方式: 有事件發生時(有資料可以讀、新的連線等等)，就會有指定的handler處理。
    * 注意: 如果主要是CPU使用量大的requests，使用reactor沒有比較好，因為會把process阻塞住。
* 總結
    * 系統I/O使用量大 -> reactor
    * 系統CPU使用量大 -> multiprocess/multithread

## Computer Network

### OSI 物理層、資料連結層、網路層

* MAC Address(Media Access Control Address): 有網路卡的設備都會有一個唯一的MAC address。
* IP Address: 32 bit長度，會分成四個區塊，通常以10進位表示。e.q. 192.168.0.1
* Subnetwork: 同一個sub network的設備傳輸packet時，會把packet交給交換器；不同的話會交給Default Gateway。
    * Subnet mask: 來源IP跟目標IP去跟來源IP的Subnet mask做"&"操作後，得出的結果相同則代表是在同一個子網路。
    * 不同子網路的傳輸需要透過router，router的IP在每一台設備上會被設定為Default Gateway。
    * 192.168.0.1 (255.255.255.0) 可以表示成 192.168.0.1/24，也就是IP為 192.168.0.xxx 的都屬於同一個子網域。
* MAC address table: for switch(交換機)。紀錄著MAC地址與端口的對應。
* Route table: for router(路由器)。紀錄著IP地址與端口的對應。 IP: 192.168.0.1/24 <-> port: 1
* ARP(Address Resolution Protocol) table: for computer and router。記錄著IP地址與MAC地址的對應。
* 總結：
```
當A要跟F溝通時: A(192.168.0.1), F(192.168.2.2)

1. 透過與Subnet mask做"&"計算得出是否在同一個Subnetwork
2. 不同Subnetwork -> A 透過ARP 得到Default Gateway的MAC address。

3. A把原本的packet包上Data-Link Header(MAC address)跟Network Header(IP address)，丟到switch
4. switch找到這個MAC address對應的port然後丟出去
5. 此時router1接收到這個packet，發現這個packet的IP address對應到route table有下一跳
6. 中間省略，就是一直跳到有一個port能把packet丟出去為止

7. router2把這個packet的IP address對應到一個port
8. router2同時也透過ARP table找到這個IP address對應的MAC address
9. switch3收到後，把packet從MAC address對應的port發出去
10. F收到packet且packet的MAC address與自己的相符，大功告成！

上面3~6的步驟對應了OSI layer 7 ~ 5(bottom-up)
上面7~10的步驟對應了OSI layer 5 ~ 7(top-down)
這過程中MAC address會不斷變化，而IP address不會。

透過 switch(Data-Link Layer) -> router(Network Layer)的幫助，
只要有兩台機器的IP address，就能將他們相連了，可喜可賀！
```

### OSI 傳輸層

原本是電腦之間傳輸封包，但因為現在有很多process運行在電腦上。
如果還是只有前三層的功能的話，我們不知道傳過來的封包具體要給哪個process。

這個時候就輪到port登場了，給每個process一個unique的port，然後封包上再加上來源port跟目標port，
這樣就不會有不知道要把封包給哪個process的問題啦。

* 丟包問題: A一次丟n個封包給B，B需要回n個ack包給A，A才會繼續下一輪的傳送。
* 順序問題: A發送的封包順序可能跟B接收到的封包順序不同。
    * 解決方法: 給B發送的封包序號(seq)，假設為x，而回傳回來的ack包也要有序號，則為x + 1。
    x + 1代表著前面x個封包(seq <= x的封包)都已經收到了，接下來要收到第x + 1這個序號的封包。
* 流量問題: A發送給B封包，如果發送的量超過B可以接受的量(這裡應該是指kernel cache)就會爆掉
    * 解決方法: 現在封包不僅有seq，還有win(窗口大小)表示傳送方的接收封包能力
    * three pointer
    ```
    l: 收到的最後一個確認號(seq = 3開始的封包還沒確認)
    r: window upper bound = l + win
    k: 已發送的最後一個封包的seq(下一個要發送的封包seq = k + 1)
    2   3   4   5   6   7   
            k
    l                   r
    ```
    * win值變化
        * 變大: r變大，窗口就相對變大了。
        * 變小: r不變，也不繼續發封包，而是等新的ack包回來(l變大)，窗口就相對變小了。
* 壅塞問題: 跟流量問題相似，但是這邊是跟網路環境壅不壅塞有關。不像上面的流量問題會有接收方告知win值，網路環境無法告訴A，需要由A判斷win值。
    * 基本判斷: 窗口大小 = min(壅塞控制窗口大小, 流量控制窗口大小)。因為總是受限於比較弱的那方。
* 連接問題: 如果接收方處於無法接收的狀態就開始送封包，這樣也只是浪費而已。
    * TCP與UDP的差別: 連接/非連接，可靠/不可靠(丟包問題)。
    * 三次握手建立連接: sync(client) -> ack + sync(server) -> ack(client)
        * 為何是三次不是兩次: server需要最後一個ack包來確認連接已建立
    * 數據傳輸: 將數據切成一段一段的byte，每一段有自己的seq。
        * sync message = 序號(seq) + 長度 + 資料
        * ack message = 序號(seq) + 長度
    * 四次揮手斷開連接: fin(client) -> ack(server) -> fin(server) -> ack(client), 此時client等待一段時間後關閉
        * 為何client最後需要等待後再關閉: 最後的ack如果沒有送達server，server可以重發fin，然後client會重發ack再重新等待。所以如果沒有這個機制的話，也就是client發完最後一個ack就直接關閉，那就沒有補發的機會了！

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

[Blocking/Non-Blocking/Multiplxing I/O](https://mp.weixin.qq.com/s?__biz=Mzk0MjE3NDE0Ng==&mid=2247494866&idx=1&sn=0ebeb60dbc1fd7f9473943df7ce5fd95&chksm=c2c5967ff5b21f69030636334f6a5a7dc52c0f4de9b668f7bac15b2c1a2660ae533dd9878c7c&scene=178&cur_album_id=1700901576128643073#rd)

[Reactor](https://mark-lin.com/posts/20190907/)

### Network related

[OSI 物理層、資料連結層、網路層](https://mp.weixin.qq.com/s/jiPMUk6zUdOY6eKxAjNDbQ)

[OSI 傳輸層](https://mp.weixin.qq.com/s?__biz=Mzk0MjE3NDE0Ng==&mid=2247491962&idx=1&sn=aa4414483edaba487c080e91ad0efb93&chksm=c2c59bd7f5b212c12231394c585f3b063b0b2d5b05d6f05fddccdb4e856875e7ee1127bb30a7&scene=178&cur_album_id=1700901576128643073#rd)

[TCP vs UDP](https://www.youtube.com/watch?v=Iuvjwrm_O5g)