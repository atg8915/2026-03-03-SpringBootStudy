# 📘 Spring Boot Day 11 — Spring Security 인증 완성 + 인증 방식 비교 + 람다식/Stream 학습

## 0. 핵심 빠른 참조 — Day10 대비 오늘 바뀐 점 (`SpringPiniaProject_2`)

| 구분 | Day10까지 | Day11(오늘) |
|------|-----------|--------------|
| `AuthenticationManager` | 미구현 | `AuthenticationManagerBuilder`로 `jdbcUserDetailsService()` + `passwordEncoder()` 연결해서 구현 완료 |
| `JdbcUserDetailsManager` | 미구현 | `dataSource` 기반 생성 + `users-by-username-query`/`authorities-by-username-query` SQL 직접 세팅 |
| `PersistentTokenRepository` | 미구현 | `JdbcTokenRepositoryImpl`에 `dataSource` 연결, `rememberMe().tokenRepository()`에도 연결 |
| `LoginFailHandler` | 빈 스텁 | 예외 타입별(`BadCredentialsException` 등) 에러 메시지 분기 + `/member/login`으로 forward |
| `LoginSuccessHandler` | 빈 스텁 | `response.sendRedirect("/")` 구현 |
| `login.html` | `<form>` 없이 정적 마크업만 존재 | `<form action="/member/login_process" method="post">` 추가로 실제 로그인 제출 가능 |
| `header.html` | `sec:authorize=""` 조건 미완성 | 로그인 상태/권한별(`isAuthenticated()`, `hasRole('ADMIN')` 등) 메뉴 분기 완성 |
| 배포 | `docker run --env-file .env`로 컨테이너 직접 실행 | `docker-compose.yml` 도입 + Docker Hub push 단계 추가된 배포 파이프라인으로 전환 |

---

## 1. SpringPiniaProject_2 — Spring Security 인증 완성

### 1-1. AuthenticationManager / JdbcUserDetailsManager — DB 연동 완성

```java
@Bean
public AuthenticationManager authenticationManager(
   HttpSecurity http,
   BCryptPasswordEncoder passwordEncoder
) throws Exception
{
   AuthenticationManagerBuilder builder=
         http.getSharedObject(AuthenticationManagerBuilder.class);
   builder
     .userDetailsService(jdbcUserDetailsService())
     .passwordEncoder(passwordEncoder());
   return builder.build();
}
@Bean
public JdbcUserDetailsManager jdbcUserDetailsService() {
   JdbcUserDetailsManager manager=
         new JdbcUserDetailsManager(dataSource);
   manager.setUsersByUsernameQuery(
         "SELECT userid as username,userpwd as password,enable "
         +"FROM springmember WHERE userid=?"
   );
   manager.setAuthoritiesByUsernameQuery(
         "SELECT userid as username , authority "
        +"FROM authority WHERE userid=?"
   );
   return manager;
}
@Bean
public PersistentTokenRepository persistentTokenRepository() {
    JdbcTokenRepositoryImpl repo = new JdbcTokenRepositoryImpl();
    repo.setDataSource(dataSource);
    return repo;
}
```
- 핵심은 `AuthenticationManagerBuilder`를 `http.getSharedObject()`로 꺼내서 `userDetailsService()`/`passwordEncoder()`를 연결함. `new`로 직접 만들지 않고 Spring Security가 제공하는 Builder를 거쳐 구성함
- `JdbcUserDetailsManager`의 쿼리 컬럼명은 `MemberVO`(`springmember` 테이블 추정)/`AuthorityVO`(`authority` 테이블) 구조와 일치
- `rememberMe()`에도 `.tokenRepository(persistentTokenRepository())`를 연결해서 remember-me 토큰이 DB에 저장되도록 구성함

### 1-2. LoginFailHandler / LoginSuccessHandler — 실제 로직 구현

```java
// LoginFailHandler
if(exception instanceof BadCredentialsException)
{
   errorMsg="아이디나 비밀번호가 틀립니다!!";
}
else if(exception instanceof InternalAuthenticationServiceException)
{
   errorMsg="아이디나 비밀번호가 틀립니다!!";
}
else if(exception instanceof DisabledException)
{
   errorMsg="휴먼 계정입니다!!";
}
...
request.setAttribute("message", errorMsg);
request.getRequestDispatcher("/member/login").forward(request, response);
```
```java
// LoginSuccessHandler
public void onAuthenticationSuccess(HttpServletRequest request, HttpServletResponse response,
      Authentication authentication) throws IOException, ServletException {
   // TODO Auto-generated method stub
   response.sendRedirect("/");
}
```
- 예외 타입별로 분기해서 사용자에게 보여줄 메시지를 다르게 세팅함. `request.setAttribute("message", ...)` 후 `forward`로 로그인 화면에 그대로 표시(Thymeleaf `[[${message}]]`로 출력)
- 인증 성공 시에는 `sendRedirect`로 단순 리다이렉트만 수행

### 1-3. header.html / login.html — 화면에서 로그인 상태 반영

```html
<li sec:authorize="!isAuthenticated()"><a href="#">회원가입</a></li>
<li sec:authorize="!isAuthenticated()"><a href="/member/login">로그인</a></li>
<li sec:authorize="isAuthenticated()">
<li sec:authorize="hasRole('ADMIN')">
  <a href="#"><span sec:authentication="name"></span>(관리자) 로그인되었습니다</a>
</li>
<li sec:authorize="hasRole('USER')">
  <a href="#"><span sec:authentication="name"></span>(일반사용자) 로그인되었습니다</a>
</li>
<li sec:authorize="isAuthenticated()">
  <a href="/member/logout">로그아웃</a>
</li>
```
- `sec:authorize` 조건이 채워져 로그인 여부/권한별로 메뉴를 분기함
- 다만 `<li sec:authorize="isAuthenticated()">`로 감싸놓고 그 안에 다시 `<li sec:authorize="hasRole('ADMIN')">`를 중첩한 탓에, `<li>` 태그가 제대로 닫히지 않은 채 다음 `<li>`들이 이어짐. 동작은 하지만 마크업 정리는 필요함
- `login.html`에는 `<form action="/member/login_process" method="post">`가 추가되어 로그인이 실제로 제출됨. remember-me 체크박스와 에러 메시지 출력 영역(`[[${message}]]`)도 함께 추가됨

### 1-4. Docker Compose + GitHub Actions 배포 파이프라인 재정비

- 이전에는 `docker run --name food-app -d --env-file .env -p 9090:9090 food-app` 형태로 컨테이너를 직접 실행
- `docker-compose.yml`을 도입해 `docker login → docker push → docker compose down/pull/up -d` 파이프라인으로 전환함

```yaml
version: "3.8"
services:
  app:
   build: .
   image: <DockerHub계정>/pinia-app
   container_name: pinia-app
   ports:
     - "9090:9090"
   restart: always
   env_file:
         - .env
```

```yaml
      - run: docker build -t pinia-app .
      - run: docker tag pinia-app ${{secrets.DOCKER_USERNAME}}/pinia-app:latest
      - run: echo "${{secrets.DOCKER_PASSWORD}}" | docker login -u "${{secrets.DOCKER_USERNAME}}" --password-stdin
      - run: docker push ${{secrets.DOCKER_USERNAME}}/pinia-app
      - run: docker compose down
      - run: docker compose pull
      - run: docker compose up -d
```
- 이미지 태그에 Docker Hub 계정명을 붙이는 규칙(`계정명/이미지명`) 확인. `docker tag`로 태깅 단계를 build와 분리해 두는 법도 학습함
- `docker login`은 `echo "${{secrets.DOCKER_PASSWORD}}" | docker login -u ... --password-stdin` 형태로 비밀번호를 표준입력으로 전달함. 이러면 비밀번호가 명령행 인자로 노출되지 않음
- `docker login`/`docker push`용 계정정보는 `${{secrets.DOCKER_USERNAME}}`/`${{secrets.DOCKER_PASSWORD}}`로 GitHub Actions Secrets 참조. 민감정보는 환경변수/Secrets로 분리하는 원칙 그대로 유지됨

---

## 2. SpringSecurityProject_2 — DB 연동 CustomUserDetailsService 신규 실습

`JdbcUserDetailsManager`(쿼리를 문자열로 세팅)가 아니라 직접 만든 `UserDetailsService` 구현체로 DB를 연동함.

```java
@Service
@RequiredArgsConstructor
public class CustomUserDetailsService implements UserDetailsService{
	private final UserMapper mapper;

	@Override
	public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
		MemberVO user=mapper.findByUserid(username);
		if(user==null)
		{
			throw new UsernameNotFoundException("UserName을 찾을 수 없습니다");
		}
		List<String> roles=mapper.findRolesByUserid(username);
		Set<GrantedAuthority> authorities=new HashSet<>();
		for(String role:roles)
		{
			authorities.add(new SimpleGrantedAuthority(role));
		}
		return new User(user.getUsername(),user.getUserpwd(),user.getEnable()==0?false:true,true , true , true , authorities);
	}
}
```
```java
@Mapper
@Repository
public interface UserMapper {
  @Select("SELECT userid as username,userpwd,enable "
  		+ "FROM springmember WHERE userid=#{userid}")
  public MemberVO findByUserid(String userid);
  @Select("SELECT authority FROM authority WHERE userid=#{userid}")
  public List<String> findRolesByUserid(String userid);
}
```
- MyBatis `@Select` 어노테이션으로 SQL을 직접 작성하고 `UserDetailsService`를 구현해서 조회 결과를 Spring Security의 `User` 객체로 변환함. 여기서는 조회 로직을 완전히 직접 제어함. `SpringPiniaProject_2`의 `JdbcUserDetailsManager`는 내장 쿼리 프로퍼티로 SQL을 넘기면 끝이었음
- `SecurityConfig`에서 `LoginFailHandler` Bean을 등록해놓고도 `formLogin()`에는 `.failureHandler(null)`을 넘겨서 실제로는 연결이 안 된 상태(버그)임. `.failureHandler(loginFailHandler())`로 고쳐야 정상 동작

```java
.formLogin(form->form
		.loginPage("/login")
		.loginProcessingUrl("/login_process")
		.defaultSuccessUrl("/",true)
		.failureHandler(null)
)
...
@Bean
public AuthenticationFailureHandler loginFailHandler()
{
	return new LoginFailHandler();
}
```
- `application.yml`의 DB 접속 정보는 `${DB_URL}`/`${DB_USERNAME}`/`${DB_PASSWORD}` 환경변수 참조로 작성함. 마스킹 원칙을 신규 프로젝트에도 그대로 적용함
- `docker-compose.yml`에는 `env_file: .env`만 추가된 상태고 `Dockerfile`은 아직 없음. 배포 설정은 다음 작업으로 이어질 예정

---

## 3. SpringSecurityProject_1 — 인메모리(하드코딩) UserDetailsService 실습

DB 연동 없이 하드코딩된 2명의 사용자(admin/user)로 Security 최소 구조만 익히는 실습.

```java
@Service
public class CustomUserDetailService implements UserDetailsService{
	@Override
	public UserDetails loadUserByUsername(String username)
			throws UsernameNotFoundException {
		if (username.equals("admin")) {
			return User.builder()
					.username("admin")
					.password("{noop}1234")
					.roles("ADMIN")
					.build();
		}
		return User.builder()
				.username("user")
				.password("{noop}1234")
				.roles("USER")
				.build();
	}
}
```
- `{noop}1234`는 Spring Security의 비밀번호 인코딩 접두사임. `{noop}`은 암호화 없음을 의미함. `BCryptPasswordEncoder` 없이 평문으로 비교하게 해주는 학습용 트릭
- `id`가 `admin`이 아니면 무조건 `user`(권한 `USER`)로 처리되므로, 실제로는 어떤 아이디를 넣어도 로그인됨. 비밀번호만 `1234`면 통과함. DB 검증이 없는 순수 실습용 코드
- `formLogin()`의 `loginProcessingUrl`을 `loginPage`와 동일하게 지정함. Spring Security가 해당 URL의 POST를 Controller 대신 직접 가로채 처리함

```java
.formLogin(form->form
	 .loginPage("/login")
	 .loginProcessingUrl("/login")
	 .defaultSuccessUrl("/",true)
	 .failureUrl("/login?error")
	 .permitAll()
)
```
- `failureHandler` 대신 `.failureUrl("/login?error")` 사용. 핸들러 클래스 없이 URL 리다이렉트만으로 실패를 처리해서 더 단순함

---

## 4. LamdaProject — 람다식 / Stream 문법 신규 학습

Spring Boot 프로젝트가 아닌 순수 Java(STS/Eclipse) 프로젝트로, 람다식과 Stream API를 정리.

```java
// 익명 클래스 방식 vs 람다식
/* Runnable r=new Runnable() {
	@Override
	public void run() {
		System.out.println("Thread 실행!!");
	}
};*/
Runnable r=()->System.out.println("Thread 실행!!");
new Thread(r).start();
```
```java
@FunctionalInterface
interface Calc{
	int sum(int a, int b);
}
...
Calc c=(a,b)->a+b;
System.out.println(c.sum(10, 20));
```
- 익명 클래스 → 람다식 순서로 Before/After를 비교하며 학습
- 추상 메서드가 1개뿐인 인터페이스(`@FunctionalInterface`)만 람다식으로 대체된다는 제약 확인
- Stream의 `forEach`/`filter`/`map`/`sorted`/`distinct`/`reduce`/`average`를 `List.of(...)` 데이터로 실습
- `EmpDAO`(`com.sist.stream` 패키지)는 Oracle에 순수 JDBC로 직접 접속해서 `emp` 테이블을 조회하는 예제:

```java
private final String URL="jdbc:oracle:thin:@localhost:1521:XE";
...
conn=DriverManager.getConnection(URL,"<DB계정>","<DB계정>");
```
- Spring Boot 프로젝트들은 `${DB_URL}` 환경변수를 씀. 이 프로젝트는 스트림 문법 자체가 학습 목적이라 접속 정보를 하드코딩 그대로 사용 중. 실제 값이 코드에 남아있다는 점은 유의
- 스트림으로 회원 목록을 필터링하고 권한별로 분류하는 예시(`filter(u -> "ADMIN".equals(u.getRole()))`, `map(Member::getName)`)도 함께 정리함. 오늘 다룬 Security 실습과 연결지어 활용

---

## 5. 비교표 — 오늘 다룬 3가지 인증(UserDetailsService) 구현 방식

| 구분 | `SpringPiniaProject_2` | `SpringSecurityProject_2` | `SpringSecurityProject_1` |
|------|------------------------|----------------------------|------------------------------|
| 구현 방식 | `JdbcUserDetailsManager` (내장 클래스 + SQL 문자열 프로퍼티) | 직접 만든 `UserDetailsService` 구현체 + MyBatis `@Select` | 직접 만든 `UserDetailsService` 구현체 + 하드코딩(DB 없음) |
| 사용자/권한 조회 | `setUsersByUsernameQuery`/`setAuthoritiesByUsernameQuery`에 SQL을 그대로 전달 | `UserMapper`로 SQL 실행 후 결과를 `Set<GrantedAuthority>`로 변환 | `if(username.equals("admin"))` 분기로 고정된 2개 계정만 반환 |
| 비밀번호 처리 | `BCryptPasswordEncoder` 실제 암호화 | `BCryptPasswordEncoder` Bean은 등록돼 있으나 이 흐름에선 미사용 | `{noop}1234` — 암호화 없이 평문 비교 |
| 로그인 실패 처리 | `LoginFailHandler` 구현체 + `AuthenticationFailureHandler` 정상 연결 | `LoginFailHandler` 구현체는 있으나 `.failureHandler(null)`로 연결 누락(버그) | 핸들러 없이 `.failureUrl("/login?error")`만 사용 |
| 자동 로그인(remember-me) | `PersistentTokenRepository`(DB 저장) 완성 | 미구현 | 미구현 |
| 학습 목적 | 실제 서비스 프로젝트의 정식 구현 | DB 연동 인증을 처음부터 직접 만들어보는 연습 | Security 최소 구조(권한/로그인/로그아웃)만 빠르게 익히는 연습 |

---

## 6. 다시 만들 때 체크리스트

```text
[SpringPiniaProject_2 — 인증 Bean 완성]
① AuthenticationManagerBuilder를 http.getSharedObject(AuthenticationManagerBuilder.class)로 꺼내서
   userDetailsService()/passwordEncoder() 연결 → build()
② JdbcUserDetailsManager(dataSource) 생성 후 setUsersByUsernameQuery/setAuthoritiesByUsernameQuery로
   회원/권한 테이블 SQL 직접 세팅
③ PersistentTokenRepository는 JdbcTokenRepositoryImpl(dataSource) + rememberMe().tokenRepository()로 연결
④ LoginFailHandler에서 예외 타입(BadCredentialsException/DisabledException 등)별 분기 후
   request.setAttribute+forward로 에러 메시지 화면 전달
⑤ login.html에 <form action method="post"> 반드시 추가해야 실제 제출 가능

[SpringSecurityProject_2 — DB 연동 직접 구현]
⑥ UserDetailsService 구현체에서 MyBatis Mapper로 회원/권한 조회 → Set<GrantedAuthority>로 변환 → User 객체 반환
⑦ formLogin()에 failureHandler Bean을 등록했으면 실제로 .failureHandler(bean)에 연결했는지 반드시 확인

[SpringSecurityProject_1 — 최소 구조 실습]
⑧ {noop}비밀번호 접두사로 암호화 없이 평문 로그인 테스트 가능(학습용 한정, 운영에는 절대 금지)
⑨ loginProcessingUrl을 loginPage와 동일하게 지정하면 Security가 해당 URL의 POST를 직접 가로채 처리

[LamdaProject — 람다식/Stream]
⑩ 익명 클래스(메서드 1개짜리 인터페이스)만 람다식으로 대체 가능 → @FunctionalInterface로 명시
⑪ Stream 체이닝 순서: filter(조건) → map(변환) → sorted(정렬) → distinct(중복제거) → forEach/collect(최종 처리)
⑫ 합계/평균은 reduce(0,Integer::sum) 또는 mapToInt(...).average().orElse(0)

[배포/보안]
⑬ Docker Hub push용 계정정보는 secrets.DOCKER_USERNAME/DOCKER_PASSWORD로 분리, docker login은
   echo "${{secrets.DOCKER_PASSWORD}}" | docker login -u ... --password-stdin 형태로 표준입력 전달
⑭ docker run 직접 실행 대신 docker-compose.yml(build/image/env_file) + docker compose down/pull/up -d로
   배포 방식을 전환하면 컨테이너 재기동 스크립트가 짧아짐
⑮ 순수 JDBC 예제 코드라도 DB 계정정보를 소스에 하드코딩하지 말고, 최소한 커밋 전에 더미 값으로 교체하는 습관 필요
```

---

## 7. README 한 줄 요약 (표에 추가할 문구)

```text
| Day 11 | `SpringPiniaProject_2` Security 인증 3종 Bean 완성(AuthenticationManager/JdbcUserDetailsManager/PersistentTokenRepository), login.html form 추가로 실제 로그인 가능, Docker Compose+GitHub Actions 배포 파이프라인 재정비 / 신규 실습 `SpringSecurityProject_1`(인메모리 UserDetailsService)·`SpringSecurityProject_2`(DB연동 CustomUserDetailsService, failureHandler 연결 누락 버그 발견)·`LamdaProject`(람다식+Stream 문법) | [📄 보기](./springboot_day11.md) |
```
