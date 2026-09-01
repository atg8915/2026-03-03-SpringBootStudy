# 📘 Spring Boot Day 18 — JWT 기반 무상태 인증 도입 (SpringJWTSecurityProject_1/2)

## 0. 핵심 빠른 참조 — 세션 방식(Day11) 대비 오늘 바뀐 점

| 구분 | 세션 기반 Security (Day11) | JWT 기반 인증 (오늘) |
|------|------------------------------|------------------------|
| 인증 상태 저장 위치 | 서버 세션(JSESSIONID 쿠키) | 클라이언트가 들고 있는 JWT(쿠키 또는 Authorization 헤더) |
| SessionCreationPolicy | 기본값(IF_REQUIRED, 세션 생성) | `STATELESS`(세션 자체를 만들지 않음) |
| 로그인 처리 주체 | Security가 `loginProcessingUrl`을 가로채 `UsernamePasswordAuthenticationFilter`가 자동 처리 | `AuthController`가 직접 `AuthenticationManager.authenticate()`를 호출하고 토큰을 발급 |
| 요청마다 인증 확인 방식 | 세션에 저장된 `SecurityContext` 조회 | `JwtAuthenticationFilter`가 매 요청마다 토큰을 파싱·검증해서 `SecurityContext`를 새로 채움 |
| 로그인 성공 후 이동 | `LoginSuccessHandler` | Controller가 `ResponseEntity.status(FOUND)` + `Location` 헤더를 직접 구성 |

---

## 1. JWT 개념 및 인증 흐름

```text
xxxxx.yyyyy.zzzzz
Header . Payload(실제 정보) . Signature
```

- JWT는 점(`.`)으로 구분된 세 부분(Header/Payload/Signature)으로 구성됨. Payload에는 사용자 식별값(`sub`)과 권한(`role`) 같은 클레임이 담김
- 세션 방식과 JWT 방식의 근본적인 차이는 "로그인 상태를 누가 들고 있는가"임
  - 세션: 로그인 성공 시 서버가 세션을 만들고 `JSESSIONID`를 발급 → 이후 요청마다 서버가 세션 저장소를 조회해서 로그인 여부를 판단(서버가 상태를 가짐)
  - JWT: 로그인 성공 시 서버가 토큰을 발급해서 클라이언트에 넘김 → 이후 요청마다 클라이언트가 `Authorization: Bearer <토큰>`(또는 쿠키)으로 토큰을 실어 보내고 서버는 토큰 자체만 검증(서버는 상태를 갖지 않음)

### 전체 동작 순서

```text
로그인 요청 (POST /member/login, username/password)
  → AuthController
  → AuthenticationManager.authenticate()
  → CustomUserDetailsService (사용자 조회)
  → 인증 성공 → UserDetails 확보
  → JwtTokenProvider.createToken(username, role)
  → 토큰을 accessToken 쿠키로 응답(302 FOUND → /home)
  → 이후 요청마다 JwtAuthenticationFilter가
     헤더/쿠키에서 토큰 추출 → validate() → SecurityContext에 인증 정보 저장
  → Controller 접근 허용
```

---

## 2. SpringJWTSecurityProject_1 — 인메모리 하드코딩 사용자 + JWT

Day11의 `SpringSecurityProject_1`(인메모리 `UserDetailsService`)을 세션 방식이 아닌 JWT 방식으로 다시 구현한 버전.

```java
// JwtTokenProvider.java
@Component
public class JwtTokenProvider {
    private final String SECRET="my-secret-key-my-secret-key--my-secret-key--my-secret-key";

    public String createToken(String username, String role) {
        return Jwts.builder()
            .setSubject(username)
            .claim("role", role)
            .setIssuedAt(new Date())
            .setExpiration(new Date(System.currentTimeMillis()+3600000)) // 1시간
            .signWith(Keys.hmacShaKeyFor(SECRET.getBytes()))
            .compact();
    }
    public String getUsername(String token) {
        return Jwts.parserBuilder().setSigningKey(SECRET.getBytes())
            .build().parseClaimsJws(token).getBody().getSubject();
    }
    public boolean validate(String token) {
        try {
            Jwts.parserBuilder().setSigningKey(SECRET.getBytes())
                .build().parseClaimsJws(token);
            return true;
        } catch(Exception ex) { return false; }
    }
}
```

- `jjwt` 라이브러리(`Jwts.builder()`)로 토큰을 생성·파싱함. `signWith()`에 쓰는 SECRET 키는 실무에서는 하드코딩 대신 `application.yml`의 `jwt.secret: ${JWT_SECRET}` 같은 환경변수로 분리해야 함(오늘은 학습용으로 하드코딩)
- `setExpiration()`으로 만료시간(1시간)을 넣어야 탈취된 토큰이 영구히 유효하지 않게 됨
- `validate()`는 파싱 과정에서 예외가 나면(서명 불일치·만료 등) `false`를 반환해 유효성을 판단함

```java
// JwtAuthenticationFilter.java (OncePerRequestFilter 상속)
String header = request.getHeader("Authorization");
if (header != null && header.startsWith("Bearer ")) {
    token = header.substring(7);
}
if (token == null && request.getCookies() != null) {
    for (Cookie cookie : request.getCookies()) {
        if ("accessToken".equals(cookie.getName())) {
            token = cookie.getValue();
            break;
        }
    }
}
if (token != null && provider.validate(token)) {
    String username = provider.getUsername(token);
    UserDetails user = userDetailsService.loadUserByUsername(username);
    UsernamePasswordAuthenticationToken auth =
        new UsernamePasswordAuthenticationToken(username, null, user.getAuthorities());
    SecurityContextHolder.getContext().setAuthentication(auth);
}
```

- 토큰을 두 군데서 찾음 — 1순위 `Authorization` 헤더, 없으면 `accessToken` 쿠키. API 호출과 브라우저 화면 접근 둘 다 지원하기 위한 구조
- 인증 성공 시 `UsernamePasswordAuthenticationToken`을 만들어 `SecurityContextHolder`에 직접 넣어줌 — 세션 방식처럼 Security가 자동으로 채워주는 게 아니라 필터가 매번 수동으로 채워야 함

```java
// JwtSecurityConfig.java
http.csrf(csrf -> csrf.disable())
    .sessionManagement(session -> session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
    .authorizeHttpRequests(auth -> auth
        .requestMatchers("/","/login").permitAll()
        .requestMatchers("/admin").hasRole("ADMIN")
        .anyRequest().permitAll())
    .addFilterBefore(filter, UsernamePasswordAuthenticationFilter.class);
```

- `addFilterBefore(filter, UsernamePasswordAuthenticationFilter.class)`로 커스텀 JWT 필터를 Security 기본 필터보다 앞에 끼워 넣음
- `CustomUserDetailsService`는 Day11 `SpringSecurityProject_1`과 동일하게 `admin`/`user` 두 계정을 하드코딩하고 `{noop}1234`(암호화 없이 평문 비교)로 처리

```java
// AuthController.java
Authentication auth = manager.authenticate(new UsernamePasswordAuthenticationToken(username, password));
UserDetails user = (UserDetails) auth.getPrincipal();
String token = provider.createToken(user.getUsername(),
    user.getAuthorities().iterator().next().getAuthority());
ResponseCookie cookie = ResponseCookie.from("accessToken", token)
    .httpOnly(true).secure(false).path("/").maxAge(3600).build();
return ResponseEntity.status(HttpStatus.FOUND)
    .header(HttpHeaders.SET_COOKIE, cookie.toString())
    .header(HttpHeaders.LOCATION, "/home")
    .build();
```

- `httpOnly(true)`로 자바스크립트에서 쿠키를 읽지 못하게 막아 XSS로 인한 토큰 탈취를 방어함. 대신 `secure(false)`라 현재는 HTTP 환경에서도 쿠키가 전송됨(HTTPS 배포 시에는 `true`로 바꿔야 함)
- `config` 패키지 안에 내용이 비어있는 `AuthenticationManager.java` 클래스가 별도로 존재함 — Spring Security가 제공하는 `AuthenticationManager` 인터페이스와 이름이 겹치는 빈 클래스라 실제로 쓰이는 곳은 없어 보임(정리 대상)

---

## 3. SpringJWTSecurityProject_2 — MyBatis DB 연동 + 다중 권한 + JWT

Day11의 `SpringSecurityProject_2`(DB 연동 `CustomUserDetailsService`)를 JWT 방식으로 다시 구현한 버전. 하드코딩 대신 실제 테이블을 조회함.

```java
// MemberMapper.java / AuthorityMapper.java
@Select("SELECT userid,username,userpwd,enable,sex FROM springmember WHERE userid=#{userid}")
public MemberVO findByUserId(String userid);

@Select("SELECT userid,authority FROM authority WHERE userid=#{userid}")
public List<AuthorityVO> getAuthorityData(String userid);
```

```java
// CustomUserDetailsService.java
MemberVO member = mService.findByUserId(username);
if (member == null) throw new UsernameNotFoundException("사용자를 찾을 수 없습니다 :"+username);
if (member.getEnable() != 1) throw new UsernameNotFoundException("비활성화된 계정입니다!!");

List<AuthorityVO> authorityList = mService.getAuthorityData(username);
List<SimpleGrantedAuthority> authorities = authorityList.stream()
    .map(a -> new SimpleGrantedAuthority(a.getAuthority()))
    .toList();

return User.builder()
    .username(member.getUserid())
    .password(member.getUserpwd())
    .authorities(authorities)
    .build();
```

- 회원 테이블(`springmember`)과 권한 테이블(`authority`)을 분리해서 조회하고 그 권한 목록을 `SimpleGrantedAuthority` 리스트로 변환해 `User` 객체에 담음 — 한 사용자가 여러 권한을 가질 수 있는 구조
- `enable != 1`(휴면 계정)일 때 별도로 예외를 던져 비활성 계정 로그인을 막음

```java
// JwtSecurityConfig.java
.requestMatchers("/admin").hasRole("ADMIN")
.requestMatchers("/user").hasAnyRole("USER","ADMIN","MANAGER")
...
@Bean
public PasswordEncoder passwordEncoder() { return new BCryptPasswordEncoder(); }
```

- `hasAnyRole("USER","ADMIN","MANAGER")`로 여러 권한 중 하나만 있어도 접근을 허용함(Project_1은 `ADMIN` 단일 권한만 체크)
- `PasswordEncoder` 빈으로 `BCryptPasswordEncoder`를 등록해둠 — `AuthenticationManager.authenticate()` 내부의 `DaoAuthenticationProvider`가 이 빈을 참조해서 DB에 저장된 암호화 비밀번호와 입력값을 비교하는 구조(Project_1의 `{noop}` 평문 비교와 대비됨)
- 토큰 발급/검증 로직(`JwtAuthenticationProvider`)은 `jwt` 패키지로 옮겨졌을 뿐 Project_1의 `JwtTokenProvider`와 코드가 동일함

---

## 4. Project_1 vs Project_2 비교

| 구분 | SpringJWTSecurityProject_1 | SpringJWTSecurityProject_2 |
|------|------------------------------|-------------------------------|
| 사용자 조회 | 하드코딩(admin/user 2명) | MyBatis로 `springmember`/`authority` 테이블 조회 |
| 권한 구조 | ADMIN/USER 단일 role | ADMIN/USER/MANAGER 다중 role(`hasAnyRole`) |
| 비밀번호 검증 | `{noop}` 평문 비교 | `BCryptPasswordEncoder` 등록(암호화 비교) |
| 토큰 발급 클래스 | `JwtTokenProvider`(`config` 패키지) | `JwtAuthenticationProvider`(`jwt` 패키지) — 로직은 동일, 이름/패키지만 다름 |
| Filter의 Authentication principal | username 문자열 | `UserDetails` 객체 자체 |

---

## 5. 다시 만들 때 체크리스트

```text
[JWT 기본 개념]
① JWT는 Header.Payload.Signature 세 부분을 점(.)으로 이어붙인 문자열
② 서버는 로그인 상태를 저장하지 않고(STATELESS), 매 요청마다 클라이언트가 보낸 토큰만 검증
③ 토큰 서명 SECRET 키는 하드코딩하지 말고 환경변수로 분리해야 함(오늘은 학습 목적으로 하드코딩)
④ setExpiration()으로 만료시간을 반드시 넣어야 탈취된 토큰의 위험 기간을 제한할 수 있음

[JwtAuthenticationFilter]
⑤ OncePerRequestFilter를 상속해서 요청마다 한 번만 실행되도록 구현
⑥ 토큰은 Authorization 헤더(Bearer) 우선, 없으면 쿠키(accessToken) 순으로 조회
⑦ 검증 성공 시 SecurityContextHolder에 인증 정보를 직접 넣어줘야 함 — 세션 방식과 달리 자동으로 채워지지 않음
⑧ addFilterBefore(filter, UsernamePasswordAuthenticationFilter.class)로 Security 기본 필터보다 먼저 실행되게 등록

[SecurityConfig]
⑨ sessionCreationPolicy(STATELESS)로 세션 생성을 아예 막아야 진짜 무상태 인증이 됨
⑩ hasRole()은 단일 권한, hasAnyRole()은 여러 권한 중 하나만 있어도 허용

[쿠키 발급]
⑪ ResponseCookie.httpOnly(true)로 자바스크립트의 쿠키 접근을 막아 XSS로 인한 토큰 탈취를 방어
⑫ HTTPS 배포 환경에서는 secure(true)로 바꿔야 함(현재는 학습용으로 false)

[DB 연동 버전(Project_2) 추가 체크]
⑬ 휴면 계정(enable!=1) 등 상태값은 UserDetailsService 조회 단계에서 예외로 걸러야 함
⑭ 권한을 여러 개 가질 수 있는 구조라면 회원 테이블과 권한 테이블을 분리하고 List<GrantedAuthority>로 변환
⑮ 비밀번호는 BCryptPasswordEncoder로 암호화 저장/비교해야 실무 수준의 보안이 됨({noop}은 학습용 트릭)
```
