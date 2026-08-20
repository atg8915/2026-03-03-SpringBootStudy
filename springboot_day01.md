# Spring Boot Day 01 — 프로젝트 구조 + JPA + Thymeleaf + 게시판 CRUD

## 0. 핵심 빠른 참조

| 구분 | JSP MVC (이전) | Spring Boot (현재) |
|------|--------------|-------------------|
| 설정 방식 | web.xml + context.xml | `application.yml` 한 파일 |
| 의존성 관리 | jar 직접 추가 | `build.gradle` (자동 다운로드) |
| 서버 | Tomcat 별도 설치 | 내장 Tomcat (실행만 하면 됨) |
| View 템플릿 | JSP + JSTL | Thymeleaf + HTML |
| DB 연동 | MyBatis 직접 설정 | JPA (자동) / MyBatis (선택) |
| Controller | 커스텀 `@Controller` + `DispatcherServlet` 직접 구현 | Spring이 자동 처리 |
| DAO | DAO 클래스 직접 작성 | `JpaRepository` 인터페이스만 작성 |
| 빌드/배포 | war → Tomcat webapps | jar 하나로 독립 실행 |

---

## 1. Vue와 React의 가상 DOM

```text
기존 jQuery 방식
  데이터 변경 → 개발자가 직접 DOM 조작 (innerHTML, append...)
  → 전체 또는 큰 범위 재렌더링 → 성능 저하

Vue / React 방식 (가상 DOM)
  데이터 변경 → 가상 DOM(메모리 트리) 업데이트
              → diff 알고리즘으로 이전 가상 DOM과 비교
              → 실제 변경된 부분만 실제 DOM에 반영 (최소 조작)
              → 성능 최적화 ⭐

실제 DOM                    가상 DOM (Virtual DOM)
─────────────────────────   ──────────────────────────────
브라우저가 직접 관리         메모리상의 JS 객체 트리
조작 비용이 큼              조작 비용이 거의 없음
직접 수정 → 전체 repaint    diff 후 변경분만 실제 DOM에 적용

Vue와 React의 차이
  Vue  : 반응형 시스템(Reactivity) + 가상 DOM 병행
         → 어떤 데이터가 변했는지 추적 → 해당 컴포넌트만 재렌더링
  React: 오직 가상 DOM diff → setState 호출 → 전체 트리 diff 후 패치

v-memo, v-once → Vue에서 불필요한 가상 DOM diff 자체를 차단하는 수동 최적화
```

---

## 2. Spring Boot 프로젝트 구조

```text
src/main/java
  └─ com.sist.web
       ├─ controller/   @Controller — URL 매핑, 화면 흐름 제어
       ├─ service/      @Service — 비즈니스 로직 (인터페이스 + 구현체)
       ├─ repository/   @Repository — JPA DB 접근
       ├─ entity/       @Entity — 테이블과 1:1 매핑 클래스
       └─ vo/           DTO — 화면 전송용 데이터 객체

src/main/resources
  ├─ templates/         Thymeleaf HTML 파일
  ├─ static/            CSS, JS, 이미지 (정적 리소스)
  ├─ mybatis/mapper/    MyBatis XML (선택)
  └─ application.yml   전체 설정 (DB, 포트, Thymeleaf 등)

build.gradle            의존성 선언 (Maven의 pom.xml 역할)
```

> Spring Boot는 설정을 최소화하고 서버를 내장해서 자동 구성(Auto Configuration)까지 해줌.  
> 이전에 직접 만들었던 `DispatcherServlet`, `ComponentScan`, `application.xml`을 이제는 Spring이 전부 자동으로 처리함.

---

## 3. build.gradle 의존성 설정

```groovy
plugins {
    id 'java'
    id 'org.springframework.boot' version '4.1.0'
    id 'io.spring.dependency-management' version '1.1.7'
}

java {
    toolchain { languageVersion = JavaLanguageVersion.of(21) }
}

dependencies {
    // 핵심 의존성
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'   // JPA
    implementation 'org.springframework.boot:spring-boot-starter-thymeleaf'  // 뷰 템플릿
    implementation 'org.springframework.boot:spring-boot-starter-webmvc'     // MVC
    implementation 'org.mybatis.spring.boot:mybatis-spring-boot-starter:4.0.1' // MyBatis (선택)
    implementation 'org.apache.tomcat.embed:tomcat-embed-jasper:11.0.24'     // JSP 지원 (선택)

    compileOnly      'org.projectlombok:lombok'          // Lombok
    annotationProcessor 'org.projectlombok:lombok'
    developmentOnly  'org.springframework.boot:spring-boot-devtools' // 핫리로드
    runtimeOnly      'com.oracle.database.jdbc:ojdbc11'  // Oracle 드라이버
}
```

| 의존성 | 역할 |
|--------|------|
| `spring-boot-starter-webmvc` | DispatcherServlet, @Controller, @RequestMapping 등 MVC 전체 |
| `spring-boot-starter-data-jpa` | JPA + Hibernate (ORM) |
| `spring-boot-starter-thymeleaf` | HTML 템플릿 엔진 |
| `mybatis-spring-boot-starter` | MyBatis 연동 (JPA와 병행 가능) |
| `spring-boot-devtools` | 코드 수정 시 자동 재시작 (개발 편의) |
| `ojdbc11` | Oracle DB 드라이버 |
| `lombok` | @Data, @RequiredArgsConstructor 자동 생성 |

---

## 4. application.yml 전체 설정

```yaml
server:
  port: 80              # 포트 (기본 8080, 80으로 변경 시 localhost로 접속)

spring:
  thymeleaf:
    cache: false        # 개발 중 캐시 끄기 (수정 즉시 반영)
    encoding: UTF-8
    prefix: classpath:templates/   # HTML 파일 위치
    suffix: .html                  # 확장자
    # JSP의 prefix=/WEB-INF/views/ suffix=.jsp 와 동일한 역할

  datasource:
    url: jdbc:oracle:thin:@localhost:1521:XE
    username: hr
    password: happy
    driver-class-name: oracle.jdbc.OracleDriver

  jpa:
    database: oracle
    properties:
      hibernate:
        dialect: org.hibernate.dialect.OracleDialect
        format_sql: true       # SQL 보기 좋게 출력
        use_sql_comments: true # SQL 주석 출력

mybatis:
  type-aliases-package: com.sist.web.vo     # VO 패키지 (mapper에서 단축명 사용)
  mapper-locations: mybatis/mapper/*.xml     # mapper XML 경로

logging:
  level:
    org.hibernate.SQL: DEBUG          # 실행 SQL 로그 출력
    org.hibernate.orm.jdbc.bind: TRACE # 바인딩 파라미터 출력
```

---

## 5. JPA 핵심 개념

```text
JPA (Java Persistence API)
  = Java 표준 ORM 명세
  = 객체(Entity) ↔ 테이블 자동 매핑
  = SQL 없이 메서드 이름으로 쿼리 자동 생성

Hibernate = JPA의 실제 구현체 (Spring Boot 기본 채택)

MyBatis vs JPA
  MyBatis : SQL 직접 작성, 복잡한 쿼리에 유리 (실무 8:2 비율)
  JPA     : SQL 자동 생성, 단순 CRUD에 유리 (JOIN/서브쿼리 약점)
  → 실무에서 둘 다 병용하는 경우 많음 ⭐
```

### Entity (테이블 매핑)
```java
// JSP MVC의 VO와 역할은 같지만 → 테이블과 직접 연결됨
@Entity
@Table(name = "jpaboard")   // 매핑할 테이블명
@Data
public class BoardEntity {
    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE)
    // Oracle : SEQUENCE 방식 / MySQL : IDENTITY(AUTO_INCREMENT)
    private int no;

    private String name;
    private String subject;
    private String content;
    private String pwd;
    private int hit;
    private Date regdate;
}
```

### Repository (DAO 역할)
```java
// JpaRepository<Entity타입, PK타입> 만 상속하면 기본 CRUD 자동 완성
@Repository
public interface BoardRepository extends JpaRepository<BoardEntity, Integer> {

    // 메서드 이름 규칙으로 SQL 자동 생성
    BoardEntity findByNo(int no);           // SELECT * FROM jpaboard WHERE no=?
    // findByNameContains(String name)      → WHERE name LIKE '%name%'
    // findByHitGreaterThan(int hit)        → WHERE hit > ?
    // findByNoBetween(int a, int b)        → WHERE no BETWEEN a AND b

    // 자동 생성이 어려운 쿼리 → @Query로 직접 작성
    @Query(value = "SELECT no, subject, name, hit, " +
                   "TO_CHAR(regdate,'yyyy-MM-dd') as dbday " +
                   "FROM jpaboard ORDER BY no DESC " +
                   "OFFSET :start ROWS FETCH NEXT 10 ROWS ONLY",
           nativeQuery = true)   // nativeQuery=true : JPQL 변환 없이 SQL 그대로 실행
    List<BoardDTO> boardListData(@Param("start") Integer start);

    // 기본 제공 메서드 (상속으로 자동)
    // save(entity)   → INSERT or UPDATE (PK 있으면 UPDATE, 없으면 INSERT)
    // delete(entity) → DELETE
    // count()        → SELECT COUNT(*)
    // findAll()      → SELECT *
}
```

### DTO (Interface 방식)
```java
// @Query 결과를 받을 때 → interface로 선언하면 JPA가 자동 구현체 생성
// 컬럼명과 메서드명(get+컬럼명) 매핑
public interface BoardDTO {
    int getNo();
    String getName();
    String getSubject();
    String getDbday();    // TO_CHAR 결과 → dbday alias와 매핑
    int getHit();
}
// ⭐ Spring Boot 이후 → public record BoardDTO(int no, String name...) 로 대체 가능
```

---

## 6. Service 레이어 — 인터페이스 + 구현체 패턴

```java
// BoardService.java (인터페이스)
public interface BoardService {
    BoardEntity findByno(int no);
    List<BoardDTO> boardListData(int start);
    void boardUpdate(BoardEntity vo);
    void boardInsert(BoardEntity vo);
    void boardDelete(BoardEntity vo);
    int boardCount();
}

// BoardServiceImpl.java (구현체)
@Service
@RequiredArgsConstructor   // final 필드 → 생성자 주입 자동 생성 (Lombok)
public class BoardServiceImpl implements BoardService {
    private final BoardRepository dao;  // final + @RequiredArgsConstructor = DI ⭐

    @Override
    public void boardInsert(BoardEntity vo) { dao.save(vo); }

    @Override
    public void boardUpdate(BoardEntity vo) { dao.save(vo); }
    // save() : PK 있으면 UPDATE, 없으면 INSERT → insert/update 동일 메서드

    @Override
    public void boardDelete(BoardEntity vo) { dao.delete(vo); }

    @Override
    public int boardCount() { return (int) dao.count(); }
}
```

> **왜 인터페이스를 쓰나?**  
> 나중에 구현체만 교체하면 되므로, MyBatis에서 JPA로 바꾸거나 Oracle에서 MySQL로 전환해도 Controller는 수정할 필요 없음.  
> 테스트 코드를 작성할 때도 Mock 객체로 교체하기 쉬움.

---

## 7. Controller — Spring Boot 방식

```java
@Controller
@RequiredArgsConstructor
@RequestMapping("board/")   // 공통 URL prefix (중복 제거)
public class BoardController {
    private final BoardService bService;  // 생성자 주입

    // GET 목록
    @GetMapping("list")   // board/list
    public String board_list(
        @RequestParam(value="page", required=false) String page,  // null 허용
        Model model) {

        if (page == null) page = "1";
        int curpage = Integer.parseInt(page);
        int start = (curpage * 10) - 10;

        model.addAttribute("list",      bService.boardListData(start));
        model.addAttribute("curpage",   curpage);
        model.addAttribute("totalpage", (int)(Math.ceil(bService.boardCount() / 10.0)));
        return "board/list";   // templates/board/list.html
    }

    // GET 상세 + 조회수 증가
    @GetMapping("detail")
    public String board_detail(@RequestParam("no") int no, Model model) {
        BoardEntity vo = bService.findByno(no);
        vo.setHit(vo.getHit() + 1);
        bService.boardUpdate(vo);         // UPDATE hit=hit+1
        model.addAttribute("vo", bService.findByno(no));  // 갱신 후 재조회
        return "board/detail";
    }

    // POST 등록
    @PostMapping("insert_ok")
    public String board_insert_ok(@ModelAttribute("vo") BoardEntity vo) {
        bService.boardInsert(vo);
        return "redirect:/board/list";    // POST 후 redirect (중복 submit 방지)
    }

    // POST 수정 (비밀번호 검증 후)
    @PostMapping("update_ok")
    public String board_update_ok(@ModelAttribute("vo") BoardEntity vo, Model model) {
        BoardEntity dbVO = bService.findByno(vo.getNo());
        String res = "no";
        if (vo.getPwd().equals(dbVO.getPwd())) {
            vo.setHit(dbVO.getHit());     // 조회수 유지
            bService.boardUpdate(vo);
            res = "yes";
        }
        model.addAttribute("res", res);
        model.addAttribute("no",  vo.getNo());
        return "/board/update_ok";
    }
}
```

| JSP MVC | Spring Boot |
|---------|------------|
| `request.getParameter("page")` | `@RequestParam("page") String page` |
| `request.setAttribute("list", list)` | `model.addAttribute("list", list)` |
| `return "../main/main.jsp"` | `return "board/list"` (templates/ 기준) |
| `return "redirect:..."` | `return "redirect:/board/list"` (동일) |
| `@RequestMapping` 직접 구현 | Spring이 자동 처리 |

---

## 8. Thymeleaf로 JSTL 대체하기

```html
<!-- 선언 : html 태그에 xmlns 추가 -->
<html xmlns:th="http://www.thymeleaf.org">

<!-- 1. 텍스트 출력 (3가지 방식) -->
<td th:text="${vo.no}"></td>          <!-- th:text : 태그 내용 교체 -->
<td>[[${vo.no}]]</td>                 <!-- 인라인 표현식 : 태그 밖에서 사용 -->
<td th:utext="${vo.content}"></td>    <!-- HTML 태그 포함 출력 (innerHTML) -->

<!-- 2. 반복문 : th:each = JSTL c:forEach -->
<tr th:each="vo : ${list}">
    <td th:text="${vo.no}"></td>
    <td><a th:href="@{/board/detail(no=${vo.no})}">[[${vo.subject}]]</a></td>
</tr>

<!-- 3. 링크 : th:href = @{URL(파라미터)} -->
<a th:href="@{/board/detail(no=${vo.no})}">상세</a>
<!-- 결과 : /board/detail?no=1 -->

<!-- 4. 조건문 : th:if / th:unless -->
<script th:if="${res=='yes'}">location.href="/board/list"</script>
<script th:if="${res=='no'}">alert("비밀번호 오류"); history.back()</script>

<!-- 5. 폼 값 바인딩 -->
<input type="text" name="name" th:value="${vo.name}">
<textarea name="content">[[${vo.content}]]</textarea>

<!-- 6. 삼항 연산자 -->
<a th:href="@{/board/list(page=${curpage>1?curpage-1:curpage})}">이전</a>
```

| JSTL (JSP) | Thymeleaf |
|------------|-----------|
| `<c:forEach var="vo" items="${list}">` | `th:each="vo : ${list}"` |
| `<c:if test="${조건}">` | `th:if="${조건}"` |
| `${vo.name}` | `th:text="${vo.name}"` / `[[${vo.name}]]` |
| `<c:url value="/board/list"/>` | `@{/board/list(page=${curpage})}` |

---

## 9. 게시판 CRUD 전체 흐름

```text
[목록] GET /board/list
  → board_list() → boardListData(start) → model → list.html
  → th:each로 목록 출력 / 이전·다음 페이징

[상세] GET /board/detail?no=1
  → board_detail() → findByno(no) → hit+1 → save() → 재조회 → detail.html
  → [[${vo.xxx}]] 인라인 출력

[등록] GET /board/insert → insert.html (빈 폼)
       POST /board/insert_ok
  → @ModelAttribute BoardEntity vo → boardInsert(vo) → redirect:/board/list

[수정] GET /board/update?no=1 → update.html (기존 값 th:value로 채움)
       POST /board/update_ok
  → DB 비밀번호 비교 → 일치 시 save() → update_ok.html → th:if로 분기

[삭제] GET /board/delete?no=1 → delete.html (비밀번호 입력)
       POST /board/delete_ok
  → DB 비밀번호 비교 → 일치 시 delete() → delete_ok.html → th:if로 분기
```

---

## 10. 다시 만들 때 체크리스트

```text
① Spring Boot 프로젝트 생성 (Spring Initializr 또는 STS)
   - 의존성 : Spring Web, Thymeleaf, Spring Data JPA, Lombok, DevTools, Oracle Driver

② application.yml 설정
   - server.port / datasource / jpa.dialect / thymeleaf

③ Entity 작성
   - @Entity, @Table(name="테이블명"), @Id, @GeneratedValue

④ DTO 작성 (목록용 interface)
   - @Query 결과 컬럼명과 getXxx() 메서드명 일치시키기

⑤ Repository 작성
   - extends JpaRepository<Entity, PK타입>
   - 기본 CRUD : save / delete / count / findAll 자동
   - 복잡한 쿼리 : @Query + nativeQuery=true

⑥ Service 인터페이스 + ServiceImpl 작성
   - @Service, @RequiredArgsConstructor, private final Repository

⑦ Controller 작성
   - @Controller, @RequiredArgsConstructor, @RequestMapping("공통경로/")
   - @GetMapping / @PostMapping
   - @RequestParam(required=false) : null 허용 파라미터
   - @ModelAttribute : form 전체를 객체로 받기

⑧ HTML 작성 (templates/ 폴더)
   - th:each / th:text / th:href / th:value / th:if
   - 인라인 : [[${변수}]]
   - 링크 : @{/경로(파라미터=${값})}
```

---

