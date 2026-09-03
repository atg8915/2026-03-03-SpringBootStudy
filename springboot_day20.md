# 📘 Spring Boot Day 20 — Docker 기반 CI/CD 전환 착수 + JWT 로그인/로그아웃 완성

## 0. 핵심 빠른 참조 — 어제 대비 오늘 바뀐 점

| 구분 | Day19(어제) | Day20(오늘) |
|------|-------------|-------------|
| 배포 방식 | GitHub Actions에서 jar만 빌드 → `rsync`로 서버 전송 → SSH 접속 후 `nohup java -jar` 직접 실행 | GitHub Actions에서 Docker 이미지 빌드 → DockerHub push 단계 추가(다만 서버 쪽 실행은 아직 예전 rsync+nohup 그대로 남아있음) |
| JWT 인증 | 토큰 발급/검증 유틸 + DB 조회 계층까지만 준비, 로그인 흐름 미완성 | 로그인/로그아웃 REST API 완성, 인증 필터를 필터체인에 연결, 로그인 화면까지 전체 흐름 연결 |

---

## 1. AWS 서버에 Docker / Docker Compose 설치 (cicdServer)

Day17에서 Kafka 컨테이너용으로 이미 한 번 다뤘던 설치 절차를, 오늘은 Docker 기반 배포로 전환하기 위해 다시 정리함(PuTTY 접속, 자바 설치까지는 Day19와 동일).

### 1-1. Docker 설치
```bash
sudo apt-get install ca-certificates curl gnupg lsb-release -y

curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] \
https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" \
  | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io -y

sudo systemctl status docker
sudo systemctl enable docker
```
- GPG 키를 `dearmor`로 변환해 신뢰 키링에 등록한 뒤, 그 키링을 참조하는 apt 저장소를 `docker.list`로 새로 등록하는 순서
- `systemctl enable docker`로 서버 재부팅 후에도 Docker 데몬이 자동으로 뜨도록 설정

### 1-2. Docker Compose 설치
```bash
sudo curl -L "https://github.com/docker/compose/releases/download/1.28.2/docker-compose-$(uname -s)-$(uname -m)" \
  -o /usr/local/bin/docker-compose
```

### 1-3. 일반 계정으로 docker 명령 사용 권한 부여
```bash
sudo usermod -aG docker ubuntu
```
- ubuntu 계정을 docker 그룹에 추가 — 이후 `docker`/`docker-compose` 명령을 `sudo` 없이 실행 가능

### 1-4. GitHub Actions용 Secrets 등록
DB 접속정보, DockerHub 계정, 서버 SSH 인증키를 리포지토리 Settings → Secrets and variables → Actions에 등록:

| Secret 이름 | 용도 |
|-------------|------|
| `DB_URL` / `DB_USERNAME` / `DB_PASSWORD` | 서버에서 만들 `.env`에 채워질 DB 접속정보 |
| `DOCKERHUB_USERNAME` / `DOCKERHUB_PASSWORD` | GitHub Actions에서 DockerHub 로그인·이미지 push |
| `SERVER_SSH_KEY` | Day19에서 만들어둔 GitHub Actions 인증용 SSH 개인키 |

---

## 2. 목표로 하는 전체 CI/CD 파이프라인 흐름

```text
Git main Push
  |
  ├─(참고) Webhook -> Jenkins -> 배포 명령  ※ Day16에서 다룬 별도 경로, 오늘 것과는 무관
  |
  └─ GitHub Actions 실행
       Repository Checkout
        -> JDK 21 설치
        -> .env 생성 (DB 접속정보)
        -> gradlew build로 jar 생성                    ]-- 여기까지 CI
        -> Docker 이미지 빌드
        -> DockerHub push
        -> AWS EC2 SSH 접속
        -> AWS EC2에 .env 생성
        -> DockerHub에서 이미지 pull
        -> docker run(또는 docker-compose)으로 실행     ]-- CD
```
- CI(빌드~이미지 push)와 CD(서버 배포)를 하나의 워크플로우 안에서 이어 붙이는 구조
- Jenkins+Webhook 경로는 Day16에서 별도로 구축해둔 CI/CD 경로이며 오늘 GitHub Actions 파이프라인과는 별개의 배포 수단임

---

## 3. GitHub Actions 워크플로우 — Docker 단계 추가 (`.github/workflows/deploy.yml`)

### 오늘 실제로 반영된 범위
```yaml
- name: SetUp JDK 21
  uses: actions/setup-java@v4     # v1 -> v4, distribution: temurin 명시
  with:
    distribution: 'temurin'
    java-version: '21'

- name: 환경설정
  run: |
    rm -f.env
    touch .env
    echo "DB_URL=${{secrets.DB_URL}}" >> .env
    echo "DB_URL=${{secrets.DB_USERNAME}}" >> .env
    echo "DB_URL=${{secrets.DB_PASSWORD}}" >> .env

- name: Login DockerHub
  run: |
    echo "${{secrets.DOCKERHUB_PASSWORD}}" | \
    docker login -u "${{secrets.DOCKERHUB_USERNAME}}" --password-stdin

- name: Build Docker Image
  run: docker build -t ${{secrets.DOCKERHUB_USERNAME}}/last-app:latest

- name: Push Docker Image
  run: docker push ${{secrets.DOCKERHUB_USERNAME}}/last-app:latest
```
- `actions/checkout`은 v2→v4, `actions/setup-java`는 v1→v4로 올리고 `distribution: temurin` 명시(예전 버전은 배포판 미지정으로도 동작했지만 최신 액션은 명시가 필요)
- `.env` 생성 단계와 DockerHub 로그인/빌드/push 3단계를 Gradle 빌드 뒤에 새로 추가

### 오늘 코드에서 발견한 문제
- `.env` 생성 스크립트에서 세 줄 모두 키 이름이 `DB_URL`로 동일하게 적혀 있음(`DB_USERNAME`/`DB_PASSWORD`로 각각 고쳐야 값이 제대로 들어감) — 복사 후 변수명만 안 고친 전형적인 실수
- `rm -f.env`는 `-f`와 파일명 사이에 공백이 빠진 형태 — 의도한 대로 `.env` 삭제가 안 될 수 있어 `rm -f .env`로 수정 필요
- Docker 이미지 빌드/push 단계는 새로 추가됐지만 그 아래 "AWS 연결" 이후 섹션(SSH 키 설정 → `rsync`로 jar 전송 → `nohup java -jar` 재실행)은 오늘 손대지 않고 그대로 남아있음 — 즉 **이미지를 만들어서 DockerHub에 올리기만 했을 뿐, 서버에서 그 이미지를 pull해서 실행하는 부분은 아직 미구현**. 서버 IP도 `<서버IP>`로 마스킹해야 할 값이 워크플로우 파일에 평문으로 남아있는 상태
- 다음 단계로 필요한 작업: SSH 접속 이후 블록을 `docker pull` + `docker run`(또는 `docker-compose up -d`)으로 교체, `.env`는 서버 쪽에서도 새로 만들어서 컨테이너에 주입

---

## 4. JWT 로그인/로그아웃 완성 (`BootLastProject`)

### JWTSecurityConfig — 필터체인에 인증 3종 Bean 연결
```java
@Bean
public JWTAuthenticationFilter jwtAuthenticationFilter(
    CustomUserDetailsService uds, JWTAuthenticationProvider provider) {
    return new JWTAuthenticationFilter(uds, provider);
}

@Bean
public SecurityFilterChain securityFilterChain(HttpSecurity http, JWTAuthenticationFilter filter)
        throws Exception {
    http
      .csrf(csrf -> csrf.disable())
      .sessionManagement(session ->
          session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
      .formLogin(form -> form.disable())
      .httpBasic(basic -> basic.disable())
      .authorizeHttpRequests(auth -> auth
          .requestMatchers("/", "/login", "/member").permitAll()
          .requestMatchers("/admin").hasRole("ADMIN")
          .anyRequest().permitAll())
      .addFilterBefore(filter, UsernamePasswordAuthenticationFilter.class);
    return http.build();
}

@Bean
public PasswordEncoder passwordEncoder() {
    return new BCryptPasswordEncoder();
}

@Bean
public AuthenticationManager authenticationManager(AuthenticationConfiguration config)
        throws Exception {
    return config.getAuthenticationManager();
}
```
- Day19까지는 `SecurityFilterChain` 골격만 있고 필터가 체인에 연결되지 않은 상태였는데, 오늘 `addFilterBefore(filter, UsernamePasswordAuthenticationFilter.class)`로 `JWTAuthenticationFilter`를 실제로 끼워 넣음
- `sessionCreationPolicy(STATELESS)`로 세션을 아예 안 쓰겠다고 선언 — JWT 무상태 인증의 핵심 설정
- `AuthenticationManager`는 직접 구현하지 않고 `AuthenticationConfiguration`에서 꺼내 쓰는 방식(스프링 시큐리티가 기본 제공하는 매니저를 그대로 사용)

### JWTAuthenticationFilter — 헤더/쿠키에서 토큰 추출 후 인증 처리
```java
protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain chain)
        throws ServletException, IOException {
    String token = null;

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
        UserDetails user = uds.loadUserByUsername(username);
        UsernamePasswordAuthenticationToken auth =
                new UsernamePasswordAuthenticationToken(user, null, user.getAuthorities());
        SecurityContextHolder.getContext().setAuthentication(auth);
    }
    chain.doFilter(request, response);
}
```
- 토큰을 찾는 순서는 1순위 `Authorization: Bearer ...` 헤더(Vue/React 같은 SPA용), 2순위 `accessToken` 쿠키(Thymeleaf 서버 렌더링용) — 두 클라이언트 방식을 모두 지원하기 위한 설계
- 토큰이 유효하면 `UsernamePasswordAuthenticationToken`을 직접 만들어 `SecurityContextHolder`에 넣어줌 — 이 지점부터 이후 필터/컨트롤러에서 `Authentication` 객체로 로그인 여부를 확인할 수 있음

### MemberRestController — 로그인/로그아웃 REST API
```java
@RequestMapping("/member/login_ok")
public ResponseEntity<?> login(@RequestParam String username, @RequestParam String password) {
    try {
        Authentication auth = manager.authenticate(
                new UsernamePasswordAuthenticationToken(username, password));
        UserDetails user = (UserDetails) auth.getPrincipal();
        String role = user.getAuthorities().iterator().next().getAuthority();
        String token = provider.createToken(username, role);

        ResponseCookie cookie = ResponseCookie.from("accessToken", token)
                .httpOnly(true).secure(false).path("/").maxAge(3600).build();

        return ResponseEntity.status(HttpStatus.FOUND)
                .header(HttpHeaders.SET_COOKIE, cookie.toString())
                .header(HttpHeaders.LOCATION, "/")
                .build();
    } catch (AuthenticationException ex) {
        return ResponseEntity.status(HttpStatus.FOUND)
                .header(HttpHeaders.LOCATION, "/member/login?error=true")
                .build();
    }
}

@GetMapping("/member/logout")
public ResponseEntity<Void> logout() {
    ResponseCookie cookie = ResponseCookie.from("accessToken", "")
            .httpOnly(true).secure(false).path("/").maxAge(0).build();
    return ResponseEntity.status(HttpStatus.FOUND)
            .header(HttpHeaders.SET_COOKIE, cookie.toString())
            .header(HttpHeaders.LOCATION, "/")
            .build();
}
```
- `AuthenticationManager.authenticate()`로 아이디/비밀번호를 검증 → 성공하면 `JWTAuthenticationProvider.createToken()`으로 토큰 발급 → `httpOnly` 쿠키(`accessToken`)에 담아 응답
- `BadCredentialsException`/`AuthenticationException` 등 인증 실패는 302로 로그인 화면(`?error=true`)으로 되돌려보냄 — 화면에서 `th:if="${param.error}"`로 실패 메시지 표시
- 로그아웃은 별도 토큰 무효화 로직 없이 `maxAge(0)`으로 쿠키만 즉시 만료시키는 방식(클라이언트 쿠키 삭제로 처리하는 가장 단순한 형태)

### CustomUserDetailsService — DB 조회 + 예외 처리 완성
```java
public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
    MemberVO member = mService.findByUsername(username);
    if (member == null) {
        throw new UsernameNotFoundException("사용자를 찾을 수 없습니다 :" + username);
    }
    if (member.getEnabled() != 1) {
        throw new UsernameNotFoundException("비활성화된 계정입니다");
    }
    List<AuthorityVO> authList = mService.getAuthorityData(member.getMember_id());
    List<SimpleGrantedAuthority> authorities = authList.stream()
            .map(a -> new SimpleGrantedAuthority(a.getAuthority()))
            .toList();

    return User.builder()
            .username(member.getUsername())
            .password(member.getPassword())
            .authorities(authorities)
            .build();
}
```
- Day19에서는 빈 클래스였던 `CustomUserDetailsService`를 오늘 `UserDetailsService`를 구현하는 형태로 완성
- 존재하지 않는 아이디, 비활성화 계정(`enabled != 1`)을 각각 별도 예외 메시지로 구분
- 권한 목록(`AuthorityVO`)을 스프링 시큐리티가 요구하는 `SimpleGrantedAuthority`로 변환하는 게 이 클래스의 핵심 역할

### 화면 연동 — 로그인 상태에 따른 분기
- `MainController`에서 `Authentication auth`를 받아 로그인 여부(`isLogin`)와 사용자명·권한을 모델에 담아 `main.html`로 전달
- `main.html`은 `th:if="${!isLogin}"`이면 로그인 버튼, `th:if="${isLogin}"`이면 사용자명+로그아웃 버튼을 표시
- `recipe/main/header.html`에는 Thymeleaf Security 확장(`xmlns:sec="http://www.thymeleaf.org/extras/spring-security"`)을 추가해 `sec:authorize="isAuthenticated()"`로 로그인 사용자만 보이는 메뉴(냉장고 관리/식단플래너)를 구성
- 로그인 화면(`member/login.html`) + 전용 CSS(`login.css`) 신규 추가 — Bootstrap 기반 카드형 로그인 폼, 아이디/비밀번호 입력 + 로그인 실패/로그아웃 알림 메시지(`param.error`/`param.logout`) 처리

---

## 5. 비교표 — 배포 파이프라인 진행 상태 (Day19 → Day20)

| 구분 | Day19 | Day20 |
|------|-------|-------|
| CI(빌드) | `./gradlew clean build -x test`로 jar만 생성 | 동일 + Docker 이미지 빌드 단계 추가 |
| 이미지 관리 | 없음(Dockerfile만 준비, 미사용) | DockerHub 로그인 → 이미지 push까지 완료 |
| CD(서버 배포) | `rsync`로 jar 전송 → SSH 접속 후 `nohup java -jar` | 아직 Day19와 동일한 방식 그대로(전환 미완료) |
| 서버 준비물 | Java 21만 설치 | Java 21 + Docker + Docker Compose 설치 완료 |

---

## 6. 다시 만들 때 체크리스트

```text
[서버에 Docker 설치]
① apt 저장소 등록 전, GPG 키를 dearmor로 변환해서 키링으로 먼저 등록
② docker-ce/docker-ce-cli/containerd.io 설치 후 systemctl enable docker로 부팅시 자동 실행 설정
③ docker-compose는 GitHub Releases에서 실행 파일을 직접 받아 /usr/local/bin에 설치
④ usermod -aG docker <계정>으로 그룹 추가해야 sudo 없이 docker 명령 사용 가능(재로그인 필요)

[GitHub Actions Docker 단계]
⑤ .env를 echo로 여러 줄 생성할 때 각 줄의 키 이름(DB_URL/DB_USERNAME/DB_PASSWORD)을 실제로 다르게 쓰는지 반드시 확인 — 복붙 후 변수명 안 고치는 실수가 나오기 쉬움
⑥ docker build/push는 secrets.DOCKERHUB_USERNAME/PASSWORD로 로그인한 뒤 진행
⑦ CI(빌드+이미지 push) 단계만 먼저 만들고 CD(서버에서 pull+run) 단계는 이후 별도로 구현해도 됨 — 다만 워크플로우 파일 안에 예전 방식(rsync+nohup)이 그대로 남아있으면 실제 배포는 여전히 예전 방식으로 동작한다는 점 주의
⑧ 서버 IP는 워크플로우 파일에도 평문으로 남기지 말고 <서버IP>로 마스킹(또는 Secrets로 분리)

[JWT 로그인/로그아웃]
⑨ SecurityFilterChain에 addFilterBefore(jwtFilter, UsernamePasswordAuthenticationFilter.class)로 필터를 실제로 연결해야 인증이 동작
⑩ 로그인 성공 시 AuthenticationManager.authenticate()로 검증 → JWT 발급 → httpOnly 쿠키에 담아 응답
⑪ 로그아웃은 별도 서버측 무효화 없이 동일한 쿠키를 maxAge(0)으로 재발급해서 만료시키는 방식으로 구현 가능
⑫ CustomUserDetailsService에서 계정 없음/비활성화 계정을 구분해서 예외 처리, 권한은 SimpleGrantedAuthority로 변환
⑬ 화면에서는 Authentication 객체 유무로 로그인 상태 분기(th:if), Thymeleaf에서 sec:authorize를 쓰려면 xmlns:sec 네임스페이스 선언 필요
```

