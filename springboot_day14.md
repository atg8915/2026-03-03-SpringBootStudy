# 📘 Spring Boot Day 14 — WebSocket + STOMP 채팅 3종 실습 (SpringWebSocket_1/_2 + SpringPiniaProject_2)

## 0. 핵심 빠른 참조 — 프로젝트별 채팅 구현 비교

| 구분 | `SpringWebSocket_1` | `SpringWebSocket_2` | `SpringPiniaProject_2` |
|------|----------------------|------------------------|---------------------------|
| 채팅 범위 | 전체 공개 채팅만 | 1:1 개인 채팅만 | 전체 채팅 + 1:1 채팅 모두 |
| 화면 상태관리 | jQuery로 DOM 직접 조작 | Pinia(`chatStore.js`) | Pinia(`chatStore.js`, 방 개념 포함) |
| sender 결정 방식 | 클라이언트 입력값(`#sender`)을 그대로 신뢰 | 클라이언트 입력값(`userId`)을 그대로 신뢰 | `HttpSession`에서 로그인 ID를 서버가 직접 꺼내 세팅 |
| Broker 목적지 | `/topic`만 등록 | `/topic` + `/queue` + `/user` | `/topic` + `/queue` + `/user` |
| 접속 endpoint | `/ws-chat` | `/ws-chat` | `/chat-ws` |
| 화면 접근 제한 | 없음 | 없음 | `sec:authorize="isAuthenticated()"`로 로그인 시에만 메뉴 노출 |

---

## 1. WebSocket + STOMP 기본 구조

세 프로젝트 모두 `WebSocketConfig`의 기본 골격은 동일함.

```java
@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer{

	@Override
	public void registerStompEndpoints(StompEndpointRegistry registry) {
		registry.addEndpoint("/ws-chat")
				.setAllowedOriginPatterns("*")
				.withSockJS();
	}

	@Override
	public void configureMessageBroker(MessageBrokerRegistry registry) {
		registry.enableSimpleBroker("/topic","/queue");
		registry.setApplicationDestinationPrefixes("/app");
		registry.setUserDestinationPrefix("/user");
	}
}
```

- `registerStompEndpoints`: 클라이언트가 처음 접속할 URI 등록. `withSockJS()`로 WebSocket이 막힌 환경에서도 HTTP 기반으로 대체 접속 가능
- `configureMessageBroker`: 서버 → 클라이언트 방향은 `enableSimpleBroker`에 등록한 경로(`/topic`, `/queue`)로, 클라이언트 → 서버 방향은 `setApplicationDestinationPrefixes("/app")`로 구분
- `setUserDestinationPrefix("/user")`: 1:1(개인) 메시지 전용 접두사. 서버가 `convertAndSendToUser()`로 보내면 실제로는 `/user/{username}/...` 형태로 변환됨
- `SpringWebSocket_1`의 chat.html에 남아있는 개념 정리 주석:
  - STOMP: WebSocket 기반 메시지를 주고받기 위한 프로토콜(규칙)
  - WebSocket: 서버-클라이언트 간 실시간 양방향 통신 기술
  - SockJS: WebSocket이 안 될 때 HTTP 기반으로 대체하는 라이브러리
  - Kafka: 서버-서버 간 대규모 이벤트 메시지를 비동기로 전달하는 분산 메시징
  - Redis: 인메모리 데이터 저장소. 캐시/세션/실시간 데이터에 주로 사용

---

## 2. SpringWebSocket_1 — 전체 공개 채팅 (jQuery)

```java
@Controller
public class ChatController {
	@GetMapping("/chat")
	public String chat_page() {
		return "chat";
	}
	@MessageMapping("/chat.send")
	@SendTo("/topic/public")
	public ChatMessage sendMessage(ChatMessage message) {
		message.setTime(new SimpleDateFormat("yyyy-MM-dd hh:mm:ss").format(new Date()));
		return message;
	}
}
```

```javascript
function connection(){
	const socket=new SockJS('/ws-chat')
	stompClient=Stomp.over(socket)
	stompClient.connect({},function(){
		stompClient.subscribe("/topic/public", function(msg){
			const data=JSON.parse(msg.body)
			showMessage(data)
		})
	})
}
function sendMessage(){
	stompClient.send('/app/chat.send',{},JSON.stringify({
		sender:$('#sender').val(),
		message:$('#message').val()
	}))
}
```

- 요청 경로 매칭: `new SockJS('/ws-chat')` ↔ `registerStompEndpoints`의 `/ws-chat`, `stompClient.send('/app/chat.send')` ↔ `@MessageMapping("/chat.send")`, `@SendTo("/topic/public")` ↔ `stompClient.subscribe('/topic/public')`
- `ChatMessage` VO(`sender`/`message`/`time`)는 서버에서 `time`만 채워서 그대로 반환. 전송 시각을 클라이언트가 아니라 서버 기준으로 남기는 방식
- Pinia 같은 상태관리 없이 jQuery로 `#chat` 리스트에 `<li>`를 직접 append하는 가장 단순한 구조

---

## 3. SpringWebSocket_2 — 1:1 개인 채팅 (Pinia)

```java
@MessageMapping("/chat.private")
public void privateMessage(ChatMessage message) {
	messagingTemplate.convertAndSend(
		"/queue/private/"+message.getReceiver(),
		message
	);
}
```

```javascript
const useChatStore=defineStore('chat',{
	state:()=>({
		stompClient:null, userId:'', message:[], msg:'', receiver:''
	}),
	actions:{
		connect(){
			const socket=new SockJS('/ws-chat')
			this.stompClient=Stomp.over(socket)
			this.stompClient.connect({},()=>{
				this.stompClient.subscribe('/queue/private/'+this.userId,(msg)=>{
					this.message.push(JSON.parse(msg.body))
				})
			})
		},
		send(){
			this.stompClient.send('/app/chat.private',{},JSON.stringify({
				sender:this.userId, receiver:this.receiver, message:this.msg
			}))
		}
	}
})
```

- `convertAndSend("/queue/private/"+receiver, ...)`: `@SendTo`처럼 요청자에게 자동 반환하는 게 아니라, 지정한 목적지로 임의 전송하는 방식. 받는 사람이 `/queue/private/{자기 id}`를 구독해 둬야 메시지를 받음
- 본인이 보낸 메시지는 화면에 안 뜸(받는 사람 큐로만 보냄) — 실습 단계에서 발견된 한계로 보임
- `chat.html`은 Vue 3 + Pinia를 CDN으로 로드해서 `store.userId`/`store.receiver`/`store.msg`를 `v-model`로 입력폼과 바인딩

---

## 4. SpringPiniaProject_2 — 로그인 세션 연동 통합 채팅

```java
@MessageMapping("/chat/public")
@SendTo("/topic/chat")
public ChatMessage publicChat(ChatMessage msg, HttpSession session) {
	msg.setSender((String)session.getAttribute("userid"));
	return msg;
}

@MessageMapping("/chat/private")
public void privateChat(ChatMessage msg, HttpSession session) {
	String sender=(String)session.getAttribute("userid");
	msg.setSender(sender);
	template.convertAndSendToUser(msg.getReceiver(), "/queue/chat", msg);
	template.convertAndSendToUser(sender, "/queue/chat", msg);
}
```

- `WebSocket_1`/`_2`는 클라이언트가 입력한 `sender` 값을 그대로 믿었지만 여기서는 `HttpSession`의 `userid`로 서버가 강제로 덮어씀. 클라이언트가 다른 사람 이름으로 위장해서 보낼 수 없게 막는 구조
- `privateChat`은 받는 사람뿐 아니라 보낸 사람 본인에게도 같은 메시지를 한 번 더 전송함. 그래야 본인 화면에도 방금 보낸 메시지가 즉시 표시됨(`WebSocket_2`에는 없던 처리)
- `header.html`에 `<li sec:authorize="isAuthenticated()"><a href="/chat">실시간 채팅</a></li>` 추가 — 로그인하지 않은 사용자에게는 채팅 메뉴 자체가 안 보임

```javascript
makeRoomId(user1,user2){
	return [user1,user2].sort().join('_')
},
changeroom(user){
	if(user=='public'){
		this.currentRoom='public'
		this.messages=this.publicMessages
	} else {
		const roomId=this.makeRoomId(this.loginUser,user)
		if(!this.privateMessages[roomId]){
			this.privateMessages[roomId]=[]
		}
		this.messages=this.privateMessages[roomId]
	}
}
```

- `makeRoomId`: 두 사용자 ID를 정렬한 뒤 합쳐서 방 ID를 만듦. `kim`+`hong`이든 `hong`+`kim`이든 항상 같은 방 ID(`hong_kim`)가 나오게 하는 목적
- `publicMessages`/`privateMessages`를 분리해서 들고 있다가 `currentRoom`에 따라 화면에 뿌릴 `messages`만 교체하는 구조
- `connect()` 액션 안에서 `/topic/users` 구독 콜백이 `this.users=...`로 대입하는데, `state()`에는 `user:[]`(단수)만 선언돼 있고 `users`(복수)는 선언이 안 된 상태 — 오타로 보이는 상태 불일치. 접속자 목록이 실제로 화면에 반영되는지 다음 실습에서 확인 필요

---

## 5. 다시 만들 때 체크리스트

```text
[WebSocketConfig 공통]
① @EnableWebSocketMessageBroker + WebSocketMessageBrokerConfigurer 구현
② registerStompEndpoints: addEndpoint(URI).setAllowedOriginPatterns("*").withSockJS()
③ configureMessageBroker: enableSimpleBroker("/topic","/queue") + setApplicationDestinationPrefixes("/app")
④ 1:1 채팅이 필요하면 setUserDestinationPrefix("/user") 추가 등록

[전체 공개 채팅]
⑤ @MessageMapping(수신 경로) + @SendTo(브로드캐스트 경로) 조합으로 구현
⑥ 클라이언트: stompClient.send(앱 경로) → 서버 처리 → stompClient.subscribe(브로드캐스트 경로)

[1:1 개인 채팅]
⑦ 서버: messagingTemplate.convertAndSend("/queue/private/"+받는사람ID, msg) 또는 convertAndSendToUser 사용
⑧ 클라이언트: 본인 큐(/queue/private/{내 id})를 반드시 구독해둬야 메시지를 받음
⑨ 보낸 사람 본인 화면에도 표시하려면 받는 사람뿐 아니라 본인에게도 한 번 더 전송할 것

[보안]
⑩ sender/작성자 정보는 클라이언트 입력값을 믿지 말고 HttpSession(로그인 세션)에서 서버가 직접 세팅
⑪ 로그인 필요한 메뉴는 sec:authorize="isAuthenticated()"로 화면에서부터 노출 제한

[상태관리]
⑫ Pinia state에서 실제 쓰는 변수명과 액션 안에서 대입하는 변수명이 일치하는지 확인(단수/복수 오타 주의)
⑬ 여러 채팅방을 다뤄야 하면 두 사용자 ID를 정렬 후 join해서 고유 방 ID를 만드는 방식 재사용 가능
```
