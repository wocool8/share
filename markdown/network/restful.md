# RESTful(基于HTTP实现介绍) 
---
## REST
REST全称是Representational State Transfer，REST是[Roy Thomas Fielding](https://en.wikipedia.org/wiki/Roy_Fielding)在他2000年的博士论文[Architectural Styles and the Design of Network-based Software Architectures](https://www.ics.uci.edu/~fielding/pubs/dissertation/top.htm)中提出的。他的设计目的如下
    
    My work is motivated by the desire to understand and evaluate the architectural design of network-based application 
    software through principled use of architectural constraints, thereby obtaining the functional, performance, and social 
    properties desired of an architecture.
    写作目的是想在符合架构原理的前提下，理解和评估以网络为基础的应用软件的架构设计，得到一个功能强、性能好、适宜通信的架构。
REST的通常被译成“表现层状态转化”，听起来比较生涩，要理解REST就要理解Representational State Transfer这个词组的每一个词代表了什么涵义  
- ### Resources
REST省略了主语表现层指的是“资源”表现层。所谓"资源"，就是网络上的一个实体，或者说是网络上的一个具体信息。它可以是一段文本、一张图片、一首歌曲、一种服务，总之就是一个具体的实在。你可以用一个URI（统一资源定位符）指向它，每种资源对应一个特定的URI。要获取这个资源，访问它的URI就可以，因此URI就成了每一个资源的地址或独一无二的识别符
- ### Representation
表现层指的是资源的表现形式，HTTP的[Content-Type](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/Content-Type)实体头部用于指示资源的MIME类型 [media type](https://developer.mozilla.org/en-US/docs/Web/HTTP/Basics_of_HTTP/MIME_types)，常用的type如下
    
    text/plain
    text/html
    image/jpeg
    image/png
    audio/mpeg
    audio/ogg
    audio/*
    video/mp4
    application/*
    application/json
    application/xml
    application/javascript
    application/octet-stream
- ### State 
- ### Transfer 

- ## RESTful


- ## 资源与URI

- ## HTTP动词

- ## 如何设计restful API

- ## restful设计误区
