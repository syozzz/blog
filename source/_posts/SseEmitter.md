---
title: 服务器端主动推送数据-SSE
author: syo
avatar: https://pic.imgdb.cn/item/5efaf31714195aa59486bf81.jpg
categories: 技术
comments: true
tags: 
 - java
 - spring
 - SseEmitter
date: 2020-7-24 17:10:00
keywords: SseEmitter
description: 服务器推送技术 SSE（Server-sent Events）基于 spring-boot 的实现 
photos: https://pic.imgdb.cn/item/5efb4ac414195aa594a4b87c.jpg
---

服务器主动向客户端推送数据，往往我们首先想到的都是 websocket 技术。但其实，HTML5 还提供了另一种方法：SSE（Server-Sent Events）即服务器推送事件。相比于 websocket 来说， SSE 更加的简单和轻量，对于某些类型的应用来说，SSE 可能会是更优的选择。

### SSE 的本质

>严格地说，HTTP 协议无法做到服务器主动推送信息。但是，有一种变通方法，就是服务器向客户端声明，接下来要发送的是流信息（streaming）。
>
>也就是说，发送的不是一次性的数据包，而是一个数据流，会连续不断地发送过来。这时，客户端不会关闭连接，会一直等着服务器发过来的新的数据流，视频播放就是这样的例子。本质上，这种通信就是以流信息的方式，完成一次用时很长的下载。
>
>SSE 就是利用这种机制，使用流信息向浏览器推送信息。它基于 HTTP 协议，目前除了 IE/Edge，其他浏览器都支持。
>
>--摘自[阮一峰的博客](http://www.ruanyifeng.com/blog/2017/05/server-sent_events.html)

### 与 websocket 的对比

|          | Websocket              | SSE                            |
| :------- | :--------------------- | :----------------------------- |
| 开发难度 | 使用较复杂，应用更臃肿 | 使用简单，应用更轻量，自动重连 |
| 数据流向 | 双向传递               | 单向，只能由服务器推送到客户端 |
| 传输性能 | 传输效率高             | 传输效率较低                   |

### 兼容性

![sse兼容性](https://pic.imgdb.cn/item/5f1a961c14195aa594e2e822.jpg)

### 基于 SpringBoot 的实现

#### 定义 SSE 接口：  

```java
@RestController
@RequestMapping("/sse")
public class SseController {
  
    @Autowired
    NotifyListener listener;

    @GetMapping("/register")
    public SseEmitter register(String taskId) {
        //4个小时超时 如果不设置 默认超时时间为30秒 设置为0永不过时
        final SseEmitter emitter = new SseEmitter(14400_000L);
        try {
            listener.addClient(taskId, emitter);
        } catch (Exception e) {
            emitter.completeWithError(e);
        }
        return emitter;
    }

}
```

创建一个 SseEmitter 对象并设置过期时间，然后 return 给客户端，这样便就创建好了一个 SSE 接口。通常会用一个处理类来保存所有创建的 SseEmitter 对象，以方便调用其 send 方法向客户端推送数据。

```java
@Component
@Slf4j
public class NotifyListener {

    private ListMultimap<String, SseEmitter> emitters;

    @PostConstruct
    public void init() {
        emitters = Multimaps.newListMultimap((new LinkedHashMap<String, Collection<SseEmitter>>()), new Supplier<List<SseEmitter>>() {
            @Override
            public List<SseEmitter> get() {
                return new ArrayList<SseEmitter>();
            }
        });
    }

    @EventListener
    @Async
    public void handler(NotifyEvent event) throws IOException {
        String taskId = event.getTaskId();
      	int Status = event.getStatus();
        List<SseEmitter> clients = emitters.get(clientId);
        if (status == TaskProgressUtil.TaskStatus.END.getStatus() ||
                status == TaskProgressUtil.TaskStatus.ERROR.getStatus()) {
            notifyClientAndClose(clients, status, taskId);
        } else {
            notifyClient(clients, status);
        }
    }

    private void notifyClient(List<SseEmitter> clients, int status) throws IOException {
        for (SseEmitter client : clients) {
            client.send(status);
        }
    }

    private void notifyClientAndClose(List<SseEmitter> clients, int status, String taskId) throws IOException {
        for (SseEmitter client : clients) {
            client.send(status);
            client.complete();
        }
        emitters.removeAll(taskId);
    }

    public void addClient(String taskId, SseEmitter emitter) {
        emitters.put(taskId, emitter);
    }

}
```

这样，当服务端需要向客户端推送数据时，只需要 publish 一个 NotifyEvent 事件即可。

#### 前端实现：

##### 1.创建请求

```javascript
const es = new EventSource(`/sse/register?taskId=${taskId}`);
```

##### 2.设置事件回调

*open* 事件，链接一旦创建，就会触发。

```javascript
es.onopen = function (event) {
  // ...
};

// 另一种写法
es.addEventListener('open', function (event) {
  // ...
}, false);
```

*message* 事件，客户端收到服务器发来的数据，就会触发。

```javascript
es.onmessage = function (event) {
  var data = event.data;
  // handle message
};

// 另一种写法
es.addEventListener('message', function (event) {
  var data = event.data;
  // handle message
}, false);
```

*error* 事件，如果发生通信错误，就会触发。

```javascript
es.onerror = function (event) {
  // handle error event
};

// 另一种写法
es.addEventListener('error', function (event) {
  // handle error event
}, false);
```

也可以通过 addEventListener 来自定义事件。

```javascript
es.addEventListener('myEvent', function (event) {
  var data = event.data;
  // handle message
}, false);
```

此时，后端发送数据就不能简单的 send 数据了，而是要 send 一个 SseEventBuilder 对象来包装数据。

```java
emitter.send(SseEmitter.event().name("myEvent").data(data));
```

当数据推送流程结束时，调用 close 方法关闭连接。

```javascript
es.close();
```

### 踩坑

如果是前端和后端联调的话，一定要记得 node 的代理不能直接代理 SSE 请求到后端接口，表现形式为请求会一直 pending，直到服务端 *complete()* 之后，代理服务器才会一口气把响应回传到前端。

### 参考链接

- MDN，[Using server-sent events](https://developer.mozilla.org/en-US/docs/Server-sent_events/Using_server-sent_events)
- 阮一峰的博客，[Server-Sent Events 教程](http://www.ruanyifeng.com/blog/2017/05/server-sent_events.html)

