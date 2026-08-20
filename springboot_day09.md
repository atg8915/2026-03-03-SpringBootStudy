# 📘 Spring Boot Day 09 — SpringPiniaProject_2 신규 프로젝트 스캐폴딩 (Food/Recipe/Chef 재시작 + Pinia CDN 연동)

## 0. 핵심 빠른 참조 — 어제(Day07) 대비 오늘 바뀐 점

| 구분 | Day07까지 | Day09 (오늘) |
|------|-----------|--------------|
| 프로젝트 | 기존 프로젝트에서 Emp/Dept로 JPA 3방식(메소드규칙/JPQL/QueryDSL) 실습 | **`SpringPiniaProject_2`** 새 저장소로 **처음부터 재스캐폴딩** (커밋 1개로 전체 골격 생성) |
| 도메인 | Emp/Dept (실습용) | **Food / Recipe / Chef** (Day01~06 때 다루던 실전 도메인으로 복귀) |
| 화면 스택 | (언급 없음) | Thymeleaf + **Bootstrap 3** + **Vue 3(CDN)** + **Pinia(CDN)** + axios + vue-demi를 `main.html`에 한 번에 로드 |
| 데이터 접근 | JPA 방식 3종 위주 | **MyBatis**(어노테이션 + XML 혼용) 우선 세팅, JPA/QueryDSL 의존성은 build.gradle에만 선언(아직 Entity/Q-class 없음) |
| 배포 설정 | — | `Dockerfile`, `.gitattributes`, `.gitignore` **처음 추가** |
| DB 계정정보 | Day07부터 `${DB_URL}` 등 환경변수 분리 (계속 유지) | 동일하게 환경변수 분리 유지 (`${DB_URL}`, `${DB_USERNAME}`, `${DB_PASSWORD}`) |
| 서버 포트 | (이전 프로젝트 값) | `9090` |

> 오늘 커밋은 메시지가 `"1111"`로 부실하지만 diff상으로는 완전히 새로운 프로젝트의 초기 골격 1커밋임. build.gradle, gradle wrapper, Application 클래스, Controller/Mapper/Service/VO, application.yml, Thymeleaf 템플릿 3개, Dockerfile, git 설정 파일까지 한 번에 추가됨

---

## 1. 프로젝트 초기 설정 파일

### `.gitattributes` / `.gitignore`
```text
# .gitattributes
/gradlew text eol=lf
*.bat text eol=crlf
*.jar binary
```
- Windows/Linux를 혼용하는 개발 환경에서 줄바꿈(EOL)이 틀어지지 않도록 `gradlew`는 LF, `.bat`은 CRLF로 강제 고정
- `.gitignore`는 Spring Initializr 기본 템플릿 그대로 두고 `.gradle`, `build/`, IDE 설정 폴더 등을 제외함

### `Dockerfile`
```dockerfile
FROM eclipse-temurin:21-jdk
WORKDIR /app
COPY build/libs/*-SNAPSHOT.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java","-jar","app.jar"]
```
- Java 21 기반 이미지로 빌드된 jar를 실행하는 가장 단순한 형태
- `EXPOSE 8080`인데 `application.yml`의 실제 포트는 `9090`이라 컨테이너 실행 시 포트 매핑에 주의 필요. `-p 9090:9090`으로 맞추거나 Dockerfile의 EXPOSE 값을 9090으로 통일해야 함

### `settings.gradle`
```groovy
rootProject.name = 'SpringPiniaProject_2'
```

---

## 2. build.gradle — 의존성 구성

```groovy
plugins {
	id 'java'
	id 'org.springframework.boot' version '4.1.0'
	id 'io.spring.dependency-management' version '1.1.7'
}

java {
	toolchain {
		languageVersion = JavaLanguageVersion.of(21)
	}
}

dependencies {
	implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
	implementation 'org.springframework.boot:spring-boot-starter-jdbc'
	implementation 'org.springframework.boot:spring-boot-starter-thymeleaf'
	implementation 'org.springframework.boot:spring-boot-starter-webmvc'
	implementation 'org.springframework.boot:spring-boot-starter-webservices'
	implementation 'nz.net.ultraq.thymeleaf:thymeleaf-layout-dialect:4.0.1'
	implementation 'org.mybatis.spring.boot:mybatis-spring-boot-starter:4.0.1'
	implementation 'com.querydsl:querydsl-jpa:5.0.0:jakarta'
    annotationProcessor "com.querydsl:querydsl-apt:5.0.0:jakarta"
    annotationProcessor "jakarta.annotation:jakarta.annotation-api"
    annotationProcessor "jakarta.persistence:jakarta.persistence-api"
	compileOnly 'org.projectlombok:lombok'
	developmentOnly 'org.springframework.boot:spring-boot-devtools'
	runtimeOnly 'com.oracle.database.jdbc:ojdbc11'
	annotationProcessor 'org.projectlombok:lombok'
	...(test 의존성 동일 패턴)
}
```
- JPA + MyBatis + QueryDSL을 처음부터 동시에 선언함. "단순 조회는 메소드규칙, 정적 SQL은 JPQL, 복잡한 동적쿼리는 QueryDSL, MyBatis는 그 외 케이스"라는 Day07 조합을 이 프로젝트에서도 이어갈 준비
- 다만 오늘 커밋에는 `@Entity`가 붙은 실제 JPA 엔티티나 QueryDSL Q-class가 아직 없음. VO들이 `jakarta.persistence.Entity`/`Id`/`Table`을 import만 해두고 실제로는 `@Data`만 붙인 순수 DTO 상태이며, JPA 전환 전 단계임
- Gradle Wrapper `9.5.1`로 업데이트됨 (`gradle-wrapper.properties`)

---

## 3. 화면 흐름 — RouterController + main_html 스위칭

```java
@Controller //화면 변경
public class RouterController {
	@GetMapping("/")
	public String main_main(Model model)
	{
		model.addAttribute("main_html","main/home");
		return "main/main";
	}
}
```
```html
<!-- main.html -->
<body>
  <th:block th:include="main/header"></th:block>
  <th:block th:include="${main_html}"></th:block>
</body>
```
- 레이아웃 패턴은 Day03 방식을 채택함. `main_html` 모델 속성으로 include 대상을 스위칭하는 구조임
- `/` 요청 → `main_html="main/home"` 세팅 → `main.html`이 `header` + `home`을 조립해서 렌더링

### `main.html` — CDN 스크립트 로드 순서
```html
<link rel="stylesheet" href="https://maxcdn.bootstrapcdn.com/bootstrap/3.4.1/css/bootstrap.min.css">
<script src="https://ajax.googleapis.com/ajax/libs/jquery/3.7.1/jquery.min.js"></script>
<script src="https://maxcdn.bootstrapcdn.com/bootstrap/3.4.1/js/bootstrap.min.js"></script>
<script src="https://unpkg.com/vue@3.3.4/dist/vue.global.js"></script>
<script src="https://unpkg.com/axios/dist/axios.min.js"></script>
<script src="https://unpkg.com/vue-demi"></script>
<script src="https://unpkg.com/pinia@2.1.7/dist/pinia.iife.prod.js"></script>
```
- jQuery/Bootstrap(화면 스타일) → Vue3 → axios → vue-demi → Pinia 순서로 로드
- Pinia는 Vue2/Vue3를 모두 지원하려고 내부적으로 `vue-demi`에 의존함. 그래서 CDN 방식에서는 Pinia 스크립트보다 vue-demi를 먼저 로드해야 함. 프로젝트명이 "Pinia"인 것도 상태관리를 Pinia로 가져가겠다는 세팅에서 나온 이름임

---

## 4. FoodMapper — 어노테이션 + XML 혼용 패턴

```java
@Mapper
@Repository
public interface FoodMapper {
	public List<FoodVO> foodListData(int start);   // ← XML(food-mapper.xml)에서 SQL 정의

	@Select("SELECT CEIL(COUNT(*)/12.0) FROM food")
	public int foodTotalPage();                     // ← 어노테이션으로 직접 SQL

	@Select("SELECT * FROM food WHERE no=#{no}")
	public FoodVO foodDetailData(int no);

	@Update("UPDATE food SET hit=hit+1 WHERE no=#{no}")
	public void foodHitIncrement(int no);
}
```
- `foodListData`만 XML 매퍼(`food-mapper.xml`)에 SQL을 두고 나머지는 어노테이션으로 직접 SQL을 붙임
  → 인터페이스 상단 주석에 남긴 원칙 *"XML = SQL이 복잡한 경우, 단순한 SQL은 어노테이션"* 을 실천한 구성임
- 인터페이스 상단에는 `@Repository`/`@Service`/`@Controller`/`@RestController`/`@Component`/`@ControllerAdvice`/`@Configuration` 각각의 역할을 정리한 스프링 어노테이션 치트시트 주석도 함께 남김

### `food-mapper.xml` — 페이징 SQL (⚠ 오타 존재)
```xml
<select id="foodListData" resultType="com.sist.web.vo.FoodVO" parameterType="int">
   SELECT no,name,poster,address
   FROM food
   ORDER BY no ASC
   OFFFSET #{start} ROWS FETCH NEXT 12 ROWS ONLY
</select>
```
- Oracle 표준 키워드는 `OFFSET`(F 한 개)인데 `OFFFSET`으로 적혀 있음. 현재 상태로는 실행 시 **SQL 문법 오류**가 날 오타이므로 다음 작업 때 반드시 수정 필요
- `12`가 하드코딩되어 있어 페이지당 행 수를 바꾸려면 XML을 직접 수정해야 함. Day04의 "rowsize 매개변수화" 패턴은 아직 적용 전

---

## 5. Service 계층 — 아직 미완성 스텁 존재

```java
public interface FoodService {
	public List<FoodVO> foodListData(int start);
	public int foodTotalPage();
	public FoodVO foodDetailData(int no);
	public int[] foodPages(int page);
}
```
```java
@Override
public int[] foodPages(int page) {
	// TODO Auto-generated method stub
	return null;
}
```
- `foodListData`/`foodTotalPage`/`foodDetailData`는 Mapper로 위임(delegate)하도록 구현 완료
- `foodPages(int page)`는 아직 구현 전이라 `return null` 상태임. 페이지 번호 배열을 계산하는 메소드로 Day04에서 다룬 `pages[]` 패턴에 해당하며, 다음 커밋에서 채워질 부분

---

## 6. VO — 오늘 새로 정의된 4종 (Oracle 테이블 스키마 주석 포함)

| VO | 대응 테이블 | 주요 필드 | 비고 |
|----|------------|-----------|------|
| `FoodVO` | `FOOD` | no, cno, name, address, price, score, theme, content, poster, hit, likecount, jjimcount, replycount 등 | 클래스 상단에 **Oracle 컬럼 스펙(NOT NULL 여부 포함) 전체를 주석으로 남김** — 향후 `@Entity` 전환 시 참고용으로 보임 |
| `RecipeVO` | (레시피 목록용) | no, title, poster, chef, link, hit | Day05에서 다뤘던 레시피 목록 항목과 동일 구조 |
| `RecipeDetailVO` | (레시피 상세용) | no, poster, title, chef, chef_poster, chef_profile, info1~3, content, foodmake | Day05의 "RecipeDetail 분리" 패턴 재사용 |
| `ChefVO` | (쉐프 정보용) | chef, poster, mem_cont1~3, mem_cont7 | 오늘 신규 등장 — 아직 사용하는 Controller/Service 없음 |

- 모든 VO가 `jakarta.persistence.Entity`/`Id`를 import는 해두고 붙이지는 않은 상태임. 현재는 순수 MyBatis용 VO이고 추후 JPA Entity로 전환할 여지를 남겨둔 구조임

---

## 7. application.yml

```yaml
server:
  port: 9090
  servlet:
    context-path: /
spring:
  datasource:
    url: ${DB_URL}
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
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
- DB 계정정보는 Day07의 `${환경변수}` 원칙을 그대로 유지함. 실제 서버 IP/계정/비밀번호는 코드에도 노트에도 노출하지 않음
- `logging.level`에 `org.hibernate.orm.jdbc.bind: TRACE`까지 켜둬서 바인딩 파라미터 값까지 콘솔에서 확인 가능함. 개발 단계 디버깅용이라 운영에서는 레벨을 낮춰야 함
- 파일 하단에 `# security`, `# spring ai`, `# websocket = 카프카` 주석만 남기고 비워둠. 향후 추가 예정 기능 목록인 듯하며 오늘은 미구현

---

## 8. 비교표 — Day07 vs Day09 데이터 접근 전략

| 구분 | Day07 (Emp/Dept 실습) | Day09 (Food/Recipe/Chef 실전) |
|------|------------------------|-------------------------------|
| 주력 방식 | JPA 3방식(메소드규칙/JPQL/QueryDSL) 비교 학습 | **MyBatis 우선** (어노테이션+XML 혼용), JPA/QueryDSL은 의존성만 선언 |
| Entity | `@Entity` 붙은 실제 JPA 엔티티(Emp/Dept) | VO만 존재, `@Entity` 미부착 (JPA 전환 전 단계) |
| 화면 | 콘솔 출력 확인용(View 없음) | Thymeleaf + Vue3 + Pinia 실제 화면 렌더링 목적 |
| 목적 | 3가지 데이터 접근 방식의 **문법/개념 학습** | 실제 서비스(맛집/레시피) **화면까지 이어지는 풀스택 골격** 구축 |

---

## 9. 다시 만들 때 체크리스트

```text
[프로젝트 초기 세팅]
① settings.gradle에 rootProject.name 지정
② build.gradle에 JPA/MyBatis/QueryDSL/Lombok/Oracle 드라이버 한 번에 선언
③ .gitattributes로 gradlew(LF)/*.bat(CRLF) 줄바꿈 고정
④ Dockerfile EXPOSE 포트를 application.yml의 server.port(9090)와 일치시킬 것 (오늘은 8080으로 불일치 상태)

[화면 레이아웃]
⑤ RouterController에서 model.addAttribute("main_html", "경로")로 include 대상 스위칭
⑥ main.html에서 th:include로 header + main_html 조립
⑦ CDN 스크립트 순서: jquery → bootstrap.js → vue → axios → vue-demi → pinia (순서 지킬 것, pinia가 vue-demi에 의존)

[MyBatis]
⑧ 단순 SQL은 Mapper 인터페이스에 @Select/@Update로 직접 작성
⑨ 복잡한 SQL(페이징 등)은 XML 매퍼로 분리, mapper-locations 경로 확인
⑩ XML에 OFFSET 같은 키워드 오타 주의 (오늘 OFFFSET 오타 발견 → 다음에 수정)

[보안]
⑪ application.yml의 DB 계정정보는 반드시 ${DB_URL} 등 환경변수로 분리 (Day07 원칙 유지)
```

---

## 10. GitHub Actions 배포 — Repository Secrets로 DB 계정정보 감추기

### 세팅 경로
```text
리포지토리 → Settings → Secrets and variables → Actions
→ New repository secret → 이름/값 입력 (DB_URL, DB_USERNAME, DB_PASSWORD)
```
- `application.yml`에 이미 적용한 "${DB_URL} 등 환경변수로 분리" 원칙을 GitHub Actions 워크플로우 실행 시점까지 이어가는 세팅임
- 코드에도 워크플로우 파일에도 실제 값이 노출되지 않고 `secrets.DB_URL` 같은 참조로만 사용됨

### 워크플로우 안에서 `.env` 파일로 만들어 주입하는 단계
```yaml
      - run: |
         echo "======================="
         echo "환경변수 설정"
         echo "======================="
         rm -f .env
         touch .env
         echo "DB_URL=${{ secrets.DB_URL }}" >> .env
         echo "DB_USERNAME=${{ secrets.DB_USERNAME }}" >> .env
         echo "DB_PASSWORD=${{ secrets.DB_PASSWORD }}" >> .env
         echo ".env 저장 완료"
```
- self-hosted 러너(Ubuntu)에서 배포 직전에 실행되는 스텝
- `rm -f .env` → `touch .env`로 매번 새로 비운 뒤, Repository Secrets에 등록해둔 값을 `>>`로 한 줄씩 append하여 `.env` 파일을 새로 생성
- 이렇게 만들어진 `.env`는 이후 `docker run --env-file .env ...` 형태로 컨테이너에 주입됨. Dockerfile이나 이미지 안에는 DB 계정정보가 전혀 포함되지 않음
- Secrets는 워크플로우 실행 중에만 메모리상 치환됨. 실제 값이 로그나 워크플로우 파일에 그대로 찍히지 않도록 GitHub Actions가 마스킹 처리해줌

---

## 11. README 한 줄 요약 (표에 추가할 문구)

```text
| Day 09 | `SpringPiniaProject_2` 신규 프로젝트 초기 스캐폴딩(Food/Recipe/Chef 재시작), MyBatis 어노테이션+XML 혼용 패턴, main.html에 Vue3+axios+vue-demi+Pinia CDN 연동, Dockerfile/.gitattributes 추가, application.yml DB정보 환경변수 유지 | [📄 보기](./springboot_day09.md) |
```

---

