---
layout: post
title: websocket 
category: web
comments: false
--- 

# 一、WebSocket
WebSocket 是 HTML5 开始提供的一种在单个 TCP 连接上进行全双工通讯的协议。

WebSocket 使得客户端和服务器之间的数据交换变得更加简单，允许服务端主动向客户端推送数据。在 WebSocket API 中，浏览器和服务器只需要完成一次握手，两者之间就直接可以创建持久性的连接，并进行双向数据传输。

在 WebSocket API 中，浏览器和服务器只需要做一个握手的动作，然后，浏览器和服务器之间就形成了一条快速通道。两者之间就直接可以数据互相传送。

现在，很多网站为了实现推送技术，所用的技术都是 Ajax 轮询。轮询是在特定的的时间间隔（如每1秒），由浏览器对服务器发出HTTP请求，然后由服务器返回最新的数据给客户端的浏览器。这种传统的模式带来很明显的缺点，即浏览器需要不断的向服务器发出请求，然而HTTP请求可能包含较长的头部，其中真正有效的数据可能只是很小的一部分，显然这样会浪费很多的带宽等资源。

HTML5 定义的 WebSocket 协议，能更好的节省服务器资源和带宽，并且能够更实时地进行通讯。

# 二、使用姿势（后端）
以下是 WebSocket 对象的相关事件。

|事件|事件处理程序|描述
|--|--|
|open    |Socket.onopen   |连接建立时触发
|message |Socket.onmessage |   客户端接收服务端数据时触发
|error   |Socket.onerror  |通信发生错误时触发
|close   |Socket.onclose  |连接关闭时触发

下面是后端实现的代码实例，用 @ServerEndpoint来指明访问地址，可以加上一些handler来处理消息。

configurator可以去除掉。本文会用它来获取连接的IP（见下文）。

@OnOpen是建立socket触发的事件；其他 @On*都对应上述表格中的事件。只有4种。

```
@Component
@ServerEndpoint(value = "/ws/v1/exam", configurator = EndpointConfigurator.class)
public class WebSocketHandler {

    private static MessageHandlerService messageHandlerService;

    private static SocketControlService socketControlService;

    @Autowired
    public void setWebSocketHandler(MessageHandlerService messageHandlerService) {
        WebSocketHandler.messageHandlerService = messageHandlerService;
    }

    @Autowired
    public void setSocketControlService(SocketControlService socketControlService) {
        WebSocketHandler.socketControlService = socketControlService;
    }

    @OnOpen
    public void onOpen(Session session, EndpointConfig config) {
        log.info("[{}] open ws", session.getId());
        socketControlService.handleConnect(session);
    }

    @OnClose
    public void onClose(Session session, CloseReason reason) {
        log.info("[{}] close ws: {}", session.getId(), reason.toString());
        messageHandlerService.handleClose(session, reason.toString());
        // socketControlService.handleDisConnect(session);
    }

    @OnError
    public void onError(Session session, Throwable error) {
        // NOTE: Don't print any error exception.
        // As known, nginx will close the web socket connection after long time inactive, with causing EOFException
        // null. Just ignore those exceptions.
        log.warn("[{}] error in ws connection", session.getId());
        messageHandlerService.handleClose(session, error.getMessage());
        // socketControlService.handleDisConnect(session);
    }

    /**
     * 接收string
     */
    @OnMessage
    public void onMessage(Session session, String messages) {
        log.debug("[{}] receive String: {}", session.getId(), cutStr(messages));
        beginTime.set(System.currentTimeMillis());
        log.info("[begin] {} ", session.getId());

        messageHandlerService.handle(session, messages);

        long timeUsed = System.currentTimeMillis() - beginTime.get();
        log.info("[sessionId:{}, time:{}ms]", session.getId(), timeUsed);

        // 发送消息
        session.getBasicRemote().sendText(String.valueOf(timeUsed));
    }

    /**
     * 接收byte[]
     */
    @OnMessage
    public void onMessage(Session session, byte[] messages) {
        // empty
    }
}
```

配置参数
```
@Setter
@Configuration
@ConfigurationProperties(prefix = "webSocket")
public class WebSocketConfig {

    private int maxTextMessageBufferSize = 10 * 1024 * 1024;
    private int maxBinaryMessageBufferSize = 10 * 1024 * 1024;

    @Bean
    public ServerEndpointExporter serverEndpointExporter() {
        return new ServerEndpointExporter();
    }

    @Bean
    public ServletServerContainerFactoryBean createWebSocketContainer() {
        ServletServerContainerFactoryBean container = new ServletServerContainerFactoryBean();
        container.setMaxTextMessageBufferSize(maxTextMessageBufferSize);
        container.setMaxBinaryMessageBufferSize(maxBinaryMessageBufferSize);
        return container;
    }
}
```
WebSocket接收的消息有两种，一种是Text,一种是Binary，分别对应上述的两种messages参数类型。可以只用一种。

发送消息使用 `session.getBasicRemote().sendText()`或`session.getBasicRemote().sendBinary()`,也是可以发送Text或二进制内容。

连接websocket使用类似于 `ws://172.18.17.151:8085/ws/v1/exam` 地址。

在Chrome里可以使用 Simple WebSocket Client插件来调试。

# 三、踩过的坑和Best practice

建议先看一遍tomcat对websocket的使用介绍，就短短一页内容：http://tomcat.apache.org/tomcat-8.0-doc/web-socket-howto.html

## 3.1 Socket连接调优

在socket连接过程中，留给程序员可配置的内存大小，主要就两个参数：maxTextMessageBufferSize、maxBinaryMessageBufferSize，
分别对应字符类消息和字节类消息的BufferSize，单位是byte，默认都是8192bytes。

在scoket建立时，会预分配maxTextMessageBufferSize+maxBinaryMessageBufferSize的空间。所以，可以调小这两个数值来增加服务的websocket连接数。如果只用到其中一种消息类型（如Text），可以将另一种设置成0或很小的值。

但是，如果消息大小超过了这个bufferSize，会导致ws断开。所以得根据实际业务情况来估值。

不过，对于那种消息体很大或流式消息的情况，可以使用`MessageHandler.Partial `机制，来实现将一个消息拆分成多个call传输。鉴于本次服务没有这种场景，未能继续深究。

除了WebSocket本身的调优，还可以调节JVM的 -Xmx 参数，加大服务的最大运存；还可以及时断开Socket释放资源。

## 3.2 从Socket中获取IP

根据服务的部署情况，可以通过下面方式来获取Socket中的IP。

基于IP可以添加诸如限制黑白名单、流量限制、地域统计等功能。

### 3.2.1 直连服务

在客户和服务之间没有中转服务(如正向代理、loadBalance等)时，可以采用很简单的方式获取到IP。

```
/**
 * 针对webSocket相关的封装
 */
@Slf4j
public class WebSocketUtil {

    /**
     * 向客户端发送消息
     */
    public static void sendMsgToClient(Session session, String message) {
        if (log.isDebugEnabled()) {
            log.debug("[{}] send message: {}", session.getId(), message);
        }
        try {
            session.getBasicRemote().sendText(message);
        } catch (Exception e) {
            log.error("[{}] fail to send message", session.getId(), e);
        }
    }

    public static void sendMsgToClient(Session session, Object object) {
        if (log.isDebugEnabled()) {
            log.debug("[{}] send message: {}", session.getId(), object);
        }
        try {
            session.getBasicRemote().sendText(JsonUtil.toJsonString(object));
        } catch (Exception e) {
            log.error("[{}] fail to send message", session.getId(), e);
        }
    }

    public static InetSocketAddress getRemoteAddress(Session session) {
        if (session == null) {
            return null;
        }
        Async async = session.getAsyncRemote();

        // 在Tomcat 8.0.x版本有效
        // InetSocketAddress socketAddress =
        // (InetSocketAddress) getFieldInstance(async,"base#sos#socketWrapper#socket#sc#remoteAddress");
        // 在Tomcat 8.5以上版本有效
        return (InetSocketAddress) getFieldInstance(async, "base#socketWrapper#socket#sc#remoteAddress");
    }

    private static Object getFieldInstance(Object obj, String fieldPath) {
        String[] fields = fieldPath.split("#");
        for (String field : fields) {
            obj = getField(obj, obj.getClass(), field);
            if (obj == null) {
                return null;
            }
        }
        return obj;
    }

    private static Object getField(Object obj, Class<?> clazz, String fieldName) {
        for (; clazz != Object.class; clazz = clazz.getSuperclass()) {
            try {
                Field field;
                field = clazz.getDeclaredField(fieldName);
                field.setAccessible(true);
                return field.get(obj);
            } catch (Exception e) {
                // just ignore
                // log.debug("Failed to get field");
            }
        }

        return null;
    }

}
```

业务端调用代码：
```
    public void handleConnect(Session session) {
        try {
            String address = getAddress(session);
            log.debug("socket ip connected:{}", StringUtil.isBlank(address) ? UNKNOWN : address);
            session.getUserProperties().put(Constants.SESSION_IP, address);
        } catch (Exception ex) {
            // won't throw any exception.
            log.warn("handle connect error, will go on", ex.getMessage());
        }
    }

    private String getAddress(Session session) {
        InetSocketAddress remoteAddress = WebSocketUtil.getRemoteAddress(session);
        if (remoteAddress != null) {
            return remoteAddress.getAddress().getHostAddress();
        } else {
            return null;
        }
    }
```

原理就是基于Java反射获取Session中的InetSocketAddress，然后读取到客户端的IP地址。再将其存储在session.getUserProperties()里，供后续业务使用。

### 3.2.2 中转连接服务

在微服务模式和分布式的时代下，客户直连服务的场景基本不复存在。那么如何在中转服务后获取到客户真实的IP呢？

首先，得保证中转服务（如loadBalaner）能透传客户的IP，例如加到请求Header的某个字段里。不然后端也是巧妇难为无米之炊。

接着，得想办法在Session里获取到这个字段。WebSocket是基于TCP协议的，但是没看到和HTTP一样的header，在OnStart事件时拿到的Session，里面包含的信息也很有限，很有可能hSession就不包含header！

那得在前一步拿，就是在TCP的HandShake的时候，将拿到的IP值，作为用户属性传给Session。

第一步，在握手的时候捕获来自中转服务的字段：
```
import javax.websocket.HandshakeResponse;
import javax.websocket.server.HandshakeRequest;
import javax.websocket.server.ServerEndpointConfig;

@Slf4j
public class EndpointConfigurator extends ServerEndpointConfig.Configurator {

    private static final String IP_HEADER = "X-Original-Forwarded-For";

    @Override
    public void modifyHandshake(ServerEndpointConfig config, HandshakeRequest handshakeRequest,
            HandshakeResponse response) {
        for (Map.Entry<String, List<String>> e : handshakeRequest.getHeaders().entrySet()) {
            // log.debug(JsonUtil.toJsonString(e));
            if (IP_HEADER.equalsIgnoreCase(e.getKey())) {
                config.getUserProperties().put("client-ip", e.getValue().get(0));
            }
        }
    }
}

```
第二步，在WebSocketHandler加上这个配置：
```
@Slf4j
@Component
@ServerEndpoint(value = "/ws/v1/exam", configurator = EndpointConfigurator.class)
public class WebSocketHandler {
    ...
}
```

第三步，读取信息
```
    @OnMessage
    public void onMessage(Session session, String messages) {
        String ip = session.getUserProperties().get("client-ip");
        ...
    }
```

以上实现了从WebSocket获取IP的过程。  

# REF
> [WebSocket How-To](http://tomcat.apache.org/tomcat-8.0-doc/web-socket-howto.html)