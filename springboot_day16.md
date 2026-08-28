# 📘 Spring Boot Day 16 — 게시판(Board) JPA 구현 + 대댓글/실시간 알림 + Jenkins 파이프라인 (SpringPiniaProject_2)

## 0. 핵심 빠른 참조 — Day15 대비 오늘 바뀐 점

| 구분 | Day15 (빈 스캐폴딩) | Day16 (실제 구현) |
|------|----------------------|----------------------|
| 게시판 데이터 접근 | `BoardMapper`/`BoardService`/`BoardServiceImpl`/`BoardVO` 파일만 생성, 내부 구현 없음 | JPA로 실제 구현. `BootBoard`(Entity) + `BootBoardRepository`(JpaRepository) + `BoardServiceImpl` |
| 게시판 화면 | 없음 | `board/list.html`(목록+페이징), `board/insert.html`(등록), `board/detail.html`(상세+댓글) 신규 |
| 댓글 데이터 접근 | 없음 | MyBatis로 별도 구현. `BoardCommentMapper`(XML) + `BoardCommentRestController` |
| 전역 설정 | `@SpringBootApplication`만 존재 | `@EnableScheduling`, `@EnableAspectJAutoProxy` 추가 |
| WebSocket prefix | `/topic`, `/queue`, `/app` | `/sub`, `/pub` 추가(구독/발행 prefix 확장) |
| 메뉴 | "마이페이지" 링크 | "빠른 예약"으로 텍스트 변경 + "자유게시판" 메뉴 신규 추가 |
| 댓글 API 경로 | - | 프론트가 부르던 `/reply/list_vue`를 실제 매핑인 `/board/list_vue`로 수정해 불일치 해소 |
| 대댓글(답글) | 없음 | `group_id`/`group_step`/`group_tab`/`depth`를 MyBatis 어노테이션 쿼리로 조작하는 답글 기능 구현 |
| 실시간 알림 | 없음 | STOMP(`/sub/notice/{id}`)로 댓글 작성자에게 답글 알림을 토스트로 전송 |
| CI | 없음 | `Jenkinsfile` 신규 작성(Git 연결 확인 파이프라인) + ngrok/Docker 권한/Credentials/Pipeline Job/GitHub Webhook까지 Jenkins CI/CD 환경 전체 구축 |

---

## 1. 전역 설정 어노테이션 활성화

```java
@SpringBootApplication
@EnableScheduling
@EnableAspectJAutoProxy
public class SpringPiniaProject2Application {
	public static void main(String[] args) {
		SpringApplication.run(SpringPiniaProject2Application.class, args);
	}
}
```

- `@EnableScheduling` : `@Scheduled` 어노테이션이 붙은 메서드를 스프링이 주기적으로 실행하도록 활성화
- `@EnableAspectJAutoProxy` : `@Aspect` 기반 AOP를 사용할 수 있게 프록시 자동 생성을 켜는 설정
- 실제 사용 예시로 빈 껍데기 클래스 2개가 함께 생성됨(둘 다 내부 구현 없이 골격만 잡아둔 상태)

```java
// FindAop.java — AOP 적용 대상이 될 예정인 빈 클래스
public class FindAop {
}
```

```java
// RealFindWordTask.java — 3분 주기 스케줄 작업 골격
@Component
public class RealFindWordTask {
	@Async
	@Scheduled(fixedRate = 60*3*1000)
	public void task() {
	}
}
```

- `fixedRate = 60*3*1000` → 3분(180,000ms)마다 실행되는 주기 작업으로 설계됨
- `@Async`가 붙어있어 별도 스레드에서 비동기 실행되도록 준비됐지만, 비동기를 활성화하는 `@EnableAsync`는 아직 클래스 어디에도 추가되지 않아 실제로는 동작하지 않는 상태(다음 작업으로 이어질 미완성 구간)

---

## 2. WebSocket Broker prefix 확장

```java
registry.enableSimpleBroker(
        "/topic",
        "/queue",
        "/sub"
);
registry.setApplicationDestinationPrefixes(
        "/app",
        "/pub"
);
```

- 기존 `/topic`(전체 브로드캐스트) / `/queue`(개인 메시지) 구독 prefix에 `/sub`가 추가됨
- 기존 `/app`(클라이언트→서버 요청) prefix에 `/pub`가 추가됨
- 다만 이번 커밋에서 `/sub`, `/pub`를 실제로 사용하는 `@MessageMapping`이나 구독 코드는 아직 없음 — 이후 STOMP 관례(publish/subscribe 네이밍)로 전환하기 위한 사전 준비로 보임

---

## 3. 게시판 JPA 구현 — `BootBoard` Entity

```java
@Entity
@Table(name="bootboard")
@DynamicUpdate
@Data
public class BootBoard {
	@Id
	@GeneratedValue(strategy = GenerationType.IDENTITY)
	private int no;
	private String name;
	private String subject;
	private String content;
	@Column(insertable = true,updatable = false)
	private String pwd;
	private int hit;
	@Column(insertable = true,updatable = false,name = "regdate")
	private LocalDateTime regdate;

	@PrePersist
	public void perSist() {
		regdate=LocalDateTime.now();
	}
}
```

- `@DynamicUpdate` : UPDATE 시 변경된 컬럼만 SQL에 포함시켜 불필요한 컬럼 갱신을 줄임
- `@PrePersist` : INSERT 직전에 자동 호출되는 콜백. `regdate`를 매번 코드로 채우지 않아도 저장 시점에 자동 세팅됨
- `pwd`, `regdate`는 `updatable = false`로 고정 — 최초 등록 이후 값이 바뀌지 않도록 컬럼 자체에서 UPDATE를 차단함

```java
public interface BootBoardRepository extends JpaRepository<BootBoard, Integer> {
	public BootBoard findByNo(int no);
}
```

- `JpaRepository`만 상속해도 `save()`/`findAll(Pageable)`/`count()` 등 기본 CRUD가 자동 제공됨
- `findByNo`만 메소드 규칙으로 추가 정의

---

## 4. `BoardController` — 목록 페이징 / 등록 / 상세(조회수 증가)

```java
@GetMapping("/board/list")
public String board_list(
		@RequestParam(value = "page",required = false) String page,
		Model model) {
	if(page==null) page="1";
	int curpage=Integer.parseInt(page);
	int rowSize=10;
	Pageable pg=PageRequest.of(curpage-1, rowSize,
			Sort.by(Sort.Direction.DESC,"no"));
	Page<BootBoard> pList=bDao.findAll(pg);
	List<BootBoard> list = pList.hasContent() ? pList.getContent() : new ArrayList<>();

	int totalpage=bDao.boardTotalPage();
	model.addAttribute("list",list);
	model.addAttribute("curpage",curpage);
	model.addAttribute("totalpage",totalpage);
	model.addAttribute("main_html","board/list");
	return "main/main";
}
```

- `PageRequest.of(curpage-1, rowSize, Sort...)` : JPA Pageable은 0부터 시작하므로 화면상의 1페이지를 `curpage-1`로 변환해서 전달
- 총 페이지 수는 Pageable이 아니라 서비스의 `boardTotalPage()`(전체 건수를 직접 계산)로 별도 조회

```java
@GetMapping("/board/detail")
public String board_detail(@RequestParam("no") int no,
		Model model,HttpSession session) {
	BootBoard vo=bDao.findByNo(no);
	vo.setHit(vo.getHit()+1);
	bDao.save(vo); // 조회수 증가
	vo=bDao.findByNo(no);

	model.addAttribute("no",no);
	model.addAttribute("vo",vo);
	model.addAttribute("main_html","board/detail");
	return "main/main";
}
```

- 조회수 증가 로직 : 먼저 `findByNo`로 조회 → `hit+1` 세팅 후 `save()`로 UPDATE → 다시 `findByNo`로 최신 값을 재조회해서 화면에 전달
- 세션 기반 중복 조회 방지 로직은 아직 없음(같은 사람이 새로고침할 때마다 조회수가 계속 증가하는 상태)

---

## 5. `BoardServiceImpl` — Repository를 감싸는 얇은 서비스 계층

```java
@Service
@RequiredArgsConstructor
public class BoardServiceImpl {
	private final BootBoardRepository bDao;

	public Page<BootBoard> findAll(Pageable pg) {
		return bDao.findAll(pg);
	}
	public int boardTotalPage() {
		return (int)(Math.ceil(bDao.count()/10.0));
	}
	public BootBoard findByNo(int no) {
		return bDao.findByNo(no);
	}
	public void save(BootBoard vo) {
		bDao.save(vo);
	}
}
```

- 기존에 있던 `BoardService` 인터페이스(빈 껍데기)는 삭제되고, `BoardServiceImpl`이 인터페이스 없이 단독 클래스로 남음
- 페이지당 10건(`/10.0`) 고정값이 컨트롤러(`rowSize=10`)와 서비스(`boardTotalPage`) 두 곳에 중복으로 하드코딩돼 있음

---

## 6. 댓글은 MyBatis로 별도 구현 — `BoardCommentMapper` + XML

```java
@Mapper
@Repository
public interface BoardCommentMapper {
	public List<BootCommentVO> boardCommentListData(Map map);
	public int boardCommentCount(int board_no);
	public void boardCommentInsert(BootCommentVO vo);
}
```

```xml
<select id="boardCommentListData"
resultType="com.sist.web.vo.BootCommentVO" parameterType="int">
  SELECT no,board_no,id,name,msg,
         TO_CHAR(regdate,'yyyy-mm-dd hh24:mi:ss') as dbday,
  		   group_tab
  	FROM bootComment
  	WHERE board_no=#{board_no}
  	ORDER BY group_id DESC , group_step ASC
  	OFFSET #{start} ROWS FETCH NEXT 10 ROWS ONLY
</select>
```

- 게시글(Board) 본문은 JPA, 댓글(Comment)은 MyBatis로 같은 프로젝트 안에서 두 기술을 용도별로 혼용함
- `OFFSET ... ROWS FETCH NEXT ... ROWS ONLY` : Oracle 12c 이상에서 지원하는 페이징 문법으로 댓글 목록도 10건씩 끊어서 조회
- `boardCommentInsert`의 INSERT문은 `group_id`를 서브쿼리(`SELECT NVL(MAX(group_id)+1,1)`)로 직접 계산해서 채번 — 대댓글(그룹) 구조를 준비해둔 흔적

```java
@RestController
@RequiredArgsConstructor
public class BoardCommentRestController {
	@Async
	@GetMapping("/board/list_vue")
	public ResponseEntity<Map> board_List(
			@RequestParam("no") int board_no,
			@RequestParam("page") int page) {
		...
	}
	@Async
	@PostMapping("/reply/insert_vue")
	public ResponseEntity<Map> reply_insert(
			@RequestBody BootCommentVO vo,
			HttpSession session) {
		String id=(String)session.getAttribute("userid");
		String name=(String)session.getAttribute("username");
		vo.setId(id);
		vo.setName(name);
		bMapper.boardCommentInsert(vo);
		...
	}
}
```

- 댓글 작성자 정보(`id`,`name`)는 클라이언트가 아니라 서버가 세션에서 직접 꺼내 세팅 — 클라이언트 위조 방지 패턴(Day15 채팅 sender 처리와 동일한 방식)
- `@Async`가 컨트롤러 메서드에 붙어있지만, 여기서도 `@EnableAsync` 설정이 없어 실제 비동기 실행은 되지 않는 상태
- 다만 프론트(`boardStore.js`)는 `/reply/list_vue`, `/reply/insert_vue`를 호출하는데 실제 매핑은 `/board/list_vue`, `/reply/insert_vue`로 경로가 한쪽만 일치함 — 목록 조회 연동은 아직 안 맞는 상태로 보임

---

## 7. 화면(Thymeleaf) — 목록/등록/상세 + Pinia 댓글 위젯

```html
<!-- board/list.html -->
<tr th:each="vo:${list}">
  <th class="text-center" width="10%">[[${vo.no}]]</th>
  <th width="45%"><a th:href="@{/board/detail(no=${vo.no})}">[[${vo.subject}]]</th>
  <th class="text-center" width="15%">[[${vo.name}]]</th>
  <th class="text-center" width="20%">[[${#temporals.format(vo.regdate,'yyyy-MM-dd')}]]</th>
  <th class="text-center" width="10%">[[${vo.hit}]]</th>
</tr>
```

- `th:each`로 목록을 뿌리고, `#temporals.format`으로 `LocalDateTime`을 화면용 문자열로 변환

```js
// boardStore.js (Pinia) — 댓글 상태/액션
const useBoardStore=defineStore('board_comment',{
	state:()=>({
		list:[], curpage:1, totalpage:0, board_no:0,
		count:0, msg:'', stomp:null, ...
	}),
	actions:{
		async boardCommentListData(board_no){
			this.board_no=board_no
			const res=await api.get('/reply/list_vue',{
				params:{ page:this.curpage, board_no:board_no }
			})
			this.setCommentData(res)
		},
		async boardCommentInsert(msgRef){
			if(this.msg==='') { msgRef?.focus(); return }
			const res=await api.post('/reply/insert_vue',{
				page:this.curpage, board_no:this.board_no, msg:this.msg
			})
			this.setCommentData(res)
		}
	}
})
```

```js
// boardView.js — detail.html에서 별도 Vue 앱으로 댓글 영역만 마운트
const commentApp=createApp({
   setup() {
  		const store=useBoardStore();
		const msgRef=ref(null)
		onMounted(()=>{
			store.sessionId=SESSION_ID
			store.boardCommentListData(BOARDNO)
		})
		return { store, msgRef }
   }
})
commentApp.use(createPinia())
commentApp.mount("#comment")
```

- 게시글 상세 화면(`detail.html`) 안에서 댓글 영역(`#comment`)만 별도의 Vue 앱으로 마운트하는 구조 — 페이지 전체가 아니라 특정 DOM 영역만 Vue로 제어하는 부분 마운트 패턴
- 댓글 목록은 있는지 없는지(`store.count==0`)만 분기 처리돼 있고, 실제 댓글 내용을 `v-for`로 렌더링하는 부분은 아직 구현되지 않음(개수 유무만 표시하는 중간 단계)

---

## 8. 댓글 API 경로 불일치 수정

```js
// boardStore.js
async boardCommentListData(board_no){
	this.board_no=board_no
	const res=await api.get('/board/list_vue', {   // 기존: '/reply/list_vue'
		params:{ page:this.curpage, no:board_no }   // 기존 파라미터명: board_no
	})
	this.setCommentData(res)
}
```

- 프론트가 호출하던 경로/파라미터명을 실제 `@GetMapping("/board/list_vue")`(파라미터명 `no`)에 맞춰 수정 — 지난 회차에 발견했던 경로 불일치가 이번에 해소됨
- `boardCommentInsert`가 성공한 뒤 입력창 초기화(`this.msg=''`)를 추가로 붙여 등록 후에도 이전 텍스트가 남아있던 문제를 해결

---

## 9. 대댓글(답글) 기능 — MyBatis 어노테이션 쿼리 조합

```java
@Select("SELECT id,group_id,group_step,group_tab FROM bootComment WHERE no=#{no}")
public BootCommentVO boardParentInfoData(int no);

@Update("UPDATE bootComment SET group_step=group_step+1 "
		+ "WHERE group_id=#{group_id} AND group_step>#{group_step}")
public void boardGroupStepIncrement(@Param("group_id") int group_id,
		@Param("group_step") int group_step);

@SelectKey(keyProperty = "no", resultType = int.class, before = true,
		statement = "SELECT NVL(MAX(no)+1,1) as no FROM bootComment")
@Insert("INSERT INTO bootComment VALUES("
		+"#{no},#{board_no},#{id},#{name},#{msg},"
		+ "SYSDATE,#{group_id},#{group_step},"
		+ "#{group_tab},#{root},0)")
public void boardCommentReReply(BootCommentVO vo);

@Update("UPDATE bootComment SET depth=depth+1 WHERE no=#{no}")
public void boardDepthIncrement(int no);
```

- XML 매퍼 파일 대신 `@Select`/`@Update`/`@Insert`/`@SelectKey` 어노테이션으로 쿼리를 인터페이스에 직접 작성하는 방식 — 짧은 쿼리는 XML 없이도 매퍼 구현 가능
- 답글 등록 순서: ① 원댓글(`no`)의 `group_id`/`group_step`/`group_tab` 조회 → ② 같은 그룹에서 원댓글보다 `group_step`이 큰 행을 한 칸씩 밀어서(`+1`) 삽입 자리를 확보 → ③ `group_id`는 그대로, `group_step`은 원댓글+1, `group_tab`(들여쓰기 단계)은 원댓글+1, `root`는 원댓글 번호로 채워서 삽입 → ④ 원댓글의 `depth`(답글 개수 표시용)를 `+1`
- `ORDER BY group_id DESC, group_step ASC`(Day16 6절)와 조합하면 같은 그룹 안에서 `group_step` 순서대로 원댓글 바로 아래에 답글이 끼워져 보이는 구조가 됨

```java
// BoardCommentRestController
@PostMapping("/reply/reply_reply_insert_vue")
public ResponseEntity<Map> reply_reply_insert(@RequestBody BootCommentVO vo, HttpSession session) {
	BootCommentVO pvo=bMapper.boardParentInfoData(vo.getNo());
	bMapper.boardGroupStepIncrement(pvo.getGroup_id(), pvo.getGroup_step());
	vo.setGroup_id(pvo.getGroup_id());
	vo.setGroup_step(pvo.getGroup_step()+1);
	vo.setGroup_tab(pvo.getGroup_tab()+1);
	vo.setRoot(vo.getNo());
	vo.setId((String)session.getAttribute("userid"));
	vo.setName((String)session.getAttribute("username"));
	bMapper.boardCommentReReply(vo);
	bMapper.boardDepthIncrement(vo.getNo());
	...
}
```

- 답글 작성자 정보도 목록 등록과 동일하게 세션에서 직접 꺼내 세팅(클라이언트 위조 방지 패턴 재사용)

---

## 10. 실시간 댓글 알림 — STOMP 구독 + Toast UI

```java
private final SimpMessagingTemplate template;
...
if(!pvo.getId().equals(vo.getId())) {
	template.convertAndSend(
		"/sub/notice/"+pvo.getId(),
		"[☠️댓글 알람]"+vo.getId()+"님이 댓글을 달았습니다!!"
	);
}
```

- 답글 작성자와 원댓글 작성자가 다를 때만 알림 전송 — 자기 글에 자기가 답글 달 때는 알림이 가지 않음
- `enableSimpleBroker`에 등록해둔 `/sub` prefix(Day16 2절에서는 쓰지 않던 것)를 이번 회차에 실제로 사용하기 시작함

```js
// boardStore.js — Pinia
connect(id){
	const sock=new SockJS("/chat-ws")
	this.stomp=Stomp.over(sock)
	this.stomp.connect({}, ()=>{
		this.stomp.subscribe('/sub/notice/'+id, msg=>{
			this.showToast(msg.body)
			this.boardCommentListData(this.board_no)
		})
	})
},
showToast(message){
	const toast=document.getElementById("replyToast")
	const toastMsg=document.getElementById("toastMsg")
	toastMsg.innerText=message
	toast.classList.add("show")
	setTimeout(()=>{ hideToast() }, 5000)
}
```

```js
// boardView.js
onMounted(()=>{
	store.sessionId=SESSION_ID
	store.boardCommentListData(BOARDNO)
	store.connect(SESSION_ID)
})
onUnmounted(()=>{
	store.disconnect()
})
```

- 기존 채팅 기능과 같은 STOMP 소켓(`/chat-ws`)을 재사용해 게시글 상세 화면에서도 구독하는 구조
- 알림 수신 시 토스트만 띄우는 게 아니라 `boardCommentListData`를 다시 호출해 댓글 목록도 즉시 갱신
- 다만 `boardView.js`는 언마운트 시 `store.disconnect()`를 호출하는데, store에는 `disConnection()`(오타)만 정의돼 있어 실제로는 존재하지 않는 함수를 호출하는 상태 — 화면을 벗어날 때 소켓 연결 해제가 정상 동작하지 않을 가능성이 있음
- `board/toast.html`을 별도 프래그먼트로 만들어 `detail.html`에서 `th:include`로 삽입 — 화면 조각을 재사용 가능한 단위로 분리하는 Thymeleaf 패턴

```html
<!-- board/detail.html — 댓글 목록 렌더링(들여쓰기 + 답글 폼) -->
<table class="table" v-for="(rvo,index) in store.list" :key="index">
  <tr>
    <td>
      <span v-if="rvo.group_tab>0">
        <span v-for="i in rvo.group_tab">&nbsp;&nbsp;</span>
        <img src="/image/re_icon.png">
      </span>
      ◑◐<span>{{rvo.name}}</span> (<span>{{rvo.dbday}}</span>)
    </td>
    <td>
      <a v-if="rvo.id===store.sessionId">수정</a>
      <a v-if="rvo.id===store.sessionId">삭제</a>
      <a v-if="store.sessionId!==''" @click="store.toggleReply(rvo.no)">
        {{store.reReplyNo===rvo.no?'취소':'댓글'}}
      </a>
    </td>
  </tr>
  <tr v-if="store.reReplyNo===rvo.no">
    <td><textarea v-model="store.replyMsg[rvo.no]"></textarea>
        <button @click="store.boardCommentReplyInsert(rvo.no)">댓글</button></td>
  </tr>
</table>
```

- Day16 7절에서 "개수 유무만 표시하는 중간 단계"였던 부분이 실제 `v-for` 렌더링으로 완성됨
- `group_tab` 값만큼 `&nbsp;&nbsp;`를 반복 출력해 답글 들여쓰기를 표현 — 별도 트리 컴포넌트 없이 평면 리스트를 들여쓰기로만 계층처럼 보여주는 방식
- 답글 버튼은 댓글마다 `reReplyNo`(현재 열려있는 답글 입력창의 댓글 번호)를 토글해서 한 번에 하나의 답글 폼만 열리도록 제어

---

## 11. Jenkinsfile — Git 연결 확인 파이프라인

```groovy
// 최초 버전 — url이 문자열로 감싸이지 않음(문법 오류)
pipeline{
	stages{
		stage('Git Connection Check'){
			steps{
				git branch: 'main',
					url: https://github.com/atg8915/SpringPiniaProject_2.git
			}
		}
	}
}
```

```groovy
// 최종 버전 — url을 문자열로 감싸고 Declarative 문법에 맞게 정리
pipeline {
    agent any
    stages {
        stage('Git Connection Check') {
            steps {
                echo "=============="
                echo "Git 연결 확인"
                echo "=============="
                git branch: 'main',
                    url: 'https://github.com/atg8915/SpringPiniaProject_2.git'
                echo "============="
                echo "Git 연결 완료"
                echo "============="
            }
        }
    }
}
```

- 첫 버전은 `url` 값이 따옴표 없이 그대로 들어가 있어 Groovy 문법 오류가 나는 상태였고 이후 문자열로 감싸 수정함
- Jenkins Declarative Pipeline은 `agent` 지시자가 최상위에 반드시 있어야 함 — 처음 작성 시 빠져 있던 `agent any`를 마지막 정리 단계에서 추가
- 아직은 Git 연결 확인용 `echo` + `git checkout` 단계 하나뿐이고 빌드/배포 단계는 없는 초기 스캐폴딩 상태

---

## 12. Jenkins CI/CD 환경 구축 — Ubuntu 서버 + ngrok + Pipeline Job

Jenkinsfile 자체는 11절에서 다뤘고 여기서는 그 Jenkinsfile이 실제로 동작하기까지 우분투 서버에서 진행한 설정 과정을 정리함.

### 12-1. ngrok 터널 대상을 Jenkins로 전환

```
sudo nano /home/sist/.config/ngrok/ngrok.yml
```

- 기존에는 웹 서버(web)로 연결하던 ngrok 터널을 jenkins로 변경 — 외부에서 ngrok 주소로 접속하면 Jenkins 화면이 뜸

### 12-2. Docker 권한 부여 + Jenkins 기동

```
sudo usermod -aG docker sist
sudo usermod -aG docker jenkins
sudo systemctl start jenkins
```

- `sist`, `jenkins` 계정을 docker 그룹에 추가 — Jenkins 파이프라인이 sudo 없이 docker 명령을 실행할 수 있도록 사전 준비
- Jenkins 서비스 기동 후 ngrok 주소로 접속해 초기 관리자 비밀번호를 입력하고 로그인

### 12-3. 플러그인 설치 + Docker Hub Credentials 등록

- 설정 → 플러그인 → Available plugins에서 필요한 플러그인 설치
- 설정 → Credentials → System → Global → Add Credentials
  - 종류: Username with password
  - Username: GitHub 계정명
  - Password: Docker Hub 토큰 값
  - ID: `dockerhub_info`
- 이 Credentials ID를 이후 파이프라인의 Docker Hub 로그인/푸시 단계에서 참조하도록 등록해둠

### 12-4. Pipeline Job 생성

- 메인 화면 → New Item → 이름 `cicd` → Pipeline 타입 선택
- Build Triggers에서 "GitHub hook trigger for GITScm polling" 반드시 체크 — GitHub push 이벤트가 왔을 때 자동으로 빌드를 트리거하는 핵심 옵션
- Pipeline 정의는 "Pipeline script from SCM" 선택
  - SCM: Git
  - Repository URL: 대상 GitHub 리포지토리 주소
  - Branch: main(작업 브랜치에 맞춰 설정)
- Save 후 고정 링크가 나오면 Job 생성 완료, 메인 화면에서 `cicd` Job 등록을 확인함

### 12-5. GitHub Webhook 연결

- GitHub 리포지토리 Settings → Webhooks에 `<ngrok 주소>/github-webhook/` 등록
- 이 웹훅이 push 이벤트를 Jenkins로 전달하면 12-4에서 체크해둔 GitHub hook trigger가 반응해 빌드가 시작되는 구조

### 12-6. Jenkinsfile 추가 + 최초 빌드 확인

- 스프링부트 프로젝트 루트에 Jenkinsfile 추가(내용은 11절 참고)
- Git push 후 Jenkins 메인 화면에서 실행 버튼을 눌러 최초 빌드 동작을 직접 확인

---

## 13. 다시 만들 때 체크리스트

```text
[전역 설정]
① @EnableScheduling으로 @Scheduled 메서드 활성화
② @EnableAspectJAutoProxy로 @Aspect AOP 프록시 활성화
③ @Async를 실제로 동작시키려면 @EnableAsync를 별도로 추가해야 함(빠뜨리면 동기로 실행됨)

[게시판 JPA 구현]
④ Entity에 @PrePersist로 등록일(regdate) 자동 세팅
⑤ pwd/regdate처럼 등록 후 바뀌면 안 되는 컬럼은 @Column(updatable=false)
⑥ 페이징은 PageRequest.of(화면페이지-1, rowSize, Sort...)로 0-base 변환 주의
⑦ 조회수 증가는 findByNo → hit+1 → save → 재조회 순서로 처리
⑧ 페이지당 건수(rowSize)를 컨트롤러/서비스 두 곳에 중복 하드코딩하지 않도록 상수화 고려

[댓글 MyBatis 구현]
⑨ 게시글은 JPA, 댓글은 MyBatis로 기술을 나눠서 쓸 수 있음(같은 프로젝트 내 혼용 가능)
⑩ Oracle 페이징은 OFFSET n ROWS FETCH NEXT m ROWS ONLY 문법 사용
⑪ 그룹(대댓글) 채번은 서브쿼리(SELECT NVL(MAX(group_id)+1,1))로 처리
⑫ 작성자 id/name은 클라이언트 값이 아니라 세션에서 서버가 직접 꺼내 세팅(위조 방지)
⑬ 프론트가 호출하는 API 경로와 @GetMapping/@PostMapping 경로가 정확히 일치하는지 항상 재확인

[WebSocket prefix]
⑭ enableSimpleBroker / setApplicationDestinationPrefixes에 prefix를 여러 개 배열로 등록 가능
⑮ prefix를 추가만 하고 실제 사용하는 매핑이 없으면 아무 효과가 없으므로, 추가한 prefix는 반드시 사용처까지 확인

[대댓글 / 실시간 알림]
⑯ 답글 삽입 전에 같은 group_id 안에서 원댓글보다 group_step이 큰 행을 먼저 밀어야(+1) 삽입 순서가 꼬이지 않음
⑰ 답글의 group_id/group_step/group_tab/root는 원댓글 정보를 조회해서 채우고, depth는 원댓글 쪽을 증가시킴
⑱ STOMP convertAndSend 대상 경로(/sub/notice/{id})는 subscribe하는 경로와 정확히 일치해야 함
⑲ 프론트 연결(connect)과 해제 메서드 이름이 실제 정의된 메서드명과 일치하는지 항상 재확인(오타 시 조용히 실패)
⑳ 답글 들여쓰기는 별도 트리 구조 없이 group_tab 값만큼 반복 렌더링(&nbsp; 등)으로도 표현 가능

[Jenkinsfile]
㉑ Declarative Pipeline은 최상위에 agent 지시자가 반드시 있어야 함
㉒ git url처럼 문자열 값은 반드시 따옴표로 감싸야 함(안 그러면 파이프라인 파싱 단계에서 오류)

[Jenkins CI/CD 환경 구축]
㉓ Jenkins를 외부에 노출하려면 ngrok 설정에서 터널 대상을 Jenkins 포트로 바꿔야 함
㉔ Jenkins가 docker 명령을 sudo 없이 쓰게 하려면 jenkins 계정을 docker 그룹에 추가해야 함
㉕ Docker Hub 인증은 Credentials(Username with password)로 등록해두고 파이프라인에서 ID로 참조
㉖ Pipeline Job에서 "GitHub hook trigger for GITScm polling"을 체크해야 push 시 자동 빌드가 동작함
㉗ GitHub Webhook 주소는 반드시 `/github-webhook/` 경로로 끝나야 Jenkins가 이벤트를 수신함
㉘ Job의 Pipeline 정의는 "Pipeline script from SCM"으로 설정해야 리포지토리의 Jenkinsfile을 그대로 읽어옴
```
