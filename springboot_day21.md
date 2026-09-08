# 📘 Spring Boot Day 21 — `SpringPostgreProject` 초기 세팅부터 Oracle↔PostgreSQL 이관 + AI 임베딩까지

## 0. 핵심 빠른 참조 — 이전 대비 바뀐 점

| 구분 | 이전 | 오늘 |
|------|------|------|
| DB 종류 | Oracle(`ojdbc11`) 중심으로 진행 | **PostgreSQL(`org.postgresql:postgresql`)** 신규 프로젝트 시작 |
| Git 작업 방식 | 로컬에서 바로 `main` 브랜치에 push | **main → develop → feature/기능명** 3단 브랜치 전략 + `gh pr create`/`merge`로 PR 경유 |
| 화면 렌더링 | — | Thymeleaf `th:each` + 인라인 표현식(`[[...]]`)으로 목록 출력 |
| DB 연동 범위 | PostgreSQL 단일 연결, `MemberMapper` 조회만 존재 | **Oracle + PostgreSQL 동시 연결**(다중 DataSource) + 패키지별 매퍼 분리 |
| 데이터 처리 | 없음(단순 목록 조회) | Oracle 레시피 데이터를 조회해 PostgreSQL로 이관하고, **Spring AI Embedding**으로 벡터화해 `pgvector`에 저장 |

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

## 4. 다중 데이터소스(Oracle + PostgreSQL) 동시 연결

### application.yml — datasource를 이름별로 분리 바인딩
```yaml
spring:
  datasource:
    postgres:
      jdbc-url: jdbc:postgresql://localhost:5432/recipe
      username: postgres
      password: "<DB계정>"
      driver-class-name: org.postgresql.Driver
    oracle:
      jdbc-url: jdbc:oracle:thin:@<서버IP>:1521:XE
      username: hr
      password: <DB계정>
      driver-class-name: oracle.jdbc.driver.OracleDriver
```
- 기존에는 `spring.datasource` 아래에 `url/username/...`을 바로 썼는데, 두 개의 DB를 동시에 쓰려면 `spring.datasource.postgres`, `spring.datasource.oracle`처럼 **커스텀 프로퍼티 그룹**으로 나눠야 함
- Spring Boot가 자동으로 만들어주는 `DataSource` 빈은 하나뿐이라, 이름을 분리한 프로퍼티는 `@ConfigurationProperties`로 직접 바인딩해야 함

### DataSourceConfig — DataSource 2개를 이름 붙여 Bean 등록
```java
@Configuration
public class DataSourceConfig {
	@Bean(name="oracleDataSource")
	@ConfigurationProperties(prefix = "spring.datasource.oracle")
	public DataSource oracleDataSource() {
		return DataSourceBuilder.create().build();
	}

	@Bean(name="postgresDataSource")
	@ConfigurationProperties(prefix = "spring.datasource.postgres")
	public DataSource postgresDataSource() {
		return DataSourceBuilder.create().build();
	}
}
```
- `@ConfigurationProperties(prefix=...)`로 yml의 각 그룹을 각각 다른 `DataSource` 객체로 바인딩
- `@Bean(name=...)`으로 이름을 지정해둬야 나중에 `@Qualifier`로 어느 DataSource인지 구분해서 주입 가능

---

## 5. MyBatis 이중 SqlSessionFactory 구성

```java
@Configuration
@MapperScan(basePackages = "com.sist.web.mapper.oracle",
            sqlSessionFactoryRef = "oracleSqlSessionFactory")
public class OracleMapperScanConfig {
}
```
```java
@Configuration
public class OracleMyBatisConfig {
	@Bean(name="oracleSqlSessionFactory")
	public SqlSessionFactory oracleSqlSessionFactory(
		@Qualifier("oracleDataSource") DataSource dataSource
	) throws Exception {
		SqlSessionFactoryBean factory = new SqlSessionFactoryBean();
		factory.setDataSource(dataSource);
		PathMatchingResourcePatternResolver resolver = new PathMatchingResourcePatternResolver();
		factory.setMapperLocations(resolver.getResources("classpath*:/mapper/oracle/*.xml"));
		return factory.getObject();
	}

	@Bean(name="oracleSessionTemplate")
	public SqlSessionTemplate oracleSessionTemplate(
		@Qualifier("oracleSqlSessionFactory") SqlSessionFactory sqlSessionFactory
	) {
		return new SqlSessionTemplate(sqlSessionFactory);
	}
}
```
- PostgreSQL 쪽도 `PostgresMapperScanConfig` + `PostgresMyBatisConfig`로 완전히 동일한 구조를 한 벌 더 만듦(빈 이름과 패키지만 `postgres`로 교체)
- 매퍼 인터페이스를 `mapper.oracle` / `mapper.postgres` 패키지로 물리적으로 분리하고 `@MapperScan`의 `sqlSessionFactoryRef`로 "이 패키지는 이 SqlSessionFactory를 쓴다"고 지정하는 방식
- `SqlSessionFactoryBean`에 `@Qualifier`로 지정한 `DataSource`를 주입하고 `mapperLocations`도 `mapper/oracle/*.xml` / `mapper/postgres/*.xml`로 나눠서 XML끼리 섞이지 않게 함
- `application.yml`의 `mybatis.mapper-locations`도 리스트 형태로 두 경로를 모두 등록해둬야 함
  ```yaml
  mybatis:
    type-aliases-package: com.sist.web.vo
    mapper-locations:
      - classpath:/mapper/oracle/*.xml
      - classpath:/mapper/postgres/*.xml
    configuration:
      map-underscore-to-camel-case: true
  ```

---

## 6. Oracle → PostgreSQL 레시피 데이터 이관

```java
@Mapper
@Repository
public interface OracleRecipeMapper {
	public List<RecipeVO> oracleRecipeAllData();
}
```
```xml
<mapper namespace="com.sist.web.mapper.oracle.OracleRecipeMapper">
  <select id="oracleRecipeAllData" resultType="com.sist.web.vo.RecipeVO">
    SELECT * FROM recipe ORDER BY rcp_seq
  </select>
</mapper>
```
```java
@Mapper
@Repository
public interface PostgresRecipeMapper {
	public void postgresRecipeInsert(RecipeVO vo);
	public void recipeVectorInsert(RecipeVectorVO vo);
}
```
```java
@Service
@RequiredArgsConstructor
public class RecipeService {
	private final OracleRecipeMapper oMapper;
	private final PostgresRecipeMapper pMapper;

	public void recipeInsert() {
		List<RecipeVO> list = oMapper.oracleRecipeAllData();
		for (RecipeVO vo : list) {
			pMapper.postgresRecipeInsert(vo);
		}
	}
}
```
```java
@RestController
@RequiredArgsConstructor
public class RecipeController {
	private final RecipeService rs;
	private final RecipeVectorService ss;

	@GetMapping("/recipe")
	public String recipe_insert() {
		rs.recipeInsert();
		return "데이터 저장 완료";
	}
}
```
- 기존 `MemberMapper`(단일 `@Select` 조회)는 `RecipeMapper`로 이름을 바꿨다가, Oracle/PostgreSQL 이중 매퍼 구조로 옮기면서 실질적으로 `OracleRecipeMapper` / `PostgresRecipeMapper`로 대체됨
- `RecipeService`는 Oracle에서 전체 목록을 조회한 뒤 for문으로 하나씩 PostgreSQL에 INSERT — DB 간 데이터 이관을 서비스 계층 코드로 직접 구현한 형태
- Controller가 `@RestController`로 바뀌어서 `return` 값이 화면 대신 문자열 그대로 응답 바디로 나감

---

## 7. Spring AI Embedding으로 레시피 벡터화 + pgvector 저장

### build.gradle
```gradle
implementation 'org.springframework.ai:spring-ai-starter-model-google-genai-embedding:2.0.1'
runtimeOnly 'com.oracle.database.jdbc:ojdbc17'
```

### application.yml
```yaml
spring:
  ai:
    google:
      genai:
        api-key: ${GEN_KEY}
        embedding:
          api-key: ${GEN_KEY}
          text:
            options:
              model: gemini-embedding-001
              dimensions: 768
    model:
      embedding:
        text: google-genai
```
- API 키는 `${GEN_KEY}` 환경변수로만 참조하고 yml에는 직접 값을 쓰지 않음
- `dimensions: 768`처럼 임베딩 벡터의 차원 수를 모델 옵션으로 지정

### RecipeVectorService — 텍스트 생성 → 임베딩 → pgvector 문자열 변환
```java
@Service
@RequiredArgsConstructor
public class RecipeVectorService {
	private final PostgresRecipeMapper pMapper;
	private final OracleRecipeMapper oMapper;
	private final EmbeddingModel model;

	public void recipeVectorInsert() {
		List<RecipeVO> list = oMapper.oracleRecipeAllData();
		for (RecipeVO recipe : list) {
			String content = createContent(recipe);      // 검색용 문서 생성
			float[] vector = model.embed(content);        // Spring AI Embedding 호출
			String embedding = convertVector(vector);      // float[] → pgvector 문자열

			RecipeVectorVO vo = RecipeVectorVO.builder()
					.recipe_id((long) recipe.getRcp_seq())
					.content(content)
					.embedding(embedding)
					.build();

			pMapper.recipeVectorInsert(vo);
		}
	}

	private String createContent(RecipeVO vo) {
		return """
				레시피명: %s
				조리방법: %s
				요리종류: %s
				영양정보: %s kcal
				...
				""".formatted(vo.getRcp_nm(), vo.getRcp_way2(), vo.getRcp_pat2(), vo.getInfo_eng());
	}

	private String convertVector(float[] vector) {
		StringBuilder sb = new StringBuilder("[");
		for (int i = 0; i < vector.length; i++) {
			if (i > 0) sb.append(",");
			sb.append(vector[i]);
		}
		sb.append("]");
		return sb.toString();
	}
}
```
```xml
<insert id="recipeVectorInsert" parameterType="com.sist.web.vo.RecipeVectorVO">
  INSERT INTO recipe_vector(recipe_id, content, embedding)
  VALUES(#{recipe_id}, #{content}, #{embedding}::vector)
</insert>
```
- `EmbeddingModel.embed(String)`이 `float[]`를 반환 — Spring AI가 실제 벡터 생성 API 호출을 감싸주는 인터페이스임
- 레시피의 여러 필드(이름/조리법/영양정보 등)를 하나의 텍스트 블록(`createContent`)으로 합쳐서 임베딩 입력으로 사용 — 임베딩은 "하나의 문자열"을 받는 구조라 여러 컬럼을 사람이 읽는 문장 형태로 합치는 전처리가 필요함
- PostgreSQL의 `pgvector` 확장은 벡터를 `[0.1,0.2,...]` 형태의 문자열로 받아 `::vector`로 캐스팅해야 저장됨 — `float[]`를 그 문자열 포맷으로 직접 변환하는 `convertVector`가 필요한 이유
- `RecipeVO`(원본 데이터) / `RecipeVectorVO`(id-content-embedding) 두 VO로 "원본 테이블"과 "벡터 테이블"을 분리 설계

---

## 8. 비교표 — Git 작업 흐름 (이전 방식 vs 오늘 방식)

| 구분 | 이전 | 오늘 |
|------|------|------|
| 브랜치 구성 | `main` 단일 브랜치에서 바로 작업 | `main`/`develop`/`feature/기능명` 3단 구성 |
| 커밋 반영 방식 | `git push`로 `main`에 직접 반영 | `feature`에서 커밋 → PR 생성 → `develop`으로 병합 |
| PR 도구 | 사용 안 함 | `gh pr create` / `gh pr merge`(GitHub CLI) |
| 목적 | 빠른 반영 | 기능 단위 격리 + 리뷰 경유 후 통합 |
| DB 구성 | PostgreSQL 단일 DataSource | Oracle + PostgreSQL **다중 DataSource**(`@Qualifier`+이중 SqlSessionFactory) |
| 데이터 흐름 | 없음 | Oracle 조회 → PostgreSQL 이관 → Spring AI Embedding으로 벡터화 → `pgvector` 저장 |

---

## 9. 다시 만들 때 체크리스트

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

[다중 DataSource(Oracle + PostgreSQL)]
⑩ application.yml에서 spring.datasource.오라클/spring.datasource.postgres처럼 이름별로 그룹 분리
⑪ DataSourceConfig에서 @ConfigurationProperties(prefix=...) + @Bean(name=...)으로 DataSource 2개 등록
⑫ 매퍼 패키지를 mapper.oracle / mapper.postgres로 물리 분리 후 @MapperScan(sqlSessionFactoryRef=...)로 연결
⑬ SqlSessionFactoryBean에는 @Qualifier로 지정한 DataSource 주입 + mapperLocations도 DB별 경로로 분리
⑭ mybatis.mapper-locations를 리스트로 오라클/포스트그레 경로 둘 다 등록

[Spring AI Embedding + pgvector]
⑮ build.gradle에 spring-ai-starter-model-google-genai-embedding 추가, API 키는 ${환경변수}로만 참조
⑯ 여러 컬럼을 합친 문장(createContent)을 만들어 EmbeddingModel.embed()에 전달 → float[] 반환
⑰ float[]를 "[0.1,0.2,...]" 문자열로 변환해 INSERT 시 #{embedding}::vector로 캐스팅
⑱ 원본 테이블 VO와 벡터 테이블 VO(id/content/embedding)를 분리 설계
```
