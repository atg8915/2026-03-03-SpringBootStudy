# 📘 Spring Boot Day 13 — SpringPiniaProject_2 댓글 수정/삭제 완성 + Minikube 용어 정리

## 0. 핵심 빠른 참조 — Day12 대비 오늘 바뀐 점

| 구분 | Day12까지 | Day13(오늘) |
|------|-----------|--------------|
| 댓글 기능 범위 | 조회(list_vue) + 등록(insert_vue)만 존재 | 수정(update_vue, PUT) + 삭제(delete_vue, DELETE) 추가로 CRUD 완성 |
| Pinia 스토어 액션 | `commentListData` / `commentInsert` 2개 | `commentUpdate` / `commentDelete` / `toggleUpdate` / `move` 4개 추가 |
| 댓글 화면 | 목록만 출력, 수정/삭제 버튼 없음 | 본인 댓글에만 수정·삭제 버튼 노출, 수정 클릭 시 인라인 textarea 토글 |
| `CommentMapper`의 `@Param` | `org.springframework.data.repository.query.Param`(JPA용 import, 실제로는 안 씀) | `org.apache.ibatis.annotations.Param`(MyBatis용)으로 정정 |
| 댓글 개수 응답 키 | `map.put(count, count)` → key가 문자열 `"count"`가 아니라 변수 `count`(정수)로 들어가던 버그 | `map.put("count", count)`로 수정 |
| Minikube 학습 내용 | 설치·배포 실습 절차 위주 | 클러스터/노드/파드/kubectl 등 핵심 용어 + 도커 이미지 배포 흐름을 개념적으로 재정리 |

---

## 1. 댓글 수정/삭제 REST API — Mapper / Service / Controller / XML

```java
// CommentMapper.java
public interface CommentMapper {
	public void commentDelete(int no);
	public void commentUpdate(CommentVO vo);
}
```

```xml
<!-- comment-mapper.xml -->
<delete id="commentDelete" parameterType="int">
  DELETE FROM piniaComment
  WHERE no=#{no}
</delete>
<update id="commentUpdate" parameterType="com.sist.web.vo.CommentVO">
  UPDATE piniaComment SET
  msg=#{msg}
  WHERE no=#{no}
</update>
```

```java
// CommentRestController.java
@DeleteMapping("/comment/delete_vue")
public ResponseEntity<Map> comment_delete(
		@RequestParam("no") int no,
		@RequestParam("page") int page,
		@RequestParam("fno") int fno)
{
	Map map=new HashMap();
	try {
		cService.commentDelete(no);
		map=commonsData(page, fno);
	}catch(Exception ex) {
		ex.printStackTrace();
		return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).build();
	}
	return ResponseEntity.ok(map);
}

@PutMapping("/comment/update_vue")
public ResponseEntity<Map> comment_update(@RequestBody CommentVO vo)
{
	Map map=new HashMap();
	try {
		cService.commentUpdate(vo);
		map=commonsData(vo.getPage(), vo.getFno());
	}catch(Exception ex) {
		ex.printStackTrace();
		return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).build();
	}
	return ResponseEntity.ok(map);
}
```

- 삭제는 `@RequestParam`(쿼리스트링), 수정은 `@RequestBody`(JSON 바디)로 파라미터를 받음. 삭제는 `no/page/fno` 3개면 되지만 수정은 `CommentVO` 전체(수정할 메시지 포함)를 받아야 해서 방식이 갈림
- 두 메서드 모두 처리 후 `commonsData(page, fno)`를 다시 호출해서 최신 목록/페이징 정보를 응답에 담음. 덕분에 프론트는 목록을 따로 재조회하지 않고 수정·삭제 응답 하나로 화면을 그대로 갱신함
- `CommentService`/`CommentServiceImpl`은 기존 패턴대로 얇은 계층 그대로 둠. Mapper 호출만 받아서 넘김

---

## 2. Pinia 스토어 — 수정/삭제 액션 추가

```javascript
const useCommentStore=defineStore('comment',{
	state:()=>({
		rList:[],
		curpage:1,
		totalpage:0,
		count:0,
		sessionId:'',
		fno:0,
		msg:'',
		upReplyNo:null,
		updateMsg:{},
	}),
	actions:{
		toggleUpdate(no,msg){
			this.upReplyNo=this.upReplyNo===no?null:no
			this.updateMsg[no]=msg
		},
		async commentUpdate(no){
			const res=await api.put('/comment/update_vue',{
				no:no,
				fno:this.fno,
				page:this.curpage,
				msg:this.updateMsg[no]
			})
			this.rList=res.data.rList
			this.curpage=res.data.curpage
			this.totalpage=res.data.totalpage
			this.count=res.data.count
			this.upReplyNo=null
		},
		async commentDelete(no){
			const res=await api.delete('/comment/delete_vue',{
				params:{ page:this.curpage, fno:this.fno, no:no }
			})
			this.rList=res.data.rList
			this.curpage=res.data.curpage
			this.totalpage=res.data.totalpage
			this.count=res.data.count
		},
		move(page){
			this.curpage=page
			this.commentListData(this.fno)
		}
	}
})
```

- `upReplyNo`는 수정 모드인 댓글 번호를 들고 있는 state임. 같은 댓글을 다시 누르면 `toggleUpdate`가 `null`로 되돌려서 수정 모드를 껐다 켰다 함
- `updateMsg`: 댓글 번호(`no`)가 key인 객체. 한 번에 하나만 수정 모드여도 댓글마다 입력 중인 텍스트를 따로 들고 있을 수 있음
- `commentDelete`는 `axios`의 `delete` 메서드에서 body 대신 `params`로 데이터를 보냄. GET처럼 쿼리스트링에 실려 가니 서버가 `@RequestParam`으로 받는 것과 짝이 맞음
- `move(page)`는 이전/다음 버튼에 걸린 페이징 액션. 클릭하면 현재 페이지를 바꾸고 목록을 다시 읽어옴

---

## 3. Thymeleaf 화면 — 수정/삭제 버튼 + 인라인 수정 textarea

```html
<a class="a-link btn btn-xs btn-info" v-if="rvo.id===store.sessionId"
   @click="store.toggleUpdate(rvo.no,rvo.msg)"
>{{store.upReplyNo===rvo.no?'취소':'수정'}}</a>
<a class="a-link btn btn-xs btn-warning" v-if="rvo.id===store.sessionId"
   @click="store.commentDelete(rvo.no)"
>삭제</a>
```

```html
<tr v-if="store.upReplyNo===rvo.no">
  <td colspan="2">
    <textarea rows="4" cols="70" style="float: left"
        v-model="store.updateMsg[rvo.no]"
    ></textarea>
    <button type="button" class="btn-primary"
        @click="store.commentUpdate(rvo.no)"
    >수정</button>
  </td>
</tr>
```

- 수정/삭제 버튼은 `v-if="rvo.id===store.sessionId"` 조건을 걸어 댓글 작성자 본인 것에만 노출됨. `sessionId`에는 화면 진입 시 `onMounted`에서 서버가 내려준 로그인 세션 아이디를 넣음
- "수정" 버튼을 누르면 버튼 텍스트가 `수정 ↔ 취소`로 바뀌고 그 아래 `<tr>`에 textarea가 열리는 토글 방식임. 실제 저장은 textarea 옆 별도 "수정" 버튼(`store.commentUpdate`)이 처리함
- `<script th:inline="javascript">`로 `NO`뿐 아니라 `SESSION_ID`(`${session.userid}`)도 JS 변수로 꺼내 `onMounted` 시점에 `store.sessionId`에 대입함

---

## 4. 사소하지만 중요한 버그 수정 2건

| 위치 | 수정 전 | 수정 후 | 문제 |
|------|---------|---------|------|
| `CommentMapper.java` import | `org.springframework.data.repository.query.Param` | `org.apache.ibatis.annotations.Param` | Spring Data JPA용 `@Param`을 MyBatis Mapper에 잘못 가져다 씀. MyBatis는 자기 패키지의 `@Param`이어야 파라미터 바인딩이 정상 동작함 |
| `CommentRestController.comment_list` | `map.put(count, count)` | `map.put("count", count)` | 문자열 리터럴 `"count"` 대신 `int` 변수 `count`를 키로 넣어서, 실제로는 `count`값(정수)이 키가 되고 프론트에서 `res.data.count`로 못 꺼내던 상태였음 |

- 두 버그 모두 컴파일 에러가 안 남. 조용히 동작만 어긋나니 실행해서 값이 안 들어오는 걸 보고서야 발견하는 흔한 실수임
- MyBatis Mapper를 만질 때는 `org.apache.ibatis.annotations.Param`을 import 했는지부터 확인하는 습관이 필요함

---

## 5. Minikube 개념 정리 — 용어 + 이미지 배포 흐름

```text
도커 이미지 배포 흐름
  docker build -t 이미지명 .              # 이미지 생성
  docker tag 이미지명 허브명/이미지명       # 태그 생성
  docker login -u 허브명                  # 도커 허브 연결
  docker push 허브명/이미지명              # 허브에 업로드
  docker pull 허브명/이미지명              # 다른 서버에서 다운로드
  → deployment.yaml에 이미지 등록

minikube 실행 순서
  1. minikube start
  2. kubectl apply -f ~/k8s/deployment.yaml
  3. kubectl get pods         (running 아니면 kubectl logs pod_name으로 원인 확인)
  4. kubectl get svc
  5. minikube service service_name
```

```text
쿠버네티스 핵심 용어
  1. 클러스터 : 쿠버네티스가 관리하는 전체 컴퓨터 묶음 (Minikube는 1개만 사용)
  2. 노드     : 클러스터를 이루는 각 컴퓨터(서버) 한 대
  3. 파드     : 실행 가능한 가장 작은 단위, 내부적으로 컨테이너(도커)를 담고 있음
  4. 크루(kubectl) : 쿠버네티스에 명령을 내리는 CLI 도구

계층 구조
  컴퓨터 → 쿠버네티스 클러스터 → 노드(서버) → 파드(실행 파일 모음) → 컨테이너(실행 단위/앱)
```

- Minikube는 비용이 안 들고 실제 클러스터 운영 없이 자신의 컴퓨터에서 쿠버네티스 개념(오브젝트 모델)을 테스트해볼 수 있어서 씀
- "클러스터 - 노드 - 파드 - 컨테이너"는 아래로 갈수록 실행 단위가 작아지는 계층 구조임. 파드가 곧 컨테이너는 아니고 파드 안에 컨테이너가 들어 있음
- 실제 서비스 환경에서는 게시판/상품/회원처럼 서버를 여러 개(MSA)로 나눠 쿠버네티스로 묶어 관리함. 비용 부담이 적은 대안으로는 `docker-compose`도 있음

---

## 6. 다시 만들 때 체크리스트

```text
[댓글 수정/삭제 API]
① 삭제는 @RequestParam(쿼리스트링) / 수정은 @RequestBody(JSON)로 파라미터 수신 방식이 다름
② 처리 후 commonsData(page, fno)로 최신 목록을 다시 만들어 응답에 실어보낼 것(프론트 재조회 불필요)
③ MyBatis Mapper의 @Param은 반드시 org.apache.ibatis.annotations.Param을 import할 것

[Pinia 스토어]
④ 수정 모드 토글 state(upReplyNo)와 댓글별 임시 입력값(updateMsg[no])을 분리해서 관리
⑤ axios.delete는 body 대신 { params: {...} } 형태로 쿼리스트링 전달

[화면]
⑥ 수정/삭제 버튼은 rvo.id===store.sessionId로 작성자 본인 여부를 반드시 검사할 것
⑦ 응답 Map에 값 넣을 때 map.put("count", count)처럼 키는 항상 문자열 리터럴로 쓸 것(변수 그대로 넣지 않기)

[Minikube]
⑧ 이미지 배포 흐름 순서: build → tag → login → push → pull → deployment.yaml 등록
⑨ 용어는 "클러스터 > 노드 > 파드 > 컨테이너" 계층으로 암기, kubectl은 명령 도구(리모컨)로 구분
```
