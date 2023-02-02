---
title: Study Note
date: 2023-01-13 08:49:22
tags: 杂记
description: study note from when i was still in bytedance, remove some duplicated stuff and internal resource
---

### Alogithm

leetcode

### Note

#### MIT 6.284

Fault Tolerance-Raft

### Blogs & Repos

some interesting articles. 
frantic.im/leaving-facebook
github.com/donnemartin/system-design-primer
zhihu.com/question/23921846
1point3acres.com/bbs/thread-839064-1-1.html
github.com/spiffe/spire
isovalent.com/blog/post/2021-12-08-ebpf-servicemesh

### Project

- DataEyes
  - storage [SQL & NoSQL]
- DESRPC


### System Design Step

1. outline use case, constraints and assumptions
2. create high level design
3. scale the design
   1. benchmark/load test
   2. profile for bottlebecks
   3. address bottlenecks while evaluating alternatives and tradeoffs
   4. repeat

### Language

#### Rust

tokio.rs/tokio/tutorial

#### Golang

zhuanlan.zhihu.com/26972862
zhihu.com/column/interview
research.swtch.com/interface
codeburst.io/diving-deep-into-the-golang-channels-549fd4ed21a8
mo4tech.com/golang-gmp-model-for-concurent-scheduling.html
developpaper.com/source-code-analysis-of-golang-channel

### Engineer

### Database

zhuanlan.zhihu.com/p/222958908

### Message Queue

### Kafka

zero copy: deveoppaper.com/what-is-the-so-called-zero-copy-technology-in-kafka
page cache: andriymz.github.io/kafka/kafka-disk-write-performance/##zero-copy-data-transfer

### Rocketmq

## Mesh & Service Registry

## Interview Note

1. how golang channel implements
2. what is the difference between process and thread
3. have met that kill a process don't workd, reason and how to deal with it. [link](https://unix.stackexchange.com/questions/5642/that-if-kill-9-does-not-workd)
4. how to communicate between process & thread
5. tcp sliding window
6. assume client and server already bulid a tcp connection, and client send data to it, what will happen if service don't read data from the tcp socket
7. how to address OOM issue
   1. slow resident memory increase: using prof tools to locate abormal memory allocate
   2. suddenly memory increase: find logs, hook memory allocate library, using tmpfs
   3. OOM when the process starts: Load large data, using tmpfs
