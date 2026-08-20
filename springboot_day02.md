# Spring Boot Day 02

AOP + 예외처리 + Food/Goods CRUD(Thymeleaf vs Vue) + Docker 배포

## 0. 핵심 빠른 참조

| 구분 | Food (Thymeleaf 방식) | Goods (Vue.js + Axios 방식) |
|------|----------------------|------------------------------|
| Controller | `@Controller` | `@RestController` |
| 반환값 | `String` (뷰 이름) | `ResponseEntity<Map>` (JSON) |
| 데이터 렌더링 위치 | 서버 (Thymeleaf가 HTML에 값 삽입) | 클라이언트 (Vue가 axios로 받아 렌더링) |
| 화면 문법 | `th:each`, `th:text`, `[[${}]]` | `v-for`, `{{ }}`, `axios.get()` |
| 페이징 처리 | 서버에서 계산 → Model로 전송 | 서버에서 계산 → JSON으로 전송 → Vue가 클릭 이벤트로 재요청 |
| 요청 흐름 | 요청마다 새 HTML 전체 반환 | 최초 1회 HTML 로드 후 데이터만 비동기 교체 |

| 공통 계층 구조 | 역할 |
|---------------|------|
| Entity | 테이블 매핑 (`@Entity`, `@Table`, `@Id`) |
| Repository | `JpaRepository` 상속, `findByNo` 등 메서드 자동 생성 |
| Service (인터페이스+Impl) | 페이징 계산, hit(조회수) 증가 등 비즈니스 로직 |
| Controller | 요청 매핑 + Model/JSON 응답 |

---

## 1. AOP (`FoodAOP.java`)

```java
@Aspect  // 공통으로 적용되는 기능
@Component // 일반 클래스 => 메모리 할당
/*
 * 	 메소드 호출 위치
 * 	   @Before / @After / @AfterThrowing / @AfterReturning
 * 				  finally     catch			 return : 정상 수행
 * 	   => 메소드 진입전
 *     => try => @Around
 * 	 어떤 메소드에 적용
 *     => excution("* 패키지.클래스.메소드(매개변수)")
 *     					   -----   *   ..
 *     					   *Controller
 */
// 트랜잭션 / 로그파일  Footer에 들어가는 데이터 출력
// Cookie
public class FoodAOP {
	@Around("execution(* com.sist.web.controller.*Controller.*(..))")
	public Object log(ProceedingJoinPoint jp) throws Throwable {
		// 메소드 진입 전 → proceed() → 메소드 진입 후, 실행시간 측정
	}
}
```

- `com.sist.web.controller` 패키지 아래 `*Controller` 클래스의 모든 메서드에 걸림
- 동작 시점은 `@Before`가 진입 전, `@After`가 finally, `@AfterThrowing`이 catch, `@AfterReturning`이 정상 반환. `@Around`는 진입 전과 후를 모두 잡아 `try`로 감싸는 개념임
- 주석에 적힌 용도 예시는 트랜잭션 처리, 로그 파일 기록, 공통 Footer 데이터, 쿠키 처리. Controller마다 반복되는 로직을 한 곳으로 분리함

---

## 2. 전역 예외 처리 (`CommonsException.java`)

```java
@ControllerAdvice  // 통합 예외처리
public class CommonsException {
	@ExceptionHandler(Exception.class)
	public void excetion(Exception ex) { ex.printStackTrace(); }

	@ExceptionHandler(Throwable.class)
	public void throwable(Throwable ex) { ex.printStackTrace(); }
}
```

- `@ControllerAdvice`를 붙이면 모든 Controller에서 발생하는 예외를 한 클래스가 받아 통합 처리함
- `@ExceptionHandler(예외타입.class)`는 특정 예외가 발생했을 때 실행할 메서드를 지정. Exception과 Throwable을 각각 잡아 대응시킴

---

## 3. Thymeleaf 문법 총정리 (`FoodController.java` 주석 기준)

### 요청 처리 흐름
```text
브라우저 요청
   |
DispatcherServlet
   |
@Controller
   |
@Service
   |
@Repository
   |
Oracle 연동
   |
Model 객체 생성
   |
ThymeLeaf 전송 (JSP=> Java+HTML 대비 → HTML + Model → 파싱)
```

### 표현식 5종
| 표현식 | 의미 | 예시 |
|--------|------|------|
| `${}` | Model에서 전송한 데이터 (EL) | `${vo.name}` |
| `*{}` | Form 객체 (사용빈도 낮음) | `*{name}` |
| `#{}` | properties 값 | `#{title}`, `#{db['driver']}` |
| `@{}` | URL 생성 | `@{/board/insert}` |
| `~{}` | Fragment(Layout) include | `~{layout/header}` |

### 디렉티브(th: / v-) 정리
| 디렉티브 | 설명 |
|----------|------|
| `th:text="${vo.name}"` | 텍스트 출력, `v-text`/`{{}}` 대응 |
| `th:utext=""` | HTML 태그 포함 출력 (XSS 위험 → 가급적 사용 금지 권장) |
| `th:value` | input 값 채우기 |
| `th:href="@{/food/detail(no=1)}"` | `/food/detail?no=1` 로 변환 |
| `th:src` | img/embed/iframe 등 데이터 출력이 필요한 src |
| `th:each="vo:${list}"` | 반복문. `vo,status:${list}` 형태로 `status.index/count/first/last/even/odd` 사용 가능 |
| `th:if` / `th:unless` | 조건문 |
| `th:switch` / `th:case` | switch문 |
| `th:selected="${name=='홍길동'}"` | option 선택 |
| `th:style`, `th:replace`, `th:insert` | 스타일/조각 삽입(include) |
| `#regdate.format(today,'yyyy-MM-dd')` | 날짜 포맷 객체 |
| `#number.sequence(startPage,endPage)` | 페이지 번호 시퀀스 생성 |

### 반환값 / 매개변수 정리
| 항목 | 설명 |
|------|------|
| `return "food/detail"` | forward, Model 전송 있을 때 |
| `return "redirect:/food/detail"` | redirect, `_ok`(update/insert/delete) 후 request 초기화되므로 사용 |
| `void` 반환 | 다운로드 또는 `@ResponseBody`로 JSON 전송 시 |
| `@RequestParam("데이터명")` | 일반 파라미터 1개씩 받기 |
| `@ModelAttribute("vo")` | VO 전체 받기 |
| 내장 객체 | `HttpServletRequest`, `HttpServletResponse`, `HttpSession`, `Model`, `RedirectAttributes`, (로그인 시)`Principal` |

---

## 4. Food 모듈

Thymeleaf 서버사이드 렌더링 방식.

### FoodEntity (`food` 테이블)
- PK는 `no`. 주요 필드는 int로 `cno, likecount, jjimcount, hit, replycount`, String으로 `name, reserve, images, address, phone, parking, poster, time, content, price, theme, type`, double로 `score`

### FoodRepository 메서드 이름 규칙
```text
findByNo(int no)                → SELECT * FROM food WHERE no=?
findByAddressContains(address)  → LIKE '%address%'
findByAddressStartsWith(address)→ LIKE 'address%'
findByAddressEndsWith(address)  → LIKE '%address'
findTop5By()                    → 상위 5건
findDistinctBy()                → 중복 제거
findBy컬럼명Between              → BETWEEN
findByOrderBy컬럼명Desc          → ORDER BY ... DESC
count() / save() / delete()     → 기본 제공
```

### FoodServiceImpl 페이징 + 조회수 처리 핵심 로직
```java
// 목록: 페이지당 12건, no 오름차순 정렬
Pageable pg = PageRequest.of(page-1, 12, Sort.by(Sort.Direction.ASC,"no"));
Page<FoodEntity> pList = foodRepo.findAll(pg);
// Page => List 변환 (hasContent() 체크 후 getContent())

// 상세: 조회 → hit+1 → save(update) → 재조회 후 반환
FoodEntity vo = foodRepo.findByNo(no);
vo.setHit(vo.getHit()+1);
foodRepo.save(vo);
return foodRepo.findByNo(no);

// 페이지 블록 계산 (BLOCK=10)
startPage = ((page-1)/BLOCK*BLOCK)+1
endPage   = ((page-1)/BLOCK*BLOCK)+BLOCK  (totalpage 초과 시 totalpage로 보정)
```

> 주석으로 남긴 계층 역할. Controller는 요청값·결과값만 다뤄 JSP/HTML을 전송하고, Service가 요청을 처리하고, Repository는 DB 연결만 맡음

### FoodController 목록·상세
```text
GET /food/list?page=n
  → foodListData(page) + getPageData(page)
  → model: list, curpage, totalpage, startPage, endPage
  → return "food/list"

GET /food/detail?no=n
  → foodDetailData(no) (내부에서 hit 증가)
  → model: vo
  → return "food/detail"
```

### list.html / detail.html (Thymeleaf)
- 목록 화면은 `th:each="vo:${list}"`로 돌리고 `@{/food/detail(no=${vo.no})}` 링크를 검. 페이징은 `th:if="${startPage>1}"`, `#numbers.sequence(startPage,endPage)`, `th:class="${curpage==i?'active':''}"` 로 처리
- 상세 화면은 `[[${vo.name}]]` 인라인 표현식으로 각 필드를 출력하고, 이미지는 `th:src="${vo.poster}"`

---

## 5. Goods 모듈

Vue.js + REST API 방식으로, Food와 대비됨.

### GoodsEntity (`goods_all` 테이블)
- PK는 `no`. 주요 필드는 int로 `goods_discount, hit, replycount, jjimcount`, String으로 `goods_name, goods_sub, goods_price, goods_first_price, goods_poster, goods_delivery`

### GoodsController vs GoodsRestController
```text
GoodsController (@Controller)
  GET /goods/list → return "goods/list"  (빈 HTML 뼈대만 반환, 데이터는 없음)

GoodsRestController (@RestController)
  GET /goods/list_vue?page=n
    → goodsListData(page) + goodsPageData(page)
    → Map{list, curpage, totalpage, startPage, endPage}
    → ResponseEntity.ok(map)  (JSON 반환)
    → 예외 시 ResponseEntity.status(INTERNAL_SERVER_ERROR)
```

### GoodsServiceImpl
- 페이징 로직은 12건·BLOCK=10으로 Food와 동일하고, `save()` 기반 hit 증가 패턴도 그대로 재사용

### list.html (Vue 3 + Axios)
```javascript
// 핵심 흐름
mounted() → dataRecv()
dataRecv(): axios.get('/goods/list_vue', {params:{page:curpage}})
            → 응답을 list/curpage/totalpage/startPage/endPage에 매핑
move(page): curpage 변경 → dataRecv() 재호출 (페이지 클릭 시 서버 재요청)
range(start,end): 페이지 번호 배열 생성 (v-for용)
```
- 템플릿 문법은 `v-for="vo in list"`, `{{vo.goods_name}}`, `:src="vo.goods_poster"`, `@click="move(i)"`
- Food(Thymeleaf)와의 차이는 여기서 갈림. Food는 서버가 매 요청마다 완성된 HTML을 내려주고, Goods는 최초 빈 HTML만 받은 뒤 Vue가 JSON을 받아 화면을 그림. SSR과 CSR의 대비 구조임

---

## 6. application.yml 핵심 설정

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
```
- JPA/Hibernate가 DB별 SQL 문법을 자동 생성한다고 주석에 메모해 둠. `NVL` vs `IFNULL`, `OFFSET` vs `LIMIT`, `SYSDATE`/`NOW()`/`CURRENT_DATE` 등 DB마다 다름
- 하단에 `# security / # spring ai / # websocket = 카프카` 로 다음 학습 예정 항목 메모

---

## 7. 우분투 26 서버 환경 설정

실습 순서 그대로 정리.

```bash
# 1) SSH 설치 및 실행
sudo apt install openssh-server ssh
sudo systemctl enable --now ssh
sudo systemctl status ssh   # 확인

# 2) Java 21 설치
sudo apt install openjdk-21-jdk -y

# 3) 환경변수 등록
sudo nano ./.bashrc
# export JAVA_HOME="경로"
# export PATH=$PATH:...
source ./.bashrc

# 4) Git 설치
sudo apt install git -y

# 5) 프로젝트 복사 및 이동
cd 2026-03-03-springBootstudy
sudo cp -r SpringThymeLeafProject_1/ ~/app
cd app

# 6) application.yml 내부 IP 수정
sudo nano application.yml   # localhost → 실제 서버 IP로 변경

# 7) Gradle 빌드
chmod +x gradlew
./gradlew build             # 또는 sudo ./gradlew build

# 8) 빌드 결과 확인 및 실행
cd build/libs
java -jar 프로젝트명-SNAPSHOT.jar
```

---

## 8. Docker 배포 파이프라인

### 최초 설치 (1회)
```bash
sudo apt-get install ca-certificates curl gnupg lsb-release -y
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt install docker-ce docker-ce-cli containerd.io -y
sudo docker login atg8915   # 도커허브 계정
```

### Dockerfile

프로젝트 클론 폴더 안에 위치해야 함.

```dockerfile
FROM eclipse-temurin-21-jdk
WORKDIR /app
COPY build/libs/*-SNAPSHOT.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java","-jar","app.jar"]
```

### 배포 전체 흐름
```text
① Dockerfile 작성 (Spring 클론 폴더 안)
② sudo docker build -t food-app .              # 이미지 생성 (이름은 임의 지정 가능)
   sudo docker images -a                        # 확인
③ sudo docker tag food-app atg8915/food-app     # 도커허브 태그
④ sudo docker push atg8915/food-app             # 업로드
⑤ (다른 서버에서) sudo docker pull atg8915/food-app
⑥ sudo docker run --name food-app -it -d -p 8080:8080 이미지ID
⑦ sudo docker ps -a                             # 실행 확인
```

### 방화벽 확인

선택 사항. 확인 후 바로 지워도 됨.

```bash
sudo netstat -tnlp | grep 8080
sudo ufw status
sudo ufw allow 8080
sudo ufw enable
sudo ufw status
```

### 정리/삭제 명령
```bash
sudo docker stop food-app
sudo docker rm food-app
sudo docker rmi atg8915/food-app:latest
sudo docker rmi food-app:latest
```

---

## 9. 다시 만들 때 체크리스트

```text
[서버 준비]
① Ubuntu에 ssh/java21/git 설치 → .bashrc에 JAVA_HOME 등록
② 프로젝트 clone/복사 → application.yml의 datasource url IP 수정
③ ./gradlew build → build/libs/*.jar 생성 확인

[Spring Boot 코드 - 계층별]
④ Entity: @Entity, @Table(name=""), @Id + 필드 (테이블 명세 주석으로 남겨두기)
⑤ Repository: extends JpaRepository, findByXxx 메서드 이름 규칙 활용
⑥ Service(인터페이스+Impl): 페이징(Pageable/PageRequest/Sort) + hit 증가 로직(@Service, @RequiredArgsConstructor)
⑦ Controller 선택
   - 서버 렌더링(Thymeleaf) → @Controller, Model에 담아 view 이름 반환
   - REST/Vue 연동 → @RestController, ResponseEntity<Map>으로 JSON 반환
⑧ 공통기능
   - AOP(@Aspect+@Around)로 Controller 호출 로그/시간 측정 공통화
   - @ControllerAdvice + @ExceptionHandler로 예외 통합 처리

[화면]
⑨ Thymeleaf: th:each/th:if/th:text/[[]]/@{} 로 목록·상세·페이징 구현
⑩ Vue+Axios: mounted()에서 axios.get으로 REST 컨트롤러 호출 → v-for로 렌더링, 페이지 클릭 시 재요청

[배포]
⑪ Dockerfile 작성(클론 폴더 내부) → build → tag → push → (배포서버) pull → run -p 8080:8080
⑫ 필요 시 ufw로 8080 포트 허용
```

