---
title: "Jimmy Chat Architecture"
date: 2024-01-22T08:21:11+08:00
draft: false
tags: [Backend]
categories: []
author: "Jimmy_kiet"
isCJKLanguage: true
---

## Current Architecture

![](/img/Chat-System-Origin.png)

## Requirements

### Acknowledgment of Messages

messages should have three status: sent, delivered, read.

### Push Notification of Messages

notify offline users of new messages once their status becomes online.

## Chat System Component

* Stateful Services: chat functionality
* Stateless Services: other functionality except chat
* Third-party integration: push notification

## Storage

1. generic data: user profile, user friend list -> relational Database
2. chat data -> key value stores

## API Design

1. Send Message(sender, receiver)
2. Read Unread Messages(user)

## Detailed Design

1. Service Discovery(Optional)
2. Sending or Receiving messages
    * How to send real time messages for two users which are connected to different websocket servers?
    1. WebSocket server sends the message to the message service, and message is stored in the database in case user B is offline.
    2. When the messages are delivered to the receiver, they are deleted from the database.

## Reference

[System Design: Chat Application](https://medium.com/@m.romaniiuk/system-design-chat-application-1d6fbf21b372)

