# 📘 Spring Boot Day 15 — STOMP 채팅에 Spring Security 연동 (SpringPiniaProject_2)

## 0. 핵심 빠른 참조 — Day14 대비 오늘 바뀐 점

| 구분 | Day14 (HttpSession 기반) | Day15 (Principal 기반) |
|------|---------------------------|--------------------------|
| 로그인 사용자 확인 | `@MessageMapping` 메서드에 `HttpSession` 주입 → `session.getAttribute("userid")` | `@MessageMapping` 메서드에 `Principal` 주입 → `p.getName()` |
| 화면 진입 경로 | `/chat` | `/chat/chat` |
| Thymeleaf 로그인 ID 추출 | `session.userid` | `#authentication.name` |
| 접속자 목록 | 없음 | `/chat/join` 메시지 → 서버가 `Set<String>`에 누적 후 `/topic/users`로 브로드캐스트 |
| 방(room) 상태 관리 | `currentRoom = 'public'` 고정 문자열 | 1:1일 때 `currentRoom`에 실제 `roomId`(`user1_user2`)를 저장 |
| 메시지 전송 | 화면에 send 버튼만 있고 실제 전송 액션 미구현 | `sendPublic` / `sendPrivate` / `send()` 액션으로 실제 전송 완성 |

---

## 1. WebSocket에서 Principal로 로그인 사용자 확인하기

STOMP 메시지 핸들러(`@MessageMapping`)는 HTTP 요청이 아니라서 `HttpSession`을 직접 주입받을 수 없음. 대신 Spring Security가 WebSocket 연결에도 인증 정보를 실어주기 때문에, 파라미터로 `Principal`을 받으면 로그인한 사용자 이름을 바로 꺼낼 수 있음.

```java
@MessageMapping("/chat/public")
@SendTo("/topic/chat")
public ChatMessage publicChat(ChatMessage msg, Principal p) {
    // Spring Security 인증 정보 => Principal로 전달됨
    msg.setSender(p.getName());
    return msg;
}

@MessageMapping("/chat/private")
public void privateChat(ChatMessage msg, Principal p) {
    String sender = p.getName();
    msg.setSender(sender);

    template.convertAndSendToUser(msg.getReceiver(), "/queue/chat", msg);
    template.convertAndSendToUser(sender, "/queue/chat", msg);
}
```

- 클라이언트가 보낸 `sender` 값을 그대로 믿지 않고, 서버가 `Principal`에서 직접 꺼낸 값으로 덮어씀
- Thymeleaf에서는 세션 속성 대신 Security 인증 객체로 로그인 ID를 꺼냄

```html
<script th:inline="javascript">
const LOGIN_USER = /*[[${#authentication.name}]]*/ '';
</script>
```

- 화면 이동 경로도 `/chat` → `/chat/chat`으로 바뀜(`GetMapping` 메서드명도 `chat_page`로 변경, 반환 뷰도 `chat/chat`)
- `header.html`의 메뉴 링크도 같이 `/chat/chat`으로 수정

---

## 2. 접속자 목록 브로드캐스트 (`/chat/join`)

```java
private final Set<String> onlineUsers = ConcurrentHashMap.newKeySet();

@MessageMapping("/chat/join")
public void join(Principal p) {
    String username = p.getName();
    onlineUsers.add(username);
    template.convertAndSend("/topic/users", onlineUsers);
}
```

- 여러 명이 동시에 접속/해제할 수 있으므로 동시성 안전한 `ConcurrentHashMap.newKeySet()` 사용
- 사용자가 접속하면 전체 목록을 `/topic/users`로 다시 뿌려서 모든 클라이언트가 최신 접속자 목록을 받음

> 주의: `join()`을 호출하는 로직이 클라이언트(`chatStore.js`)에 아직 없고, 접속 종료 시 `onlineUsers`에서 제거하는 로직도 없음. `SessionConnectedEvent` / `SessionDisconnectEvent` import는 추가됐지만 `@EventListener`로 실제 연결되지 않은 상태라 다음 작업으로 이어질 미완성 구간임.

---

## 3. Pinia `chatStore.js` — 방 상태 관리 방식 변경

Day14까지는 `currentRoom`이 `'public'` 아니면 무조건 임의 문자열이었는데, 오늘은 1:1 채팅방의 `currentRoom` 자체를 `roomId`로 저장하도록 바뀜.

```js
changeRoom(user) {
    if (user === 'public') {
        this.currentRoom = 'public'
        this.messages = this.publicMessages
    } else {
        const roomId = this.makeRoomId(this.loginUser, user)
        this.currentRoom = roomId          // roomId를 그대로 상태로 저장
        if (!this.privateMessages[roomId]) {
            this.privateMessages[roomId] = []
        }
        this.messages = this.privateMessages[roomId]
    }
    this.scrollToBottom()
}
```

- `currentRoom`이 `roomId`이기 때문에, 메시지 수신 시에도 `makeRoomId(m.sender, m.receiver)`로 같은 방인지 바로 비교 가능
- 상대방 이름을 화면에 표시할 때 쓰는 `getOtherUser(roomId)` 액션 추가

```js
getOtherUser(roomId) {
    if (roomId === 'public') return ''
    const users = roomId.split('_')
    return users[0] === this.loginUser ? users[1] : users[0]
}
```

### 메시지 전송 액션 완성

```js
sendPublic(message) {
    this.stomp.send('/app/chat/public', {}, JSON.stringify({ message }))
},
sendPrivate(to, message) {
    this.stomp.send('/app/chat/private', {}, JSON.stringify({ receiver: to, message }))
},
send() {
    if (!this.msg.trim()) return
    if (this.currentRoom === 'public') {
        this.sendPublic(this.msg)
    } else {
        const users = this.currentRoom.split('_')
        const receiver = users[0] === this.loginUser ? users[1] : users[0]
        this.sendPrivate(receiver, this.msg)
    }
    this.msg = ''
}
```

- `connect()`에서 `/topic/users`, `/topic/chat`, `/user/queue/chat` 세 채널을 구독
- `/user/queue/force-disconnect` 구독도 추가됐는데, 이건 중복 로그인 시 강제 로그아웃을 위한 채널로 보이지만 서버 쪽에서 이 큐로 메시지를 보내는 코드는 아직 없어서 클라이언트만 미리 구독 준비해둔 상태임

---

## 4. 화면(`chat.html`) — 메시지/접속자 목록 실제 렌더링

Day14까지는 화면 골격만 있고 실제 데이터 바인딩이 비어있었는데, 오늘 `v-for`/`v-if`로 실데이터를 그리도록 채워짐.

```html
<!-- 접속자 목록 -->
<div v-for="u in store.users" :key="u"
     class="list-group-item" @click="store.changeRoom(u)">
    {{ u }}
</div>

<!-- 채팅방 헤더 -->
<span v-if="store.currentRoom === 'public'">전체 채팅</span>
<span v-else>1:1 채팅 - {{ store.getOtherUser(store.currentRoom) }}</span>

<!-- 메시지 목록 -->
<div class="message-row" v-for="(m, index) in store.messages" :key="index">
    <div v-if="m.sender !== store.loginUser" class="bubble left">
        <div v-if="store.currentRoom !== 'public'" class="sender">{{ m.sender }}</div>
        {{ m.message }}
    </div>
    <div v-else class="bubble right">{{ m.message }}</div>
</div>

<!-- 입력창 -->
<input v-model="store.msg" @keyup.enter="store.send()" placeholder="메시지 입력">
<button @click="store.send()">전송</button>
```

- 본인 메시지(`m.sender === loginUser`)는 오른쪽 말풍선, 상대방 메시지는 왼쪽 말풍선으로 분기
- 1:1 채팅일 때만 상대방 이름(`sender`)을 말풍선 위에 표시(전체 채팅에서는 생략)
- `onMounted()`에서 `store.connect`(함수 참조만 저장)로 되어 있던 Day14 버그가 `store.connect()`(실제 호출)로 수정됨

---

## 5. 새로 만들어진 빈 스캐폴딩 (`Board` 관련)

`BoardMapper` / `BoardService` / `BoardServiceImpl` / `BoardVO` / `BoardRestController` 파일이 새로 생성됐지만 내부 구현은 없는 빈 껍데기 상태임. 게시판 기능을 다음에 이어서 만들기 위한 준비 단계로 보임.

---

## 6. 다시 만들 때 체크리스트

```text
[STOMP + Security 연동]
① @MessageMapping 메서드 파라미터에 HttpSession 대신 Principal을 받는다
② p.getName()으로 로그인 사용자 ID를 꺼내 sender를 서버가 직접 세팅한다(클라이언트 값 신뢰 X)
③ Thymeleaf에서 로그인 ID는 #authentication.name으로 꺼낸다

[접속자 목록]
④ Set<String>(ConcurrentHashMap.newKeySet())으로 동시 접속자 목록 보관
⑤ /chat/join 메시지를 받으면 목록에 추가하고 /topic/users로 전체 브로드캐스트
⑥ (다음 작업) 접속 종료 시 목록에서 제거하는 로직은 SessionDisconnectEvent 리스너로 보완 필요

[Pinia 방 상태 관리]
⑦ 1:1 방은 currentRoom에 roomId(정렬된 두 아이디를 _로 join)를 그대로 저장
⑧ makeRoomId(sender, receiver)로 메시지 수신 시에도 동일 방 여부를 판별
⑨ getOtherUser(roomId)로 화면에 표시할 상대방 이름을 room에서 역산

[메시지 송수신 완성]
⑩ sendPublic/sendPrivate로 destination을 분리하고, send()에서 currentRoom 기준으로 분기
⑪ connect()는 반드시 함수 호출(store.connect())로 실행 — 참조만 저장하면 연결 안 됨
```
