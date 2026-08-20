# 📘 Spring Boot Day 10 — Spring Security 로그인 뼈대 세팅 (`security first`)

## 0. 핵심 빠른 참조 — Day09 대비 오늘 바뀐 점

| 구분 | Day09까지 | Day10 (오늘) |
|------|-----------|--------------|
| 인증/인가 | (없음) | **Spring Security** 처음 도입 — `SecurityConfig`(`SecurityFilterChain`) 추가 |
| 로그인 흐름 | (없음) | `/member/login` 폼 로그인 + `LoginSuccessHandler`/`LoginFailHandler`(둘 다 **미구현 스텁**) |
| 인증 관련 Bean | (없음) | `AuthenticationManager`/`JdbcUserDetailsManager`/`PersistentTokenRepository` **모두 `return null`인 스텁 상태** |
| 비밀번호 암호화 | (없음) | `BCryptPasswordEncoder` Bean만 정상 등록 |
| VO | FoodVO/RecipeVO/RecipeDetailVO/ChefVO | **`MemberVO`/`AuthorityVO`** 신규 (회원/권한 테이블 매핑) |
| 화면 | header/home/food 상세 등 | `member/login.html`(로그인 폼) / `member/join.html`(빈 스텁) 신규, `header.html`에 `sec:authorize` 네임스페이스 추가 |
| build.gradle 의존성 | JPA/MyBatis/QueryDSL 위주 | **`spring-boot-starter-security`, `oauth2-client`, `jackson-databind`, `websocket`, `thymeleaf-extras-springsecurity6`** 5개 추가 |

> 커밋 메시지는 `"security first"`로 짧지만, diff를 보면 **Spring Security 인증 골격을 통째로 얹은 착수 단계 커밋** — `SecurityConfig`의 필터 체인(csrf/로그인/로그아웃/remember-me)은 동작 가능한 수준으로 작성됐지만, **실제 DB 연동 인증 로직(AuthenticationManager 등 3개 Bean)은 전부 `null` 반환**이라 지금 상태로 로그인을 시도하면 인증 자체가 완성되지 않은 상태임

---

## 1. build.gradle — Security 관련 의존성 5종 추가

```groovy
dependencies {
	implementation 'org.springframework.boot:spring-boot-starter-security'
	implementation 'org.springframework.boot:spring-boot-starter-oauth2-client'
	implementation 'com.fasterxml.jackson.core:jackson-databind'
	implementation 'org.springframework.boot:spring-boot-starter-websocket'
	implementation 'org.thymeleaf.extras:thymeleaf-extras-springsecurity6'
}
```
- `spring-boot-starter-security` — 오늘 세팅한 `SecurityConfig`의 기반
- `oauth2-client` — 소셜 로그인 대비로 미리 추가된 것으로 보임(오늘 커밋에는 실제 OAuth2 설정 코드는 없음)
- `thymeleaf-extras-springsecurity6` — `header.html`에서 쓴 `sec:authorize` 속성을 쓰기 위한 필수 의존성
- `websocket` — 실시간 채팅 메뉴(`header.html`에 이미 있던 "실시간 채팅" 링크)를 염두에 둔 선추가로 추정, 오늘 커밋에는 관련 설정 없음

---

## 2. SecurityConfig — 인증/인가 필터 체인

```java
@Configuration
@EnableWebSecurity
@RequiredArgsConstructor
public class SecurityConfig {
	private final LoginSuccessHandler loginSuccessHandler;
	private final LoginFailHandler loginFailHandler;
	private final DataSource dataSource;

	@Bean
	public SecurityFilterChain filter(HttpSecurity http) throws Exception {
		http
		  .csrf(csrf -> csrf.disable())
		  .authorizeHttpRequests(auth -> auth
				  .requestMatchers("/","/member/**").permitAll()
				  .requestMatchers("/admin/**").hasRole("ADMIN")
				  .anyRequest().permitAll()
		  )
		  .formLogin(form -> form
			    .loginPage("/member/login")
			    .loginProcessingUrl("/member/login_process")
			    .usernameParameter("userid")
			    .passwordParameter("userpwd")
			    .defaultSuccessUrl("/",false)
			    .successHandler(loginSuccessHandler)
			    .failureHandler(loginFailHandler)
			    .permitAll()
		  )
		  .rememberMe(remember -> remember
				  .key("my-secret-key")
				  .rememberMeParameter("remember-me")
				  .tokenValiditySeconds(60*60*24)
		  )
		  .logout(logout -> logout
			    .logoutUrl("/member/logout")
			    .logoutSuccessUrl("/")
			    .invalidateHttpSession(true)
			    .deleteCookies("remember-me","JSESSIONID")
		  );
		return http.build();
	}
}
```
- 클래스 상단에 **Spring Security 개념 총정리 주석**을 남겨둠 (인증 vs 인가, DispatcherServlet 필터 체인 위치, AuthenticationFilter → AuthenticationManager → AuthenticationProvider → UserDetailService 흐름)
- `csrf.disable()` — 학습 단계라 CSRF 방어를 꺼둔 상태(주석에 "일반 보안 = csrf.disable()"이라고만 메모, 운영 전환 시 재검토 필요)
- 경로별 권한: `/`, `/member/**`는 전체 허용, `/admin/**`는 `ROLE_ADMIN` 필요 — 다만 바로 아래 `.anyRequest().permitAll()`이 있어 **사실상 전체 허용과 동일한 효과**(위 두 규칙이 뒤 규칙에 가려지는 형태는 아니지만, 세부 경로 외 나머지가 전부 열려있어 인가 정책이 아직 느슨함)
- `rememberMe().key("my-secret-key")` — remember-me 토큰 서명용 키가 **소스에 문자열 그대로 하드코딩**되어 있음. 지금은 튜토리얼용 더미 값이지만, Day07~09에서 지켜온 "`${환경변수}`로 분리" 원칙을 여기에도 적용해야 함
- `logout()`에서 `remember-me` / `JSESSIONID` 쿠키를 함께 삭제하도록 세팅

---

## 3. 인증 관련 Bean 3종 — 전부 `null` 반환 (미완성 스텁)

```java
@Bean
public AuthenticationManager authenticationManager(
		HttpSecurity http, BCryptPasswordEncoder passwordEncoder) throws Exception {
	return null;
}

@Bean
public JdbcUserDetailsManager jdbcUserDetailsManager() {
	return null;
}

@Bean
public PersistentTokenRepository persistentTokenRepository() {
	return null;
}

@Bean
public BCryptPasswordEncoder passwordEncoder() {
	return new BCryptPasswordEncoder();
}
```
- `passwordEncoder()`만 실제 구현이 있고, 나머지 3개(`authenticationManager`/`jdbcUserDetailsManager`/`persistentTokenRepository`)는 **자리만 잡아둔 채 `return null`**
- `DataSource dataSource`를 필드로 주입은 받아뒀지만 아직 어디에도 사용하지 않음 → 다음 작업에서 `JdbcUserDetailsManager(dataSource)`처럼 DB 연동을 채워 넣을 자리로 추정
- **주의**: 이 상태로 실제 로그인(`/member/login_process`)을 시도하면 인증 처리 로직이 없어 정상 동작하지 않음 — 오늘 커밋은 "필터 체인 골격 + 화면"까지만 완성

---

## 4. LoginSuccessHandler / LoginFailHandler — 미구현 스텁

```java
@Component
public class LoginSuccessHandler implements AuthenticationSuccessHandler {
	@Override
	public void onAuthenticationSuccess(HttpServletRequest request, HttpServletResponse response,
			Authentication authentication) throws IOException, ServletException {
		// TODO Auto-generated method stub
	}
}
```
```java
@Component
public class LoginFailHandler implements AuthenticationFailureHandler {
	@Override
	public void onAuthenticationFailure(HttpServletRequest request, HttpServletResponse response,
			AuthenticationException exception) throws IOException, ServletException {
		// TODO Auto-generated method stub
	}
}
```
- 둘 다 `@Component`로 등록만 해두고 본문은 아직 구현되지 않은 상태 — `SecurityConfig`에는 이미 연결(`successHandler`/`failureHandler`)돼 있어 **인터페이스 구현체를 먼저 배치하고 로직은 다음에 채우는 순서**로 진행 중

---

## 5. VO 신규 2종 — 회원/권한 테이블 매핑

| VO | 대응 테이블 | 주요 필드 | 비고 |
|----|------------|-----------|------|
| `MemberVO` | `MEMBER`(추정) | userid, username, userpwd, enable, sex | 클래스 상단에 Oracle 컬럼 스펙 주석 포함(`USERID`/`USERNAME`/`USERPWD` NOT NULL, `ENABLE` NUMBER(1)) |
| `AuthorityVO` | `AUTHORITY`(추정) | userid, authority | `authority` 필드 주석에 `// 권한 => ROLE_ADMIN` — Spring Security의 `hasRole("ADMIN")`과 연결되는 값으로 보임 |

- 둘 다 `@Data`만 붙은 순수 VO — Day09의 다른 VO들처럼 아직 `@Entity` 전환 전 단계
- `JdbcUserDetailsManager`가 정상 구현되면 이 두 테이블 구조(`MEMBER`/`AUTHORITY`)를 기반으로 `users-by-username-query`/`authorities-by-username-query` SQL을 채워 넣을 것으로 예상

---

## 6. 화면 — 로그인 폼 / 회원가입 스텁 / header 권한 표시 준비

### `member/login.html`
```html
<div class="row">
  <h3>Login</h3>
  <table class="table">
    <tr>
      <th>ID</th>
      <td><input type="text" name=userid size=20 class="input-sm"></td>
    </tr>
    <tr>
      <th>PW</th>
      <td><input type="password" name=userpwd size=20 class="input-sm"></td>
    </tr>
    <tr>
      <td colspan="2">
        <button class="btn-sm btn-danger" type="submit">로그인</button>
        <button class="btn-sm btn-primary" type="button" onclick="javascript:history.back()">취소</button>
      </td>
    </tr>
  </table>
</div>
```
- `input name`이 `SecurityConfig`의 `.usernameParameter("userid")` / `.passwordParameter("userpwd")`와 **정확히 일치** — 커스텀 파라미터명을 쓸 때 폼 필드명도 반드시 맞춰야 한다는 점이 확인됨
- 다만 `<form>` 태그와 `action`/`method`가 빠져 있어 **아직 실제 제출은 되지 않는 정적 마크업 상태** (버튼이 `type="submit"`이어도 감싸는 폼이 없음) → 다음 작업에서 `<form th:action="@{/member/login_process}" method="post">` 추가 필요

### `member/join.html`
```html
<body>
</body>
```
- `<title>` 외 내용이 전혀 없는 빈 스텁 — 회원가입 화면 자리만 잡아둔 상태

### `header.html` — sec 네임스페이스 추가
```html
<html xmlns:th="http://www.thymeleaf.org"
      xmlns:sec="http://www.thymeleaf.org/extras/spring-security">
...
<!--  <li sec:authorize=""><a href="#">쉐프</a></li> -->
```
- `thymeleaf-extras-springsecurity6`를 쓰기 위한 `xmlns:sec` 네임스페이스를 추가
- "쉐프" 메뉴를 기존 위치에서 빼서 `sec:authorize` 속성과 함께 **주석 처리한 채로 이동** — 로그인 권한에 따라 노출 여부를 제어할 예정이지만 `sec:authorize=""` 값이 비어 있어 아직 조건이 채워지지 않음(예: `sec:authorize="isAuthenticated()"`)

---

## 7. 비교표 — Day09 vs Day10

| 구분 | Day09 | Day10 (오늘) |
|------|-------|--------------|
| 관심사 | 프로젝트 초기 스캐폴딩(화면/데이터 골격) | **인증/인가(Security) 골격** 추가 |
| 완성도 | Mapper/Service 델리게이트는 대부분 동작 | 필터 체인은 작성됐지만 **DB 연동 인증 Bean 3종은 전부 null** |
| 미완성 표시 방식 | `foodPages()`가 `return null` | `authenticationManager`/`jdbcUserDetailsManager`/`persistentTokenRepository`가 `return null`, 핸들러 2종은 빈 메서드로 방치 |
| 화면-보안 연결 | (해당 없음) | `login.html` input name ↔ `usernameParameter`/`passwordParameter` 매칭, `header.html`에 `sec:authorize` 준비만 해둠 |

---

## 8. 다시 만들 때 체크리스트

```text
[Security 기본 세팅]
① build.gradle에 spring-boot-starter-security + thymeleaf-extras-springsecurity6 추가
② @EnableWebSecurity + SecurityFilterChain Bean으로 인가 규칙/로그인/로그아웃/remember-me 설정
③ formLogin().usernameParameter()/passwordParameter() 커스텀 시 로그인 폼 input name도 반드시 동일하게 맞출 것

[다음에 채워야 할 것 — 오늘은 미완성]
④ AuthenticationManager Bean에 실제 AuthenticationProvider(DaoAuthenticationProvider 등) 연결
⑤ JdbcUserDetailsManager(dataSource)로 MemberVO/AuthorityVO 테이블 기반 인증 쿼리(users-by-username-query, authorities-by-username-query) 작성
⑥ LoginSuccessHandler/LoginFailHandler 본문 구현(리다이렉트/에러 메시지 처리)
⑦ login.html에 <form action/method> 추가해야 실제 제출 가능
⑧ header.html의 sec:authorize="" 조건식 채우기(예: isAuthenticated(), hasRole('ADMIN'))

[보안 유지 원칙]
⑨ rememberMe().key("my-secret-key") 같은 하드코딩 값도 ${환경변수}로 분리(Day07부터 이어온 원칙)
⑩ csrf.disable()은 학습 단계 한정 — 운영 전환 시 재검토
```

---

