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

### Process & Thread & Process/Thread pool

* Process: 在記憶體中執行的程式(Program)
    * 資源分配的最小單位
    * 由kernel執行context switch
    * context switch: 從執行A process轉為執行B process，需要先記錄A process當前的狀態，再去讀取B process之間的狀態，再開始執行B process。
    * logic cpu = cpu 個數 * cpu 核心數 -> 同一個時間點有多少計算單位能使用
* Thread: 程式碼片段實際的執行者，包含在Process內
    * 運算排程的最小單位
    * 同一個Process內的threads可以共享Process的記憶體空間 -> 資源共享競爭(Race Condition)
* Concurrent v.s Parallel
    * Concurrent: 透過context switch達成很多process能被cpu執行
    * Parallel: 很多process同時被很多cpu執行
* Process/Thread pool
    * 提前建立好Process/Thread，這樣可以減少建立和結束Process/Thread的資源消耗。

## Reference

[Process & Thread & Process/Thread pool](https://pjchender.dev/computer-science/cs-process-thread/)

