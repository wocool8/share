# REST(基于HTTP实现介绍) 
---
## URI(Uniform Resource Identifier)
URI是资源的唯一标识，根据URI对资源进行绑定，URI对应的是某个特定资源，所以在设计的URI中不能包含动词，

    /mumu/books/bookName
## HTTP Verbs
[HTTP动词](https://developer.mozilla.org/en-US/docs/Web/HTTP/Methods)有很多，只简单介绍以下四种
- ### GET
The HEAD method asks for a response identical to that of a GET request, but without the response body.
- ### POST
The POST method is used to submit an entity to the specified resource, often causing a change in state or side effects on the server.
- ### PUT
The PUT method replaces all current representations of the target resource with the request payload.
- ### PATCH
The PATCH method is used to apply partial modifications to a resource.
- ### DELETE
The DELETE method deletes the specified resource
## REST
REST全称是Representational State Transfer，REST是[Roy Thomas Fielding](https://en.wikipedia.org/wiki/Roy_Fielding)在他2000年的博士论文[Architectural Styles and the Design of Network-based Software Architectures](https://www.ics.uci.edu/~fielding/pubs/dissertation/top.htm)中提出的。他的设计目的如下
    
    My work is motivated by the desire to understand and evaluate the architectural design of network-based application 
    software through principled use of architectural constraints, thereby obtaining the functional, performance, and 
    social properties desired of an architecture.
    写作目的是想在符合架构原理的前提下，理解和评估以网络为基础的应用软件的架构设计，得到一个功能强、性能好、适宜通信的架构。
REST的通常被译成“表现层状态转化”，听起来比较生涩，要理解REST就要理解Representational State Transfer这个词组的每一个词代表了什么涵义  
- ### Resources(资源)
REST省略了主语表现层指的是“资源”表现层。所谓"资源"，就是网络上的一个实体，或者说是网络上的一个具体信息。它可以是一段文本、一张图片、一首歌曲、一种服务，是一个具体的存在形式。可以用一个URI指向它，每种资源对应一个特定的URI。要获取这个资源，访问它的URI就可以，因此URI就成了每一个资源的地址或独一无二的识别符
- ### Representation(表现层)
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
- ### State Transfer(状态转化)
就是客户端和服务器互动的一个过程，由于HTTP是无状态的，资源状态是维护在服务端的，在互动过程中涉及到数据和状态的变化, 这种变化叫做状态转换。
资源是唯一的，对资源的状态改变使用的HTTP动词的对应语义实现对资源数据的增(PUT)删(DELETE)改(PUT/PATCH)查(GET)
## RESTful
REST是一种软件架构风格，RESTful是遵循REST架构风格的(一种实现)
## 如何设计restful API
- ### 通讯协议使用HTTPs
- ### 合理设计uri


    GET  https://example.com/appName/getBooks 获取所有书
    POST https://example.com/appName/addBooks 添加一本书
    POST https://example.com/appName/updateBooks/:bookId 修改一本书
    POST https://example.com/appName/deleteBooks/:bookId 删除一本书

在Web系统中通常会看到以上URI，在RESTful的URI中不可以包含动词，修改后URI如下

    https://example.com/api/books 获取所有书
    https://example.com/api/books 添加一本书
    https://example.com/api/books/bookId 修改一本书
    https://example.com/api/books/bookId 删除一本书    
    
- ### 合理使用http动词


    GET（SELECT）：从服务器取出资源（一项或多项）
    POST（CREATE）：在服务器新建一个资源
    PUT（UPDATE）：在服务器更新资源（客户端提供改变后的完整资源）
    PATCH（UPDATE）：在服务器更新资源（客户端提供改变的属性）
    DELETE（DELETE）：从服务器删除资源
    
- ### 向客户端返回状态码和提示信息


    200 OK ：服务器成功返回用户请求的数据，操作是幂等的
    201 CREATED ：用户新建或修改数据成功。
    204 NO CONTENT ：用户删除数据成功。
    400 INVALID REQUEST ：用户发出的请求有错误，服务器没有进行新建或修改数据的操作，该操作是幂等的
    401 Unauthorized ：表示用户没有权限（令牌、用户名、密码错误）
    403 Forbidden ： 表示用户得到授权（与401错误相对），但是访问是被禁止的
    404 NOT FOUND ：用户发出的请求针对的是不存在的记录，服务器没有进行操作，该操作是幂等的

其他[状态码](https://www.restapitutorial.com/httpstatuscodes.html)
    
   
    






## restful设计误区

## restful设计优点

## 开源框架对REST的支持
- ### SpringMvc
- ### Jersey
- ### Play
