# 📘 Spring Boot Day 21 — `SpringPostgreProject` 초기 세팅부터 Oracle↔PostgreSQL 이관 + AI 임베딩까지, `SpringRecipeAIProject` Jenkins-Docker 배포 파이프라인

## 0. 핵심 빠른 참조 — 이전 대비 바뀐 점

| 구분 | 이전 | 오늘 |
|------|------|------|
| DB 종류 | Oracle(`ojdbc11`) 중심으로 진행 | **PostgreSQL(`org.postgresql:postgresql`)** 신규 프로젝트 시작 |
| Git 작업 방식 | 로컬에서 바로 `main` 브랜치에 push | **main → develop → feature/기능명** 3단 브랜치 전략 + `gh pr create`/`merge`로 PR 경유 |
| 화면 렌더링 | — | Thymeleaf `th:each` + 인라인 표현식(`[[...]]`)으로 목록 출력 |
| DB 연동 범위 | PostgreSQL 단일 연결, `MemberMapper` 조회만 존재 | **Oracle + PostgreSQL 동시 연결**(다중 DataSource) + 패키지별 매퍼 분리 |
| 데이터 처리 | 없음(단순 목록 조회) | Oracle 레시피 데이터를 조회해 PostgreSQL로 이관하고, **Spring AI Embedding**으로 벡터화해 `pgvector`에 저장 |
| Jenkins 파이프라인(`SpringRecipeAIProject`) | Git 연결 확인만 하는 최소 파이프라인 | 빌드 → Docker 이미지 빌드/푸시 → 컨테이너 재배포까지 포함한 전체 배포 파이프라인 |
| 레시피 추천 설계 | 없음 | 재료 입력 → 임베딩 검색 → pgvector 유사도 검색 → 재료 충족률 계산까지 전체 흐름 설계 |

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

## 8. `SpringRecipeAIProject` — Jenkins 파이프라인으로 빌드~Docker 배포 자동화

### Jenkinsfile 전체 단계
```groovy
pipeline {
    agent any
    environment {
        APP_DIR = "~/app"
        JAR_NAME = "SpringRecipeAIProject-0.0.1-SNAPSHOT.jar"
    }
    stages {
        stage('Check Out') {
            steps { checkout scm }
        }

        stage('Create .env') {
            steps {
                withCredentials([
                    string(credentialsId: 'post-url', variable: 'POST_URL'),
                    string(credentialsId: 'gen-key', variable: 'GEN_KEY')
                ]) {
                    sh '''
                        echo "SPRING_PROFILES_ACTIVE=prod" > .env
                        echo "POST_URL=${POST_URL}" >> .env
                        echo "GEN_KEY=${GEN_KEY}" >> .env
                        chmod 600 .env
                    '''
                }
            }
        }

        stage('Gradlew Build') {
            steps { sh './gradlew clean build -x test' }
        }

        stage('Docker Build') {
            steps { sh 'docker build -t <도커계정>/ai-app:latest .' }
        }

        stage('DockerHub Login') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub_info',
                    usernameVariable: 'DH_USER',
                    passwordVariable: 'DH_PASS'
                )]) {
                    sh 'echo "$DH_PASS" | docker login -u "$DH_USER" --password-stdin'
                }
            }
        }

        stage('Docker Push') {
            steps { sh 'docker push <도커계정>/ai-app:latest' }
        }

        stage('Container Stop')   { steps { sh 'docker stop ai-app || true' } }
        stage('Container Remove') { steps { sh 'docker rm ai-app || true' } }
        stage('DockerHub Pull')   { steps { sh 'docker pull <도커계정>/ai-app:latest' } }

        stage('Docker Run') {
            steps {
                sh 'docker run -d --name ai-app -p 9090:9090 --env-file .env <도커계정>/ai-app:latest'
            }
        }
    }
}
```
- 기존 Jenkinsfile(Git 연결 확인만 하는 최소 파이프라인)에서 한 단계 나아가, 빌드부터 Docker 배포까지 전체 흐름을 파이프라인 하나에 담음
- `environment` 블록은 파이프라인 전체에서 공통으로 쓰는 변수(앱 경로, jar 이름)를 선언하는 용도

### `withCredentials` 두 가지 바인딩 타입
- `string(credentialsId:, variable:)` — 토큰/URL처럼 값 하나를 변수 하나로 바인딩(`POST_URL`, `GEN_KEY`)
- `usernamePassword(credentialsId:, usernameVariable:, passwordVariable:)` — 아이디/비밀번호 쌍을 변수 두 개로 한 번에 바인딩(DockerHub 로그인)
- 두 경우 모두 Jenkins Credentials Store에 미리 등록해둔 값을 블록 안에서만 환경변수로 꺼내 쓰고, 블록 밖으로는 노출되지 않음

### `.env` 파일을 파이프라인에서 생성해 컨테이너에 주입
- `echo ... > .env` / `echo ... >> .env`로 파이프라인이 빌드 시점에 직접 `.env` 파일을 생성
- `chmod 600 .env`로 소유자만 읽고 쓸 수 있게 권한을 제한 — 소스코드·이미지 안에는 민감정보를 전혀 남기지 않음
- `docker run --env-file .env ...`로 컨테이너 실행 시점에 환경변수를 주입하는 방식 — Spring 쪽은 `application.yml`에서 `${POST_URL}`처럼 환경변수를 참조해 이 값을 읽음

### Docker 로그인 — 비밀번호를 커맨드라인 인자로 남기지 않는 방식
```bash
echo "$DH_PASS" | docker login -u "$DH_USER" --password-stdin
```
- `docker login -u ... -p ...`처럼 비밀번호를 인자로 바로 넘기면 프로세스 목록·로그에 노출될 수 있어, 표준입력(stdin)으로 전달받는 `--password-stdin` 방식을 사용

### 컨테이너 재배포 패턴 — stop → rm → pull → run
- 기존 컨테이너를 내리고(`stop`) 지운 뒤(`rm`) 최신 이미지를 받아(`pull`) 새로 띄우는(`run`) 4단계 고정 흐름
- `stop`/`rm` 뒤에 `|| true`를 붙여, 컨테이너가 아직 없는 최초 배포 시에도 해당 단계가 실패로 파이프라인을 멈추지 않게 함

### `application.yml` — DB URL 하드코딩 → 환경변수 참조로 전환
```yaml
spring:
  datasource:
    url: ${POST_URL}
    username: postgres
    password: <DB계정>
    driver-class-name: org.postgresql.Driver
```
- 기존에는 `url: jdbc:postgresql://localhost:5432/recipe`처럼 접속 주소를 코드에 직접 박아뒀는데, `.env`로 주입되는 `${POST_URL}` 환경변수를 참조하도록 변경
- 운영 서버 주소가 바뀌어도 코드 수정 없이 Jenkins Credentials 값만 바꾸면 되는 구조

---

## 9. 재료 기반 레시피 추천 — 전체 흐름 설계

```text
<브라우저>(HTML/바닐라 JS)
  | 재료 선택 → POST 전송
RecipeController (@RestController)
  | ingredients 전달
RecipeService
  1) 재료 존재 여부 확인
  2) 검색 문장 생성
  3) EmbeddingModel로 검색 문장을 벡터(float[])로 변환
  4) PostgreSQL + pgvector로 유사 레시피 검색
  5) 검색된 레시피에서 content 추출
  6) 냉장고 보유 재료 ↔ 레시피 재료 비교
  7) 재료 상태 판정(부족 / 전체 만족) + 충족률 계산
  → 충족률 기준 상위 5개 추천 리스트 반환
```
- 앞서 구현한 "Oracle→PostgreSQL 이관 + 임베딩 저장" 파이프라인(1~7절)에 이어, 저장된 벡터를 실제로 "검색"에 활용하는 쪽 설계를 정리한 단계
- 검색 쿼리도 레시피 저장 때와 동일하게 "문장 생성 → `EmbeddingModel.embed()`로 벡터화" 과정을 거쳐야 pgvector 유사도 비교가 가능
- 응답을 화면 렌더링이 아니라 데이터로 내려줘야 해서 Controller를 `@RestController`로 전환할 필요가 있음(기존 `@Controller` + `@ResponseBody` 방식에서 한 단계 더 나아간 형태)

---

## 10. 비교표 — Git 작업 흐름 (이전 방식 vs 오늘 방식)

| 구분 | 이전 | 오늘 |
|------|------|------|
| 브랜치 구성 | `main` 단일 브랜치에서 바로 작업 | `main`/`develop`/`feature/기능명` 3단 구성 |
| 커밋 반영 방식 | `git push`로 `main`에 직접 반영 | `feature`에서 커밋 → PR 생성 → `develop`으로 병합 |
| PR 도구 | 사용 안 함 | `gh pr create` / `gh pr merge`(GitHub CLI) |
| 목적 | 빠른 반영 | 기능 단위 격리 + 리뷰 경유 후 통합 |
| DB 구성 | PostgreSQL 단일 DataSource | Oracle + PostgreSQL **다중 DataSource**(`@Qualifier`+이중 SqlSessionFactory) |
| 데이터 흐름 | 없음 | Oracle 조회 → PostgreSQL 이관 → Spring AI Embedding으로 벡터화 → `pgvector` 저장 |
| Jenkins 파이프라인 범위 | Git 연결 확인만 하는 최소 파이프라인 | 빌드 → Docker 이미지 빌드/푸시 → 컨테이너 재배포까지 전체 배포 파이프라인 |
| 민감정보 전달 방식 | 파이프라인에서 다루지 않음 | Jenkins Credentials → `.env` 파일 생성(`chmod 600`) → `docker run --env-file`로 컨테이너에 주입 |

---

## 11. 다시 만들 때 체크리스트

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

[Jenkins CI/CD — 빌드~Docker 배포]
⑲ withCredentials(string / usernamePassword)로 Jenkins Credentials Store 값을 파이프라인 환경변수로 바인딩
⑳ .env 파일은 파이프라인에서 생성 후 chmod 600으로 권한 제한, 소스코드·이미지에는 민감정보 넣지 않음
㉑ docker login은 echo "$비밀번호" | docker login --password-stdin 방식으로 커맨드라인 인자 노출 방지
㉒ 컨테이너 재배포는 stop → rm → pull → run 순서(stop/rm은 || true로 최초 배포 시 실패 무시)
㉓ application.yml은 하드코딩 값 대신 ${환경변수}로 참조해 .env 주입값을 읽도록 구성

[재료 기반 레시피 추천 흐름 설계]
㉔ 검색도 저장과 동일하게 "문장 생성 → EmbeddingModel.embed()로 벡터화" 과정을 거쳐야 pgvector 유사도 비교 가능
㉕ 재료 선택 → 임베딩 검색 → pgvector 유사도 검색 → 냉장고 재료와 비교해 충족률 계산 → 상위 N개 추천 순서로 설계
```
