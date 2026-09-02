# 📘 Spring Boot Day 19 — AWS EC2 서버 구축 + BootLastProject 초기화 + rsync 배포 파이프라인 + JWT 인증 골격

## 0. 핵심 빠른 참조 — 오늘 새로 시작한 것 4가지

| 구분 | 내용 |
|------|------|
| 서버 준비 | EC2 인스턴스(cicdServer) 생성 + 보안그룹 포트 개방(8080/9090/9092) + GitHub Actions 인증용 SSH 키 페어 등록 |
| 신규 프로젝트 | `BootLastProject` 초기 스캐폴딩(Edu/Recipe/Main 3개 도메인 뼈대) |
| 배포 방식 | GitHub Actions로 jar 빌드 → `rsync`로 서버 전송 → SSH 접속 후 `nohup java -jar`로 직접 실행 (Docker 미사용) |
| 인증 골격 | Day18에서 익힌 JWT 발급/검증 로직을 새 프로젝트에 이식 (`JWTAuthenticationProvider`) |

---

## 1. BootLastProject 초기 스캐폴딩

```text
com.sist.web
 ├─ aop         : EduAspect, RecipeAspect (골격만, 내용 없음)
 ├─ commons     : ControllerException, RestControllerException
 ├─ config      : JWTSecurityConfig, WebSocketConfig
 ├─ controller  : EduController, MainController, RecipeController
 ├─ manager     : EduTask, RecipeTask
 ├─ mapper      : EduMapper, RecipeMapper
 ├─ security    : JWTAuthenticationFilter, JWTAuthenticationProvider
 ├─ service     : EduService/Impl, RecipeService/Impl
 └─ vo          : CourseVO, RecipeVO
```

- 이전에 진행하던 `SpringPiniaProject_2`와 별개로 새 프로젝트를 만들면서 Edu(교육)·Recipe(레시피)·Main 세 도메인 컨트롤러/서비스/매퍼 뼈대를 한 번에 잡아둠
- `build.gradle`에 Spring Boot 4.1.1 기준으로 JPA, JDBC, Thymeleaf, WebMVC, WebSocket, MyBatis, JWT(jjwt), Oracle JDBC 드라이버, thymeleaf-layout-dialect 의존성을 한꺼번에 추가
- `.gitattributes`로 `gradlew`는 LF, `*.bat`는 CRLF, `*.jar`는 binary로 줄바꿈/파일타입 처리 규칙을 명시

---

## 2. AWS EC2 인스턴스 생성 및 서버 준비

### 2-1. 인스턴스 생성 설정값

| 항목 | 값 |
|------|-----|
| 이름 | cicdServer |
| AMI | 우분투 26.04 |
| 인스턴스 유형 | small |
| 키페어 | cicdKey (유형 ED25519, PuTTY용 .ppk로 다운로드) |
| 네트워크 | HTTP 트래픽 허용 |
| 스토리지 | 15GiB, 파일 시스템 없음 |

- 인스턴스 시작 후 상태 체크 화면에서 퍼블릭 IP 확인 가능 — 이후 모든 접속/워크플로우 설정에 이 IP를 사용(노트에는 `<서버IP>`로 마스킹)

### 2-2. 보안그룹 인바운드 규칙

인스턴스 생성 시 자동으로 만들어진 launch-wizard 보안그룹에 인바운드 규칙 추가:

| 유형 | 포트 | 소스 | 용도 |
|------|------|------|------|
| Oracle-RDS | - | Anywhere-IPv4 | DB 접속 |
| 사용자 지정 TCP | 8080 | Anywhere-IPv4 | 애플리케이션(Spring Boot) |
| 사용자 지정 TCP | 9090 | Anywhere-IPv4 | Jenkins |
| 사용자 지정 TCP | 9092 | Anywhere-IPv4 | Kafka |

- 보안그룹에서 포트를 미리 열어두지 않으면 서버 안에서 프로세스가 정상 기동돼도 외부에서 접속 자체가 막힘 — 애플리케이션이 쓰는 포트는 인스턴스 생성 직후 한 번에 정리해두는 게 편함

### 2-3. PuTTY 접속 및 서버 초기 세팅

- PuTTY에서 호스트에 `<서버IP>` 입력, 세션 이름 ubuntu로 저장
- Connection > SSH > Auth 항목에서 다운로드한 키페어(.ppk) 등록
- 로그인 계정은 AMI 기본 사용자인 ubuntu
- 접속 후 패키지/JDK 설치:
  ```bash
  sudo apt update && sudo apt upgrade
  sudo apt install openjdk-21-jdk -y
  ```
- JAVA_HOME 경로(`/usr/lib/jvm/java-21-openjdk-amd64`) 확인해두고 `.bashrc` 맨 아래에 등록:
  ```bash
  export JAVA_HOME=/usr/lib/jvm/java-21-openjdk-amd64
  export PATH=$PATH:$JAVA_HOME/bin
  ```
  등록 후 `source ./.bashrc`로 현재 세션에 즉시 반영
- `mkdir app`으로 jar 파일을 받을 배포 디렉토리 미리 생성(GitHub Actions의 rsync 전송 대상 경로)

### 2-4. GitHub Actions 인증용 SSH 키 페어 생성 및 등록

- SSH는 공개키(서버 쪽 `authorized_keys`)와 짝이 맞는 비공개키를 가진 쪽만 인증을 통과시키는 구조 — GitHub Actions가 이 서버에 접속하려면 GitHub 쪽에 비공개키를, 서버 쪽에 공개키를 각각 등록해야 함
- 서버에서 `.ssh` 디렉토리로 이동해 기존 `authorized_keys` 확인:
  ```bash
  cd .ssh
  sudo cat authorized_keys
  ```
- GitHub Actions 전용 키 페어를 새로 생성:
  ```bash
  ssh-keygen -t ed25519 -C "<이메일>"
  ```
  (별도 파일명·패스프레이즈 없이 엔터로 기본값 진행)
- 생성된 공개키를 `authorized_keys`에 추가:
  ```bash
  cat id_ed25519.pub >> authorized_keys
  ```
- 비공개키(`cat id_ed25519`)는 `-----BEGIN OPENSSH PRIVATE KEY-----` ~ `-----END OPENSSH PRIVATE KEY-----` 전체를 복사해서 별도로 보관 — 공개키와 비공개키가 한 쌍으로 연결돼야 인증이 통과됨
- GitHub 저장소 Settings → Secrets and variables → Actions에서 `SERVER_SSH_KEY`라는 이름으로 위 비공개키 전체를 등록 → 워크플로우에서 `${{ secrets.SERVER_SSH_KEY }}`로 참조(3장 GitHub Actions 워크플로우에서 사용)

---

## 3. GitHub Actions 배포 워크플로우 (`.github/workflows/deploy.yml`)

### 전체 흐름
```text
push to main
  → Checkout
  → JDK 21 설치
  → gradlew 실행 권한 부여(chmod +x)
  → ./gradlew clean build -x test
  → SSH 키 준비(secrets.SERVER_SSH_KEY → ~/.ssh/id_ed25519, chmod 600)
  → ssh-agent로 SSH 키 등록
  → known_hosts에 서버 등록(ssh-keyscan)
  → rsync로 jar 파일을 서버로 전송
  → SSH 접속해서 기존 프로세스 kill → 서버의 .env 로드 → nohup java -jar로 재실행
```

### 작업 중 고친 문법 실수들
- `ssh-private-key:${{secrets.SERVER_SSH_KEY}}` → 콜론 뒤 공백 누락으로 YAML 파싱 깨짐 → `ssh-private-key: ${{secrets.SERVER_SSH_KEY}}`로 수정
- heredoc(`<< 'EOF'`) 종료 태그 들여쓰기 문제 — `EOF`가 앞에 공백이 있으면 종료 태그로 인식되지 않아 스크립트가 안 끝남 → 들여쓰기 제거해서 열이 맞도록 수정

### 민감정보 처리 — 노출했다가 되돌린 사례
1. 처음엔 워크플로우 파일 상단에 `env:`로 `DB_URL`/`DB_USERNAME`/`DB_PASSWORD`를 직접 선언해서 추가함
2. 곧바로 "레포에 DB 계정정보가 워크플로우 파일에 그대로 노출된다"는 걸 인지하고 해당 `env:` 블록 전체를 삭제
3. 대신 서버 쪽 `~/app/.env` 파일을 SSH 접속 후 `source`로 읽어들이는 방식으로 전환 (`set -a` / `source ~/app/.env` / `set +a`)
   - `set -a`는 이후 선언되는 변수가 자동으로 export되도록 하는 옵션 — `.env` 파일 안의 변수들을 자식 프로세스(java)까지 넘기기 위해 필요
4. 서버 접속 주소(`<서버IP>`)는 known_hosts 등록과 rsync 대상 양쪽에 하드코딩돼 있음 — Secrets로 옮기는 건 다음 과제로 남음

---

## 4. JWT 인증 골격 이식 (`security`, `service`, `mapper`, `vo`)

### JWTSecurityConfig — 필터체인
```java
@Configuration
@EnableWebSecurity
public class JWTSecurityConfig {
    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
          .csrf(csrf -> csrf.disable())
          .formLogin(form -> form.disable())
          .httpBasic(basic -> basic.disable())
          .authorizeHttpRequests(auth -> auth
              .requestMatchers("/", "/login", "/member").permitAll()
              .requestMatchers("/admin").hasRole("ADMIN")
              .anyRequest().permitAll()
          );
        return http.build();
    }
}
```
- `formLogin`/`httpBasic`을 모두 끄고 커스텀 인증(JWT)만 쓰겠다는 선언
- `anyRequest().permitAll()`이라 아직 실제로 인증을 강제하는 경로는 없음 — 필터체인 골격만 잡아둔 상태(다음 단계에서 `JWTAuthenticationFilter`를 체인에 끼워 넣어야 실제로 동작)

### JWTAuthenticationProvider — 토큰 발급/파싱/검증
```java
private final String SECRET = "<시크릿키>";

public String createToken(String username, String role) {
    return Jwts.builder()
            .setSubject(username)
            .claim("role", role)
            .setIssuedAt(new Date())
            .setExpiration(new Date(System.currentTimeMillis() + 3600000)) // 1시간
            .signWith(Keys.hmacShaKeyFor(SECRET.getBytes()))
            .compact();
}

public String getUsername(String token) {
    return Jwts.parserBuilder()
            .setSigningKey(SECRET.getBytes())
            .build()
            .parseClaimsJws(token)
            .getBody()
            .getSubject();
}

public boolean validate(String token) {
    try {
        Jwts.parserBuilder().setSigningKey(SECRET.getBytes()).build().parseClaimsJws(token);
        return true;
    } catch (Exception ex) {
        return false;
    }
}
```
- Day18(`SpringJWTSecurityProject_1/2`)에서 익힌 발급/파싱/검증 3종 메서드를 그대로 새 프로젝트에 옮겨옴
- 만료 시간 계산은 `현재시각 + 3600000ms(=1시간)`

### Member/Authority 조회 계층
```java
@Mapper
@Repository
public interface MemberMapper {
    @Select("SELECT username,password,enabled,sex,name,member_id FROM member WHERE username=#{username}")
    public MemberVO findByUsername(String username);
}

@Mapper
@Repository
public interface AuthorityMapper {
    @Select("SELECT no,member_id,authority FROM authority WHERE member_id=#{member_id}")
    public List<AuthorityVO> getAuthorityData(int member_id);
}
```
- `MemberService`/`MemberServiceImpl`이 두 Mapper를 감싸서 Controller에는 Service만 노출하는 구조(Controller-Service-Mapper 3계층)
- `CustomUserDetailsService`는 아직 빈 클래스로만 생성됨(다음 작업 예정)
- `MemberVO`/`AuthorityVO`에 `@Data`(Lombok) 적용 — getter/setter/toString을 어노테이션 하나로 대체

---

## 5. 프론트엔드 CDN 확장 + Dockerfile

- `edu/main/main.html`, `recipe/main/main.html`에 jQuery, Bootstrap, Vue3, Axios, vue-demi, Pinia, 카카오맵 SDK를 한 번에 CDN으로 추가
  - 카카오맵 SDK appkey는 `<API키>`로 마스킹해서 관리해야 함(원본 커밋에는 평문으로 들어감 — 다음에 환경변수/서버사이드 주입 방식 검토 필요)
- `Dockerfile` 신규 추가:
  ```dockerfile
  FROM eclipse-temurin:21-jdk
  WORKDIR /app
  COPY build/libs/*-SNAPSHOT.jar app.jar
  EXPOSE 8080
  ENTRYPOINT ["java","-jar","app.jar"]
  ```
  - 다만 오늘 완성한 배포 워크플로우는 이 Dockerfile을 아직 쓰지 않고 jar를 직접 `nohup java -jar`로 실행하는 방식 — Dockerfile은 미리 준비만 해둔 상태

---

## 6. 비교표

### 6-1. 배포 방식 비교 (Day06/Day17 Docker 배포 vs 오늘 rsync+jar 직접 실행)

| 구분 | Day06/Day17까지의 Docker 배포 | 오늘(BootLastProject) rsync 배포 |
|------|-------------------------------|-----------------------------------|
| 전달 대상 | Docker 이미지(build→tag→push→pull→run) | jar 파일 자체(`rsync`로 직접 전송) |
| 서버에서 실행 | `docker run`으로 컨테이너 기동 | `nohup java -jar`로 프로세스 직접 실행 |
| 환경변수 주입 | `docker run -e` 또는 `.env`+Compose | SSH 접속 후 `.env`를 `source`로 셸에 직접 로드 |
| 격리 수준 | 컨테이너 단위로 격리 | 서버에 직접 프로세스로 떠 있음(격리 없음) |
| 재배포 시 | 컨테이너 재생성 | 기존 프로세스 `pkill` 후 재실행 |

### 6-2. JWT 구현 비교 (Day18 전용 실습 프로젝트 vs 오늘 실전 프로젝트 이식)

| 구분 | Day18 (`SpringJWTSecurityProject_1/2`) | 오늘 (`BootLastProject`) |
|------|------------------------------------------|-----------------------------|
| 목적 | JWT 자체를 배우기 위한 전용 실습 프로젝트 | 기존에 만들던 서비스에 인증 기능을 얹는 실전 적용 |
| 필터 체인 연동 | `JwtAuthenticationFilter`까지 체인에 등록 완료 | `SecurityFilterChain`만 구성, 필터는 아직 미연결 |
| 인증 진입점 | `AuthController`가 로그인 처리 전담 | 아직 로그인 Controller 없음(Mapper/Service 계층까지만 준비) |
| 상태 | 무상태 인증 전체 흐름 완성 | 토큰 발급/검증 유틸 + DB 조회 계층까지만 완성, 로그인 흐름 미완성 |

---

## 7. 다시 만들 때 체크리스트

```text
[AWS EC2 인스턴스 준비]
① 인스턴스 생성 시 AMI/인스턴스 유형/키페어(ED25519)/스토리지를 한 번에 설정
② 보안그룹 인바운드 규칙에 애플리케이션이 쓰는 포트(8080, Jenkins 9090, Kafka 9092)를 미리 열어둘 것 — 안 열어두면 프로세스가 떠 있어도 외부 접속 불가
③ PuTTY 접속 계정은 AMI 기본 사용자(우분투는 ubuntu) 고정, 개인키(.ppk)는 Auth 항목에 등록
④ JDK 설치 후 JAVA_HOME은 .bashrc에 직접 등록하고 source로 즉시 반영
⑤ GitHub Actions가 서버에 접속하려면 서버용으로 새로 만든 키 페어의 공개키는 authorized_keys에, 비공개키는 GitHub Secrets(SERVER_SSH_KEY)에 각각 등록해야 함
⑥ 서버 IP는 노트·워크플로우 어디에도 평문으로 남기지 말고 <서버IP>로 마스킹

[신규 프로젝트 스캐폴딩]
⑦ build.gradle에 필요한 스타터(JPA/JDBC/Thymeleaf/WebMVC/WebSocket/MyBatis/JWT) 한 번에 추가
⑧ .gitattributes로 gradlew(LF)/bat(CRLF)/jar(binary) 줄바꿈 규칙 명시

[GitHub Actions rsync 배포]
⑨ SSH 키는 secrets.SERVER_SSH_KEY로 등록 → ~/.ssh/id_ed25519로 저장 후 chmod 600
⑩ ssh-agent 액션 사용 시 `key: value` 형식에서 콜론 뒤 공백 빠뜨리지 않기
⑪ heredoc(`<< 'EOF'`) 사용 시 종료 태그(`EOF`) 앞에 공백 넣지 않기(들여쓰기하려면 `<<-'EOF'` 사용)
⑫ DB 계정정보 같은 민감정보는 워크플로우 파일에 env로 직접 쓰지 말고, 서버의 .env를 SSH 접속 후 source로 로드
⑬ 서버 IP·API 키는 항상 마스킹해서 기록(<서버IP>, <API키>)

[JWT 골격 이식]
⑭ SECRET 키 하드코딩 → HMAC 서명(Keys.hmacShaKeyFor)으로 토큰 생성
⑮ createToken/getUsername/validate 3종 메서드가 JWT 처리의 최소 단위
⑯ SecurityFilterChain만 만들어둔 상태로는 인증이 강제되지 않음 — JwtAuthenticationFilter를 체인에 addFilterBefore로 끼워 넣어야 실제 동작
⑰ MemberVO/AuthorityVO에 @Data(Lombok) 붙여서 보일러플레이트 제거
```
