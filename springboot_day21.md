# 📘 Spring Boot Day 21 — 신규 프로젝트 `SpringPostgreProject` 초기 세팅 + Git 브랜치 전략 도입

## 0. 핵심 빠른 참조 — 이전 대비 오늘 바뀐 점

| 구분 | 이전 | 오늘 |
|------|------|------|
| DB 종류 | Oracle(`ojdbc11`) 중심으로 진행 | **PostgreSQL(`org.postgresql:postgresql`)** 신규 프로젝트 시작 |
| Git 작업 방식 | 로컬에서 바로 `main` 브랜치에 push | **main → develop → feature/기능명** 3단 브랜치 전략 + `gh pr create`/`merge`로 PR 경유 |
| 화면 렌더링 | — | Thymeleaf `th:each` + 인라인 표현식(`[[...]]`)으로 목록 출력 |

---

## 1. 신규 프로젝트 초기 스캐폴딩 (`SpringPostgreProject`)

### build.gradle — PostgreSQL + MyBatis 조합
```gradle
dependencies {
	implementation 'org.springframework.boot:spring-boot-starter-jdbc'
	implementation 'org.springframework.boot:spring-boot-starter-thymeleaf'
	implementation 'org.springframework.boot:spring-boot-starter-webmvc'
	implementation 'org.springframework.boot:spring-boot-starter-webservices'
	implementation 'org.mybatis.spring.boot:mybatis-spring-boot-starter:4.0.1'
	compileOnly 'org.projectlombok:lombok'
	developmentOnly 'org.springframework.boot:spring-boot-devtools'
	runtimeOnly 'org.postgresql:postgresql'
	annotationProcessor 'org.projectlombok:lombok'
}
```
- 지금까지는 `ojdbc11`(Oracle) 드라이버만 써봤는데, 오늘 처음 `org.postgresql:postgresql` 드라이버로 새 프로젝트를 시작함
- ORM은 JPA가 아니라 MyBatis(`mybatis-spring-boot-starter`)만 의존성에 추가함

### application.yml
```yaml
server:
  port: 9090
  servlet:
    context-path: /
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/recipe
    username: postgres
    password: <DB계정>
    driver-class-name: org.postgresql.Driver
  thymeleaf:
    cache: false
    encoding: UTF-8
    prefix: classpath:templates/
    suffix: .html
    mode: HTML
mybatis:
  type-aliases-package: com.sist.web.vo
```
- PostgreSQL 기본 포트(`5432`)로 JDBC URL 구성, 드라이버는 `org.postgresql.Driver`
- `mybatis.type-aliases-package`를 지정해두면 Mapper XML(또는 어노테이션)에서 VO 클래스를 풀 패키지명 없이 짧은 이름으로 참조 가능

### Controller / Mapper / VO 3계층
```java
@Controller
@RequiredArgsConstructor
public class MemberController {
	private final MemberMapper mMapper;

	@GetMapping("/list")
	public String member_list(Model model) {
		List<MemberVO> list = mMapper.memberListData();
		model.addAttribute("list", list);
		return "list";
	}
}
```
```java
@Mapper
@Repository
public interface MemberMapper {
	@Select("SELECT * FROM member")
	public List<MemberVO> memberListData();
}
```
```java
@Data
public class MemberVO {
	private String id, name, sex;
}
```
- `@Select` 어노테이션 방식으로 XML 없이 바로 SQL을 붙이는 가장 단순한 MyBatis 매퍼 형태
- Controller는 Mapper를 직접 주입받아 호출 — Service 계층 없이 최소 구성으로 시작

---

## 2. Thymeleaf 목록 출력 + 인라인 표현식

```html
<ul>
  <li th:each="vo:${list}" style="color: red;">[[${vo.id}]]([[${vo.name}]])</li>
</ul>
```
- `th:each`로 리스트를 순회하면서 `<li>` 태그를 반복 생성
- `[[${...}]]` 형태는 Thymeleaf 인라인 표현식 — 태그 안에서 `th:text` 없이 텍스트 중간에 바로 값을 끼워 넣을 때 사용
- `style="color: red;"`처럼 태그에 인라인 스타일을 직접 붙여 화면 요소 색상 확인 가능

---

## 3. Git 브랜치 전략 — main / develop / feature + PR

```text
git pull origin main
git checkout -b develop
git add .
git commit -m ""
git push
--------------------------
git pull origin develop
git checkout -b feature/login
git add .
git commit -m ""
git push feature/login

main      : 배포용
 |
develop   : 개발 통합
 |
feature/로그인 : 기능 개발 단위 브랜치
```
- `main`(배포) → `develop`(개발 통합) → `feature/기능명`(개별 기능) 순서로 브랜치를 파생시키는 구조
- 새 기능은 항상 `feature` 브랜치에서 작업 후 `develop`으로 병합하고 `develop`이 안정화되면 `main`으로 올리는 흐름

### GitHub CLI로 PR 생성/병합
```bash
gh pr create --base develop --head feature/login
gh pr create --base develop --head feature/login \
  --title "로그인 기능 구현" --body "스프링 보안이용"
gh pr merge
```
- `--base`는 병합 대상 브랜치, `--head`는 병합할 작업 브랜치
- `git status`로 현재 브랜치 상태를 먼저 확인한 뒤 PR을 생성하는 순서로 진행

---

## 4. 비교표 — Git 작업 흐름 (이전 방식 vs 오늘 방식)

| 구분 | 이전 | 오늘 |
|------|------|------|
| 브랜치 구성 | `main` 단일 브랜치에서 바로 작업 | `main`/`develop`/`feature/기능명` 3단 구성 |
| 커밋 반영 방식 | `git push`로 `main`에 직접 반영 | `feature`에서 커밋 → PR 생성 → `develop`으로 병합 |
| PR 도구 | 사용 안 함 | `gh pr create` / `gh pr merge`(GitHub CLI) |
| 목적 | 빠른 반영 | 기능 단위 격리 + 리뷰 경유 후 통합 |

---

## 5. 다시 만들 때 체크리스트

```text
[PostgreSQL + MyBatis 신규 프로젝트]
① build.gradle에 spring-boot-starter-jdbc + mybatis-spring-boot-starter + org.postgresql:postgresql 추가
② application.yml의 datasource.driver-class-name은 org.postgresql.Driver, 포트는 5432 기본값
③ mybatis.type-aliases-package를 지정하면 VO를 짧은 이름으로 매핑 가능
④ Mapper 인터페이스에 @Mapper + @Repository 붙이고 @Select로 SQL 직접 작성(XML 없이 시작 가능)
⑤ DB 계정정보(비밀번호)는 노트/커밋 어디에도 평문으로 남기지 말고 마스킹

[Thymeleaf 목록 출력]
⑥ th:each="변수:${리스트}"로 반복, 태그 안에서 값 출력은 [[${변수.필드}]] 인라인 표현식 사용

[Git 브랜치 전략]
⑦ main(배포)/develop(통합)/feature(기능) 3단 구조로 브랜치 분리
⑧ 새 기능은 반드시 feature 브랜치에서 시작 → develop으로 PR 병합
⑨ gh pr create --base develop --head feature/기능명 --title "..." --body "..."로 PR 생성, gh pr merge로 병합
```
