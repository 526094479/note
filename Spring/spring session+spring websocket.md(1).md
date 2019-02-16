# 什么是WebSocket
* Websocket是基于HTTP协议的，或者说借用了HTTP的协议来完成一部分握手，在Http 请求行中附带Upgrade 等参数告诉服务器这是websocket 协议。收到回复后，接下来就是websocket 协议的通讯了。
```
GET /chat HTTP/1.1
Host: server.example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: x3JJHMbDL1EzLkh9GBhXDw==
Sec-WebSocket-Protocol: chat, superchat
Sec-WebSocket-Version: 13
Origin: http://example.com


HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: HSmrc0sMlYUkAGmm5OPpG2HaGWk=
Sec-WebSocket-Protocol: chat
```
* WebSocket 中无法获取到HttpSession 。

# 为什么要集成
* 一个基于WebSocket 聊天的页面，如果用户一直聊天，由于一直没有http 请求更新Httpsession ，会导致httpsession 失效。
* httpsession 失效的时候，由HttpSession 创建的WebSocket 会被强制关闭。

# 如何集成
* 在Spring Session支持中，我们只需要改变两件事情
  * 我们继承AbstractSessionWebSocketMessageBrokerConfigurer代替AbstractWebSocketMessageBrokerConfigurer 。
  * 重命名registerStompEndpoints方法为configureStompEndpoints
  
* AbstractSessionWebSocketMessageBrokerConfigurer在幕后做了什么：
  * WebSocketConnectHandlerDecoratorFactory作为WebSocketHandlerDecoratorFactory被添加到WebSocketTransportRegistration。这就确保了包含WebSocketSession的SessionConnectEvent可以被自动启动。Spring Session终止之后，WebSocketSession需要去终止所有开启的WebSocket链接。
  * SessionRepositoryMessageInterceptor作为HandshakeInterceptor被添加到StompWebSocketEndpointRegistration中。这就确保了会话已经被添加到WebSocket的属性中，以便更新上一次访问的时间。
  * SessionRepositoryMessageInterceptor作为ChannelInterceptor被添加到我们的入站ChannelRegistration中。这就确保每次入站消息都会被接收，Spring Session的最后一次访问时间也会被更新。
  * WebSocketRegistryListener作为一个Spring Bean被创建。这就确保了我们的所有Session id都会被映射到一个关联的WebSocket链接。通过维护这个映射，当Spring Session终止的时候我们就可以关闭所有的WebSocket链接。
  
## SessionRepositoryMessageInterceptor
* 可以看到 SessionRepositoryMessageInterceptor 实现了ChannelInterceptor，HandshakeInterceptor，重写了preSend 和 beforeHandshake 方法
* beforeHandshake 方法在websocket 内的 attributes 中添加了sessionId。（key 为SPRING.SESSION.ID，value 为sessionId）
* preSend 在每次发送信息前，获取message 的header 中的sessionId，再从spring管理的SessionRepository 中获取sessionId 对应的session，更新lastAccessedTime()
 
```
public final class SessionRepositoryMessageInterceptor<S extends Session>
		implements ChannelInterceptor, HandshakeInterceptor {
		
		private static final String SPRING_SESSION_ID_ATTR_NAME = "SPRING.SESSION.ID";
		
		private final SessionRepository<S> sessionRepository;

		
		@Override
	public Message<?> preSend(Message<?> message, MessageChannel channel) {
		if (message == null) {
			return message;
		}
		SimpMessageType messageType = SimpMessageHeaderAccessor
				.getMessageType(message.getHeaders());
		if (!this.matchingMessageTypes.contains(messageType)) {
			return message;
		}
		Map<String, Object> sessionHeaders = SimpMessageHeaderAccessor
				.getSessionAttributes(message.getHeaders());//通过Message的Header 获取到该WebSocket 中带有的SessionId。 用该SessionId 去sessionRepository 查找并更新该SessionId 的LastAccessedTime
		String sessionId = (sessionHeaders != null)
				? (String) sessionHeaders.get(SPRING_SESSION_ID_ATTR_NAME)
				: null;
		if (sessionId != null) {
			S session = this.sessionRepository.findById(sessionId);
			if (session != null) {
				// update the last accessed time
				session.setLastAccessedTime(Instant.now());
				this.sessionRepository.save(session);
			}
		}
		return message;
	}

	@Override
	public boolean beforeHandshake(ServerHttpRequest request, ServerHttpResponse response,
			WebSocketHandler wsHandler, Map<String, Object> attributes) throws Exception {
		if (request instanceof ServletServerHttpRequest) {
			ServletServerHttpRequest servletRequest = (ServletServerHttpRequest) request;
			HttpSession session = servletRequest.getServletRequest().getSession(false);
			if (session != null) {
				setSessionId(attributes, session.getId());//此处，调用下面的方法，往map 中添加多了一个属性，key 为 "SPRING.SESSION.ID", value 为"sessionId"
			}
		}
		return true;
	}
	@Override
	public void afterHandshake(ServerHttpRequest request, ServerHttpResponse response,
			WebSocketHandler wsHandler, Exception exception) {
	}

	public static String getSessionId(Map<String, Object> attributes) {
		return (String) attributes.get(SPRING_SESSION_ID_ATTR_NAME);
	}

	public static void setSessionId(Map<String, Object> attributes, String sessionId) {
		attributes.put(SPRING_SESSION_ID_ATTR_NAME, sessionId);
	}
}
```


