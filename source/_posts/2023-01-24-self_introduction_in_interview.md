---
title: self-introduction in interview
date: 2023-01-24
category: 杂记
description: how to introduce myself in an interview and common QA
---

## Brief Introduction

Hi, my name is Jiazhenwei (pronounced in Chinese). Currently, I am living in China, ShenZhen.
I would describe myself as a hardworking, reliable software engineer.
I have a bachelor's degree in science, and my major was computer science and technology.
I graduated from Yanshan university in 2019 and worked at bytedance since then. 
Tiktok- Service Architecture is the department I worked in. And I worked at tiktok for about 3 and half years.  I left tiktok last week.
Before that, I was an intern at Xiaomi and Meitu.
I have 3 and half years experience in golang development, and half a year in cpp. I learned python and rust by myself, but havn't used it in production. I'm familiar with common sql & nosql databases, such as MySQL, redis, mongo. I'm familiar with message queue, such as kafka, rocketmq. I have experience of performance optimization.  I know how to use Docker ,kubernetes and know about micorservice architecture.

## Questions about jobs

### Your skills

I have almost 4 years development experience on middleware and large scale data instensive application. Golang and CPP is the language I most familiar. And I also learned python and rust during the non-working time.

Gin/Fasthttp, and thrift is the framework I familiar with.

I have a deep understanding about most common database, including Relation Database and NoSQL, such as mysql, redis and mongo.

I am familiar with Message queue like Kafka and RocketMQ.

And I know about database migration and synchronization.

My native language is Mandarin, and I can speak fluent English and little japanese.

### Can you explain what's your main job in tiktok ?

I worked at tiktok for about 3 and half years, and i was mainly involved in 2 projects. One is desrpc, and the other is dataeyes.

#### DESRPC

Desrpc is a communication channel. We can say it is  a gateway.  A Background of this project is that tiktok were going to build an isolated idc in us, and all communication and data transfter with other idc must be proxyed and audited  by des component. DES's full name is  data exchange system, and it includes several parts, such as desrpc, des mq, des hdfs. I was one of the developers of desrpc. 

The compliance in desrpc is that desrpc needs to make sure that all request/response coming in or out the isolated idc must match the schema user registered and meet other security requirements.  If it does not match, we need to reject this request and give a reason to the client to tell why their request get rejected. 

DESRPC has several components

- **Forward & reverse proxy**

it is an edge proxy deploy in isolated idc and non-isolated idc, it was based envoy. I did some opimization work on observability on it. Such as unif metrics, support some new feature, like add or delete headers which are not in the white list. I also create a document about how desrpc filters implemented on forward & reverse proxy.

- **Nginx as L7 load balance an L4 load balance**

but not responible by us, it was handled by the lb team.

- **Gateway agent as sidecar deploy along with nginx**
  
this was responible by us. It is a sidecar deploy along with the Nginx, Nginx forward the request and response to the gateway agent, and the gateway agent will give response and some other instructions to the tlb to tell is should be passed or not. our agent validate producre can be describe in 2 parts for big picture

1. acl validate
we need to whether this source service have the authorization to invoke the destination service, if it's not, we need to reject this request and report not in allowlist error.

2. payload validate

payload validate have 3 parts, validation on query parameters, validation on headers and validation on payloads
desrpc supports 2 types of protocls, HTTP Over json and Thrift RPC. our forward proxy wraps the thrift RPC to http, so our agent only need to deal with the HTTP protocol and decode the http payload when it's thrift binarys.
validation on query and header parameter is pretty simple, we just need to check whether this parameter has registered on the IDL. For the payload, desipte it's encoding protocl, the general idea of the validation is that we parse the paylaod as an AST, and compare it to the idl control panel dispatched. (the idl will also parse into an AST). every time when we walk on the node, we need to check whether this node is at the same postition of the IDL, and whether it's type same as the IDL. if there are other validate rules, like PB valdiate, aggregate data valdiate and on demand sync validate (this rule is the extra field of this node of IDL AST). we need to execute these rule on these field. and return it's validate results.

after the validation is finished, we need to upload samples to the TTP bucket based on the sample algorithm. and print some audit logs.

- **Control center**
all DES components share one control center. This is where our users reigster their schema. we were responible for part of it. Because we have some special logic on RPC side. We need to parse the idl into the form which our agent can understand and combine it with the information annotated by the user, like what is field do, what is the aggregated data threshold, and is it on demand sync field, should we do OnDemandSync Validation on this field.

There are several jobs i have done in desrpc.
First thing is collaborating with technical trust partner to set up desrpc in their environment, and compile security assurance requirements, such as traffic sampling, exemption fields.
Second is collaborating with other engineers in tiktok to adjust their http traffic and make sure all of them have joined desrpc.
Third is design, develop, and implement sereval validate features, such as PB Over thirft/aggregated data/on demand sync.
Built and optimized  desrpc stability and obervability by doing load test, setting up monitor dashbboard and creating alarms.

#### DataEyes

DataEyes is the other project i worked on.  It is used to ensure a database is consistent with another. It support mysql, redis and mongo, and support comparion between any two of them. it also supports user defined plugin, so our users can define the compare or fix logic by themselves.
It's  basically a personal project. I was the only one responsible for it. I took over from  another colleague when I first stepped into the company.  And it's was just a mvp. it had a basically usable interactive interface, and hard to extend. I took it over and reconstructed the second version.  I designed, developed  it and worked with a front end engineer to have better user experience.  And developed more features, making it more stable and more easy to extend.
It has two parts, one is contron panel, responilbe to interact with the front page, and dispatch the task to the data panel.  and the other is the data panel, which does the actual comparsion and fix work. Our users fill the necessary information about the database and  submit their tasks on the front page. Then the control panel will split  the task into partitions and deliver the partition to data panel workers. Data panel will fetch data from the database and call the user plugin to get the comparison & fix result. Then report the result to the control panel periodically so our users can see it at the front page and save the running status. Also, the data panel will export the record, includes compare result , data before fixing, data after fixing, to the data warehouse for analysis.

### Why fasthttp so fast

1. The worker pool model is a zero allocation model, as the workers are already initialized and are ready to serve, whereas in the stdlib implementation the go c.serve() has to allocate memory for the goroutine.
2. The worker pool model is easier to tune, as you can increase/decrease the buffer size of the number of work units you are able to accept, versus the fire and and forget model in the stdlib
3. The worker pool model allows for handlers to be more connected with the server through channel communications, if the server needs to shutdown for example, it would be able to more easily communicate with the workers than in the stdlib implementation
4. The handler function definition signature is better, as it takes in only a context which includes both the request and writer needed by the handler. this is HUGELY better than the standard library, as all you get from the stdlib is a request and response writer… The work in go1.7 to include context within the request is pretty much a hack to give people what they really want (context) without breaking anyone.

## Open questions

### Why leave TikTok

I wanted to transfer to another department in Singapore, but I failed at the last step. The reason why I want to transfer is that I wanted to change my technical stack and expand my vision. After it failed, I think there is no reason for me to keep staying in TikTok so I left. I want to rest for a while. And keep searching for opportunities to go abroad or remote work.

### Why do you want to be an engineer

I think my personality is more suitable for an engineer. I can get positive feedback from doing things, like writing a high-quality code, and making a successful program. But it's hard for me to get positive feedback from social activities. I am not saying I can not handle collaboration and communication, I mean can gain joy from them.

### Why do you want to go to Japan? Have you ever been to Japan before?

### What is the biggest challenge in your work experience? How do you solve it?

### What do you see yourself in three years as an engineer

~~I think I'm going to be fullstack engineer, and an specialist in an area.~~
My current plan to get an solution architecture certificate. Then a data analylist certificate, front end certificate. Game developer certificte.

Then I will have a full picture about how to build a modern product. In that case, either I'll work for myself, or I'll become a specilist in an area.

### Why choose this company?

### How's your salary in TikTok

### Can you Speak japanese ?

I learned Japanese language for about half a year, i havn't practise it for awhile, maybe two weeks?  I know the 50 Japanese letters, katakanas and hiraganas. I can say and understand simple sentences. I can give a brief introduction about myself in japanese.

初めまして、私は jiazhenwei です。二十七歳です。中国人です。私の仕事はIT エンジニアです。日本語を勉強しています。よろしくお願いします。
はじめまして、わたしは jiazhenwei です.にじゅうななさいです。ちゅうごくじんです。わたしの仕事はITエンジニアです。にほんごをべんきょうしています。よろしくおねがいします。

I think this is all for now.

## My Question

### What does the company do and what business does it do

