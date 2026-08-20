# Spring Boot Day 03 — Layout Include 패턴 + VO 프로젝션 + Vuex Store + CI/CD/Docker

## 0. 핵심 빠른 참조

| 구분 | Day 02 방식 | Day 03 방식 (변경점) |
|------|------------|----------------------|
| 화면 구조 | 각 기능마다 별도 HTML (`food/list`, `goods/list`) | `main/main` 단일 레이아웃 + `th:include="${main_html}"`로 내용만 교체 |
| 목록 데이터 반환 | `Entity` 리스트 그대로 | **VO(interface) 프로젝션**으로 필요한 컬럼만 `@Query native` 조회 |
| Vue 상태 관리 | 컴포넌트 내부 `data()`에서 axios 직접 호출 | Vuex `store`(state/mutations/actions)로 중앙 집중 관리 |
| REST 통신 | `@RestController`만 사용 | `@RestController` + `@CrossOrigin`으로 포트 다른 Vue 서버 허용 |

| 계층 | 이번 자료에서 새로 나온 내용 |
|------|------------------------------|
| Repository | `@Query(nativeQuery=true)` + `OFFSET :start ROW FETCH NEXT 12 ROWS ONLY` 페이징 SQL 직접 작성 |
| VO | `interface`로 선언, `@Query` 결과 컬럼과 `getXxx()` 메서드명 매핑 (읽기 전용 프로젝션) |
| Controller | 화면용(`@Controller`, main_html 스위칭) vs API용(`@RestController`, CrossOrigin) 역할 분리 |
| 프론트 | Vuex `store/food.js` — state / mutations / actions 3요소로 서버 데이터 관리 |

---

## 1. Thymeleaf 레이아웃 Include 패턴 — `main.html` + `header.html`

```html
<!-- main.html : 뼈대 페이지, 모든 요청이 여기로 모임 -->
<html xmlns:th="http://www.thymeleaf.org"
      xmlns:layout="http://www.ultraq.net.nz/thymeleaf/layout">
<body>
  <th:block th:include="main/header"></th:block>
  <th:block th:include="${main_html}"></th:block>
</body>
</html>
```

- 동작 원리: Controller에서 `model.addAttribute("main_html","food/list")`처럼 문자열로 페이지 경로를 지정하면 `main.html`이 그 경로를 `th:include`로 끼워 넣음. JSP의 `<jsp:include>`와 같은 개념을 Thymeleaf로 구현한 것이고, header/footer 같은 공통 영역은 한 번만 작성해두고 본문만 요청별로 교체하는 게 장점
- `header.html`: `nav` 메뉴 구성. `<a href="/goods">스토어</a>`처럼 각 모듈 진입 링크만 둠. 주석으로 `Vuex = REST 매핑(SELECT:GET, INSERT:POST, UPDATE:PUT, DELETE:DELETE)` 메모됨

### Controller에서 main_html 지정 흐름 — Food/Goods 공통
```text
GoodsController.goods_page()
  → list, curpage 등 model에 담기
  → model.addAttribute("main_html","goods/list")
  → return "main/main"   ← 항상 이 뼈대로 forward

GoodsController.goods_detail()
  → model.addAttribute("main_html","goods/detail")
  → return "main/main"

MainController.main_page() (Food 목록 = 홈)
  → model.addAttribute("main_html","main/home")
  → return "main/main"
```
> 핵심 주석: `@Controller`는 화면 변경+데이터 전송(Router), `@RestController`는 JSON만 전송 → JavaScript 연동용

---

## 2. VO 프로젝션 — 목록 전용 인터페이스

```java
// FoodVO.java
public interface FoodVO {
	public int getNo();
	public String getName();
	public String getAddress();
	public String getPoster();
}
// GoodsVO.java
public interface GoodsVO {
	public int getNo();
	public String getGoodsName();
	public String getGoodsPrice();
	public String getGoodsPoster();
}
```
- Entity 전체가 아니라 목록 화면에 필요한 컬럼만 뽑아 쓰는 용도. Day01 정리에서 다뤘던 인터페이스 기반 DTO와 원리가 같음
- `// public recode FoodVO => 읽기 전용` 주석 → Java `record`로도 대체 가능하다는 메모. 여기서 recode는 record 오타

### Repository — native query 페이징
```java
// FoodRepository
@Query(value="SELECT no,name,poster,address "
		+ "FROM food "
		+ "ORDER BY no ASC "
		+ "OFFSET :start ROW FETCH NEXT 12 ROWS ONLY",
		nativeQuery = true)
public List<FoodVO> foodListData(@Param("start")int start);

// GoodsRepository
@Query(value="SELECT no,goods_name,goods_poster,goods_price "
		+ "FROM goods_all "
		+ "OFFSET :start ROW FETCH NEXT 12 ROWS ONLY",
		nativeQuery = true)
public List<GoodsVO> goodsListData(@Param("start")int start);

// 공통: 검색/상세용 메서드
findByAddressContains(String address)     // Food, LIKE 검색
findByGoodsNameContains(String goods_name)// Goods, LIKE 검색
findByNo(int no)                          // 상세보기 공통
```

### Service — Food/Goods 동일한 start 계산 + 페이지 블록 계산
```java
// 목록: page → start 변환
int ROWSIZE = 12;
int start = (page * ROWSIZE) - ROWSIZE;
List<FoodVO> list = foodRepo.foodListData(start);

// 페이지 블록 계산 (BLOCK=10) — Day02와 동일 공식 재사용
int totalpage = (int)(Math.ceil(count / 12.0));
startPage = ((page-1)/BLOCK*BLOCK)+1;
endPage   = ((page-1)/BLOCK*BLOCK)+BLOCK;  // totalpage 초과시 보정

// 상세: 조회수 증가 후 재조회 (Goods만 Entity 기반 detail 메서드 존재)
GoodsEntity vo = goodsRepo.findByNo(no);
vo.setHit(vo.getHit()+1);
vo.setNo(no);
goodsRepo.save(vo);
return goodsRepo.findByNo(no);
```

---

## 3. Controller 비교

| Controller | 어노테이션 | 반환 | 용도 |
|-----------|-----------|------|------|
| `MainController` | `@Controller` | `"main/main"` (main_html="main/home") | Food 목록을 홈 화면으로 렌더링 |
| `GoodsController` | `@Controller` | `"main/main"` (main_html="goods/list"/"goods/detail") | Goods 목록/상세를 서버 렌더링 |
| `MainRestController` | `@RestController` + `@CrossOrigin(origins="*")` | `ResponseEntity<Map>` | Vue(포트 다른 프론트)에서 axios로 호출하는 JSON API |

```java
// MainRestController 핵심
@GetMapping("/food/list_vue")
public ResponseEntity<Map> food_list_vue(@RequestParam("page") int page) {
    // list, pages(int[]) 를 Map에 담아 JSON으로 반환
    // 실패 시 500(INTERNAL_SERVER_ERROR)
}
```
- `@CrossOrigin(origins = "*")` 주석: `http://localhost:8081`처럼 포트가 다른 Vue 개발 서버도 이 API를 호출할 수 있게 CORS 허용

---

## 4. Vuex Store 패턴 — `store/food.js`

### 개념 정리 — 파일 내 주석 기준
```text
Vuex 4대 구성요소
  1. state     : 실제 공유 데이터 저장소 (변경되면 UI 자동 반영)
  2. mutations : state를 "동기적으로" 변경 (유일하게 state를 바꿀 수 있는 통로)
  3. actions   : 서버와 통신하는 "비동기" 함수 (axios 요청 담당)
  4. modules   : store를 기능별로 분리해서 관리 (food.js, board.js, goods.js → index.js에서 통합)

데이터 흐름
  Component(View) → dispatch(action)
                        |
                     actions (axios 요청)
                        |
                     commit(mutation)
                        |
                     mutations → state 변경
                        |
                     UI 자동 반영
```

### 코드 구조
```javascript
export default{
    namespaced:true,
    state:{
        food_data:{},
        food_detail:{}
    },
    mutations:{
        SET_FOOD_DATA(state,payload){ state.food_data=payload },
        SET_FOOD_DETAIL(state,payload){ state.food_detail=payload }
    },
    actions:{
        async foodListData({commit},page){
            await axios.get('http://localhost:8080/food/list_vue',{params:{page}})
                .then(response=>{ commit('SET_FOOD_DATA',response.data) })
        }
    }
}
```
- 호출 순서: 컴포넌트 → `dispatch('foodListData', page)` → action이 axios로 서버 호출 → 응답 오면 `commit('SET_FOOD_DATA', ...)` → mutation이 state 갱신 → 화면 자동 반영
- 폴더 구조 메모: `components`는 공통 Header/Footer, `views`는 출력 화면, `router`는 index.js로 화면 이동, `store`는 공통 데이터 — food.js/board.js/goods.js를 index.js로 통합
- 주석의 비교 메모: `vue3(vuex/pinia)` : `JSP(MVC/Spring)` : `react(redux/next)` = 각 프레임워크의 상태관리 대응 관계

---

## 5. CI/CD 개념 정리

```text
CI/CD = DevOps (Developer + Operation, 개발+운영)

CI (Continuous Integration, 지속적 통합)
  → 코드가 정상적으로 통합되는지 자동 검증
  → git push(commit+push) / merge 트리거
  → 코드 체크 → 빌드 → 테스트 → 오류 검증
  → deploy.yml, jenkins, stage 등으로 구성
  → 정상 수행되면 서버로 전송

CD (Continuous Deployment, 지속적 배포)
  → 서버에 실제 배포
  → 시점: CI가 완료된 이후
  → 산출물: Jar(war) 또는 Docker 이미지
  → 서버 재실행까지 자동화
```

---

## 6. Docker 설치 + 이미지 배포 명령어 재정리

```bash
# 1) 필요 도구 설치
sudo apt-get install ca-certificates curl gnupg lsb-release -y   # 도커 사용 도구

# 2) 키 등록
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

# 3) 저장소 목록에 등록
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt update

# 4) 도커 설치
sudo apt install docker-ce docker-ce-cli containerd.io

# 5) 이미지 확인용 명령어
sudo docker images -a
sudo docker ps -a

# 6) 이미지 생성
sudo docker build -t 이미지명 .

# 7) DockerHub 업로드
sudo docker login -u 허브명
sudo docker tag my-app 허브명/my-app
sudo docker push 허브명/my-app
sudo docker pull 허브명/my-app

# 8) 실행
sudo docker run -name s-app -it -d -p 8080:8080 이미지명

# 9) 종료/정리
sudo docker ps -a
sudo docker stop <id>
sudo docker rm <id>
sudo docker rmi 이미지
```
> `=> Dockerfile` 항목만 명시되고 내용은 이전 자료 Day02 참고 — `FROM / WORKDIR / COPY / EXPOSE / ENTRYPOINT` 구조 동일

### Docker Compose 설치 — 도커가 이미 있으면 마지막 단계만
```bash
sudo apt-get update
sudo apt-get install ca-certificates curl gnupg lsb-release

sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

sudo echo \
"deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
$(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get install docker-ce docker-ce-cli containerd.io

sudo curl -L "https://github.com/docker/compose/releases/download/1.28.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
```
> 원본 메모: "도커를 깔았다면 맨 아래 컴포즈만 설치" — 도커가 이미 설치돼 있으면 마지막 `docker-compose` 다운로드 명령만 실행하면 됨

---

## 7. application.yml — Day02 설정 그대로

```yaml
spring:
  datasource:
    url: jdbc:oracle:thin:@localhost:1521:XE
    username: hr
    password: happy
    driver-class-name: oracle.jdbc.driver.OracleDriver
  jpa:
    database: oracle
    properties:
      hibernate:
        dialect: org.hibernate.dialect.OracleDialect
        format_sql: true
        user_sql_comments: true
  thymeleaf:
    cache: false
    encoding: UTF-8
    prefix: classpath:templates/
    suffix: .html
    mode: HTML
mybatis:
  type-aliases-package: com.sist.web.vo
  mapper-locations: mybatis/mapper/*.xml
logging:
  level:
    org.hibernate.SQL: DEBUG
    org.hibernate.orm.jdbc.bind: TRACE
# security / spring ai / websocket(카프카) → 이후 학습 예정 메모 유지
```

---

## 8. 다시 만들 때 체크리스트

```text
[레이아웃 구조]
① main.html(뼈대) + header.html(공통 메뉴) 작성
② main.html에서 th:include="main/header", th:include="${main_html}"로 본문 스위칭

[데이터 계층]
③ Entity: 테이블 매핑 (컬럼 명세 주석 유지)
④ VO(interface): 목록에 필요한 컬럼만 getXxx()로 선언
⑤ Repository: 상세/검색은 findByXxx 메서드 이름 규칙, 목록은 @Query(nativeQuery=true)+OFFSET~FETCH로 페이징 SQL 직접 작성
⑥ Service: page→start 변환, count 기반 totalpage/startPage/endPage(BLOCK=10) 계산, 상세는 hit+1 후 save

[Controller 역할 분리]
⑦ 서버 렌더링용 @Controller: model에 main_html 문자열 지정 → return "main/main" 로 통일
⑧ Vue 연동용 @RestController: @CrossOrigin으로 다른 포트 허용, ResponseEntity<Map>으로 list+pages 반환

[프론트(Vuex)]
⑨ store/기능별.js: state(데이터) / mutations(SET_*, 동기 변경) / actions(async, axios+commit) 3단 구성
⑩ 컴포넌트에서는 dispatch로 action 호출 → mutation → state → 화면 자동 반영

[배포]
⑪ CI: git push/merge → 빌드/테스트 자동 검증
⑫ CD: Jar/Docker 이미지로 서버 배포, 필요 시 docker-compose 설치까지
```

---

