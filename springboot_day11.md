# 📘 Spring Boot Day 11 — SpringPiniaProject_2 Security 인증 완성 + 인증 방식 비교 실습(SpringSecurityProject_1/2) + LamdaProject(람다식) 신규

오늘은 프로젝트 4개에 커밋이 발생했다.

| 프로젝트 | 오늘 커밋 수 | 오늘 한 일 |
|------|:---:|------|
| `SpringPiniaProject_2` | 15개 | Day10에서 `null`이던 인증 Bean 3종을 실제로 구현 완료, Docker Compose/GitHub Actions 배포 스크립트 반복 수정 |
| `SpringSecurityProject_2` | 2개 | 신규 프로젝트 — DB(MyBatis) 연동 `CustomUserDetailsService` 방식으로 인증 구현 |
| `SpringSecurityProject_1` | 1개 | 신규 프로젝트 — 인메모리(하드코딩) `UserDetailsService`로 Security 최소 구조 실습 |
| `LamdaProject` | 1개 | 신규 프로젝트 — 람다식 문법 + Stream(`filter`/`map`/`sorted`/`reduce`) 학습 |

커밋 메시지가 전부 `1111`/`1234`/`Update deploy.yml`처럼 부실해서, 아래 내용은 전부 `git show`로 diff를 직접 확인해서 정리했다.

---

## 0. 핵심 빠른 참조 — Day10 대비 오늘 바뀐 점 (`SpringPiniaProject_2`)

| 구분 | Day10까지 | Day11(오늘) |
|------|-----------|--------------|
| `AuthenticationManager` | `return null` (미구현) | `AuthenticationManagerBuilder`로 `jdbcUserDetailsService()` + `passwordEncoder()` 연결해서 실제 구현 완료 |
| `JdbcUserDetailsManager` | `return null` | `dataSource` 기반으로 생성 + `users-by-username-query`/`authorities-by-username-query` SQL 직접 세팅 |
| `PersistentTokenRepository` | `return null` | `JdbcTokenRepositoryImpl`에 `dataSource` 연결해서 실제 구현, `rememberMe().tokenRepository()`에도 연결 |
| `LoginFailHandler` | 빈 스텁 | `BadCredentialsException`/`InternalAuthenticationServiceException`/`DisabledException`별 에러 메시지 분기 + `/member/login`으로 forward |
| `LoginSuccessHandler` | 빈 스텁 | `response.sendRedirect("/")` 구현 |
| `login.html` | `<form>` 태그 없이 정적 마크업만 존재 | `<form action="/member/login_process" method="post">` 추가로 **실제 로그인 제출 가능**해짐, remember-me 체크박스/에러 메시지(`[[${message}]]`) 표시 추가 |
| `header.html` | `sec:authorize=""` 값 비어있음(조건 미완성) | `sec:authorize="isAuthenticated()"` / `"!isAuthenticated()"` / `"hasRole('ADMIN')"` / `"hasRole('USER')"` 로 로그인 상태별 메뉴 분기 완성, 로그아웃 링크 추가 |
| 배포 | (Day09 기준) `docker run --env-file .env`로 컨테이너 직접 실행 | `docker-compose.yml` 신규 도입 + Docker Hub push 단계 추가된 배포 파이프라인으로 전환 |

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
- Day10에서 자리만 잡아두고 `null`이었던 3개 Bean을 전부 실제 구현으로 채웠다 (커밋 `04c427c`)
- 핵심은 `AuthenticationManagerBuilder`를 `http.getSharedObject()`로 꺼내서 `userDetailsService()`/`passwordEncoder()`를 연결하는 패턴 — `new`로 직접 만들지 않고 Spring Security가 제공하는 Builder를 통해 구성
- `JdbcUserDetailsManager`의 쿼리 컬럼명이 Day10에서 정의한 `MemberVO`(`springmember` 테이블 추정) / `AuthorityVO`(`authority` 테이블) 구조와 일치
- `rememberMe()`에도 `.tokenRepository(persistentTokenRepository())`를 연결해서, remember-me 토큰이 DB(`PersistentTokenRepository` 테이블)에 저장되도록 완성

### 1-2. 중간에 실수로 되돌렸던 부분 — `914f7d1` → `d321fc7`

같은 날 안에서 `authenticationManager` Bean 전체를 통째로 주석 처리했다가(`914f7d1`, 16:08) 10분 뒤(`d321fc7`, 16:26) 다시 주석을 풀어 원상복구한 흔적이 있다. 실제 기능 변화는 없고, 작업 중 임시로 꺼봤다가 다시 켠 것으로 보인다.

### 1-3. LoginFailHandler / LoginSuccessHandler — 실제 로직 구현

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
- 예외 타입별로 분기해서 사용자에게 보여줄 메시지를 다르게 세팅하고, `request.setAttribute("message", ...)` 후 `forward`로 로그인 화면에 그대로 표시(Thymeleaf `[[${message}]]`로 출력)
- `LoginFailHandler`에는 `MemberVO`/`mService` 등을 직접 조회하는 코드가 주석으로 남아있는데, 이건 실제 미사용 코드라 그대로 두고 지금은 `AuthenticationException` 타입 분기만 동작

### 1-4. header.html / login.html — 화면에서 로그인 상태 반영

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
- Day10까지 비어있던 `sec:authorize=""` 조건이 전부 채워졌다 — 로그인 여부/권한별로 메뉴가 분기되는 구조 완성
- 다만 `<li sec:authorize="isAuthenticated()">`로 감싸놓고 그 안에 다시 `<li sec:authorize="hasRole('ADMIN')">`를 중첩한 형태라, `<li>` 태그가 제대로 닫히지 않은 채 다음 `<li>`들이 이어지는 모양 — HTML 구조상 정리가 더 필요해 보인다(동작은 하지만 마크업이 깔끔하지 않음)
- `login.html`에는 그동안 없던 `<form action="/member/login_process" method="post">`가 드디어 추가되어 실제 로그인 제출이 가능해졌고, remember-me 체크박스와 에러 메시지 출력 영역(`[[${message}]]`)도 함께 추가됨

### 1-5. FoodMapper — 의미 없는 테스트 편집

```java
/*
   ...
   OFFFSET #{start} ROWS FETCH NEXT 12 ROWS ONLY
 </select>
 dddddd
 dddddd
*/
```
- 커밋 `64f6734`에서 주석 블록 안에 `dddddd`를 두 줄 추가한 것 — 실제 기능과 무관한 테스트성 편집으로 보인다. 코드 동작에는 영향 없음(주석 안이라 컴파일에도 영향 없음)

### 1-6. Docker Compose + GitHub Actions 배포 파이프라인 재정비

Day09까지는 `docker run --name food-app -d --env-file .env -p 9090:9090 food-app` 형태로 컨테이너를 직접 실행했는데, 오늘 `docker-compose.yml`을 새로 도입하면서 배포 스크립트가 여러 차례 수정됐다.

**`docker-compose.yml` 최종본**
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

**`.github/workflows/deploy.yml` 최종본 (배포 단계만 발췌)**
```yaml
      - run: docker build -t pinia-app .
      - run: docker tag pinia-app ${{secrets.DOCKER_USERNAME}}/pinia-app:latest
      - run: echo "${{secrets.DOCKER_PASSWORD}}" | docker login -u "${{secrets.DOCKER_USERNAME}}" --password-stdin
      - run: docker push ${{secrets.DOCKER_USERNAME}}/pinia-app
      - run: docker compose down
      - run: docker compose pull
      - run: docker compose up -d
```

오늘 이 워크플로우 파일이 변경된 6번의 커밋을 시간순으로 정리하면:
1. `677d058` — 이미지 이름을 `food-app`에서 `pinia-app`으로 변경(프로젝트 이름 변경 반영)
2. `0ad96ce` — `docker run` 방식을 통째로 걷어내고 `docker login` → `docker push` → `docker compose down/pull/up -d` 방식으로 전면 교체 (예전 `nohup java -jar ...` 방식의 주석 잔재도 함께 삭제)
3. `95d3ce6` — 이미지 태그를 `pinia-app`에서 `${{secrets.DOCKER_USERNAME}}/pinia-app`처럼 계정명을 붙인 이름으로 변경(Docker Hub push 규칙에 맞춤)
4. `51f5c1d` — `docker tag`로 별도 태깅 단계를 추가하고, `docker build`는 다시 계정명 붙은 이름으로 직접 빌드하도록 변경
5. `6af11a2` — `docker tag ... :lastest` 오타를 `:latest`로 수정
6. `e71884c` — `docker build`의 이미지 이름을 계정명 없는 `pinia-app`으로 되돌림(태그 단계에서 계정명을 붙이는 역할을 분리)

이 과정에서 `docker login`/`docker push`용 계정정보는 전부 `${{secrets.DOCKER_USERNAME}}`/`${{secrets.DOCKER_PASSWORD}}`로 GitHub Actions Secrets를 참조하고 있어, Day07~09부터 이어온 "민감정보는 환경변수/Secrets로 분리" 원칙이 여기서도 유지되고 있다.

---

## 2. SpringSecurityProject_2 — DB 연동 CustomUserDetailsService 신규 실습

`SpringPiniaProject_2`와 별개로, Spring Security 인증을 처음부터 다시 연습하는 신규 프로젝트가 두 개(`SpringSecurityProject_1`/`_2`) 생겼다. `_2`는 `JdbcUserDetailsManager`(쿼리를 문자열로 세팅)가 아니라, **직접 만든 `UserDetailsService` 구현체**로 DB를 연동하는 방식을 쓴다.

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
- MyBatis `@Select` 어노테이션으로 직접 SQL을 작성하고, `UserDetailsService`를 구현해서 조회 결과를 Spring Security의 `User` 객체로 변환하는 구조 — `SpringPiniaProject_2`의 `JdbcUserDetailsManager`(내장 쿼리 프로퍼티로 SQL을 넘기는 방식)와 달리, **조회 로직을 완전히 직접 제어**할 수 있는 방식
- `SecurityConfig`에서는 아래처럼 `LoginFailHandler` Bean을 등록해놓고도, `formLogin()`에는 `.failureHandler(null)`을 넘기고 있어 **실제로는 연결이 안 된 상태**(버그로 보임 — `.failureHandler(loginFailHandler())`로 고쳐야 실제 실패 핸들러가 동작)
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
- `application.yml`의 DB 접속 정보는 처음부터 `${DB_URL}`/`${DB_USERNAME}`/`${DB_PASSWORD}` 환경변수 참조로 작성되어 있어, 마스킹 원칙이 신규 프로젝트에도 그대로 적용됨
- 이후 커밋(`5157ef8`)에서는 `docker-compose.yml`에 `env_file: .env`를 추가만 하고 아직 `Dockerfile`은 없는 상태 — 배포 설정은 다음 작업에서 이어질 것으로 보임

---

## 3. SpringSecurityProject_1 — 인메모리(하드코딩) UserDetailsService 실습

`_1`은 DB 연동 없이, **하드코딩된 2명의 사용자(admin/user)** 로 Security 최소 구조만 익히는 실습 프로젝트다.

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
- `{noop}1234`는 Spring Security의 **비밀번호 인코딩 접두사** — `{noop}`은 "암호화 안 함(no operation)"을 의미해서, `BCryptPasswordEncoder` 없이도 평문 비교가 가능하게 해주는 학습용 트릭
- `id`가 `admin`이 아니면 무조건 `user`(권한 `USER`)로 처리되는 구조라, 실제로는 어떤 아이디를 넣어도 로그인은 되는 상태(비밀번호만 `1234`면 통과) — DB 검증이 없는 순수 실습용 코드
- `SecurityConfig`의 `formLogin()`은 `loginProcessingUrl("/login")`을 로그인 화면 경로(`loginPage("/login")`)와 **동일하게** 지정한 게 특징 — 클래스 상단 주석에 "Spring Security가 `/login` POST를 Controller 대신 가로채서 처리하는 방식"이라고 직접 설명이 남아있음
```java
.formLogin(form->form
	 .loginPage("/login")
	 .loginProcessingUrl("/login") // Security가 /login POST를 가로채 처리
	 .defaultSuccessUrl("/",true)
	 .failureUrl("/login?error")
	 .permitAll()
)
```
- `SpringSecurityProject_2`와 달리 `failureHandler` 대신 `.failureUrl("/login?error")`을 사용 — 핸들러 클래스 없이 URL 리다이렉트만으로 실패 처리를 하는, 더 단순한 방식

---

## 4. LamdaProject — 람다식 / Stream 문법 신규 학습

Security와는 무관한 별도 주제로, 자바 람다식과 Stream API를 정리하는 프로젝트가 새로 생겼다. Spring Boot 프로젝트가 아니라 **순수 Java(STS/Eclipse) 프로젝트**다.

```java
// 익명 클래스 방식(주석 처리) vs 람다식
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
- `MainClass`~`MainClass_3`로 이어지며 "익명 클래스 → 람다식" 순서로 **Before/After를 주석으로 남겨가며** 학습한 흔적이 뚜렷함
- `@FunctionalInterface`(추상 메서드 1개짜리 인터페이스)만 람다식으로 대체 가능하다는 제약을 실습
- `MainClass_2`/`MainClass_3`에서는 Stream의 `forEach`/`filter`/`map`/`sorted`/`distinct`/`reduce`/`average`를 `List.of(...)` 데이터로 하나씩 실습
- `EmpDAO`(`com.sist.stream` 패키지)는 Oracle에 순수 JDBC로 직접 접속해서 `emp` 테이블을 조회하는 예제로, DB 접속 정보가 소스에 그대로 하드코딩되어 있었음(마스킹 처리):
```java
private final String URL="jdbc:oracle:thin:@localhost:1521:XE";
...
conn=DriverManager.getConnection(URL,"<DB계정>","<DB계정>");
```
  Spring Boot 쪽 프로젝트들이 Day07부터 `${DB_URL}` 환경변수 방식으로 넘어간 것과 달리, 이 프로젝트는 스트림 문법 자체가 학습 목적이라 접속 방식은 예전 방식(하드코딩) 그대로 사용 중 — 프로젝트 성격이 다르므로 오늘 안에서 굳이 고칠 필요는 없어 보이지만, 실제 값이 코드에 남아있다는 점은 유의할 것
- `MainClass_3` 하단 주석에 로그인 인증과 연결 지을 수 있는 스트림 활용 예시(`filter(u -> "ADMIN".equals(u.getRole()))`, `map(Member::getName)` 등)를 미리 적어둔 것으로 보아, 오늘 다룬 Security 프로젝트들과 이어서 "회원 목록 필터링/권한별 분류"에 스트림을 적용할 계획으로 추정됨

---

## 5. 비교표 — 오늘 다룬 3가지 인증(UserDetailsService) 구현 방식

| 구분 | `SpringPiniaProject_2` | `SpringSecurityProject_2` | `SpringSecurityProject_1` |
|------|------------------------|----------------------------|------------------------------|
| 구현 방식 | `JdbcUserDetailsManager` (내장 클래스 + SQL 문자열 프로퍼티) | 직접 만든 `UserDetailsService` 구현체 + MyBatis `@Select` | 직접 만든 `UserDetailsService` 구현체 + 하드코딩(DB 없음) |
| 사용자/권한 조회 | `setUsersByUsernameQuery`/`setAuthoritiesByUsernameQuery`에 SQL을 그대로 넘김 | `UserMapper`로 SQL 실행 후 결과를 직접 `Set<GrantedAuthority>`로 변환 | `if(username.equals("admin"))` 분기로 고정된 2개 계정만 반환 |
| 비밀번호 처리 | `BCryptPasswordEncoder` 실제 암호화 | `BCryptPasswordEncoder` Bean은 등록돼 있으나 이 흐름에선 미사용 | `{noop}1234` — 암호화 없이 평문 비교 |
| 로그인 실패 처리 | `LoginFailHandler` 구현체 + `AuthenticationFailureHandler` 정상 연결 | `LoginFailHandler` 구현체는 만들었지만 `.failureHandler(null)`로 **연결 누락(버그)** | 핸들러 없이 `.failureUrl("/login?error")`만 사용 |
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
⑤ login.html에 <form action method="post"> 반드시 추가해야 실제 제출 가능(Day10에서 빠져있던 부분)

[SpringSecurityProject_2 — DB 연동 직접 구현]
⑥ UserDetailsService 구현체에서 MyBatis Mapper로 회원/권한 조회 → Set<GrantedAuthority>로 변환 → User 객체 반환
⑦ formLogin()에 failureHandler Bean을 등록했으면 실제로 .failureHandler(bean)에 연결했는지 반드시 확인
   (오늘 프로젝트에서 .failureHandler(null)로 연결이 빠진 버그 발견)

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
⑮ 순수 JDBC 예제 코드라도 DB 계정정보를 소스에 하드코딩하지 말고, 학습 목적이라면 최소한 커밋 전에
   더미 값으로 교체하는 습관 필요
```

---

## 7. README 한 줄 요약 (표에 추가할 문구)

```text
| Day 11 | `SpringPiniaProject_2` Security 인증 3종 Bean 완성(AuthenticationManager/JdbcUserDetailsManager/PersistentTokenRepository), login.html form 추가로 실제 로그인 가능, Docker Compose+GitHub Actions 배포 파이프라인 재정비 / 신규 실습 `SpringSecurityProject_1`(인메모리 UserDetailsService)·`SpringSecurityProject_2`(DB연동 CustomUserDetailsService, failureHandler 연결 누락 버그 발견)·`LamdaProject`(람다식+Stream 문법) | [📄 보기](./springboot_day11.md) |
```

---
