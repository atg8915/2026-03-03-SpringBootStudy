<div align="center">

# 🌱 Spring Boot Study Log

<br>

![Spring Boot](https://img.shields.io/badge/Spring_Boot-6DB33F?style=flat-square&logo=springboot&logoColor=white)
![Java](https://img.shields.io/badge/Java_21-007396?style=flat-square&logo=java&logoColor=white)
![JPA](https://img.shields.io/badge/JPA-59666C?style=flat-square&logo=hibernate&logoColor=white)
![MyBatis](https://img.shields.io/badge/MyBatis-000000?style=flat-square&logo=mybatis&logoColor=white)
![Thymeleaf](https://img.shields.io/badge/Thymeleaf-005F0F?style=flat-square&logo=thymeleaf&logoColor=white)
![Oracle](https://img.shields.io/badge/Oracle-F80000?style=flat-square&logo=oracle&logoColor=white)
![Vue.js](https://img.shields.io/badge/Vue.js-4FC08D?style=flat-square&logo=vuedotjs&logoColor=white)
![Git](https://img.shields.io/badge/Git-F05032?style=flat-square&logo=git&logoColor=white)

<br>

> 국비 교육 과정에서 진행한 **Spring Boot** 학습 내용을  
> 매일 직접 정리한 기록입니다.  
> 단순 코드 복사가 아닌, 개념·구조·흐름을 이해하고 **다시 만들 수 있도록** 작성했습니다.

<br>

![progress](https://img.shields.io/badge/진행률-진행중-yellow?style=flat-square)
![days](https://img.shields.io/badge/학습일수-진행중-blue?style=flat-square)

</div>

---

## 🔁 기술 스택 흐름

```
Spring Boot
  ├─ JPA (Hibernate) → Entity / Repository / save / delete
  ├─ MyBatis → Mapper XML / 동적 SQL
  ├─ Thymeleaf → HTML 템플릿 엔진 (JSP 대체)
  └─ application.yml → 전체 설정 통합

이전(JSP MVC) → 현재(Spring Boot) 비교
  DispatcherServlet 직접 구현  →  Spring 자동 처리
  web.xml + context.xml        →  application.yml
  DAO 클래스 직접 작성          →  JpaRepository 인터페이스만 작성
  war → Tomcat 배포             →  jar 단독 실행 (내장 서버)
```

---

## 📖 학습 기록

| 일차 | 주제 | 링크 |
|:----:|------|:----:|
| Day 01 | Spring Boot 구조(build.gradle/application.yml), JPA(Entity/Repository/save/delete), Thymeleaf(th:each/th:href/th:if), 가상DOM 개념, MVC→Boot 비교 | [📄](./springboot_day01.md) |
| Day 02 | AOP(@Around 공통로그), 전역예외처리(@ControllerAdvice), Food(Thymeleaf 서버렌더링) vs Goods(Vue+Axios REST) CRUD 비교, 우분투 서버 세팅, Docker 배포(build→tag→push→pull→run) | [📄 보기](./springboot_day02.md) |
| Day 03 | Thymeleaf 레이아웃 include 패턴(main_html 스위칭), VO 프로젝션+nativeQuery 페이징, Controller 역할 분리(@Controller vs @RestController+CrossOrigin), Vuex store(state/mutation/action), CI/CD 개념, Docker Compose 설치 | [📄 보기](./springboot_day03.md) |
| Day 04 | Recipe/Chef 목록+페이징(getPageData에 rowsize 매개변수화, pages[] 배열 통일), 레이아웃 재사용, Docker Compose(up -d/down)로 배포 간소화 | [📄 보기](./springboot_day04.md) |
| Day 05 | Recipe 검색(제목/쉐프)+상세보기(RecipeDetail 분리, foodmake 파싱), Thymeleaf+Vue 혼합 렌더링 패턴, Git Actions self-hosted runner 자동배포(+RUNNER/파일명 일치 교훈) | [📄 보기](./springboot_day05.md) |
| Day 06 | 파일 업로드 3종(HTML/Vue+FormData/jQuery 동적행), DataBoard 첨부파일 게시판(MyBatis), multipart 용량 설정, Docker 권한(usermod), Git Actions 배포를 jar 실행→Docker 컨테이너 실행으로 전환 | [📄 보기](./springboot_day06.md) |
| Day 07 | Emp/Dept 검색 3방식 비교(메소드규칙/JPQL/QueryDSL), Q-class 생성+JPAQueryFactory Bean 등록, QueryDSL 연산자·조인·Top-N 총정리, application.yml DB정보 환경변수 분리 | [📄 보기](./springboot_day07.md) |
| Day 08 | Recipe JPA→MyBatis 전환(Mapper+XML), CDN 방식 Vue+Pinia(빌드 없이 Thymeleaf에 삽입), th:inline으로 서버값 전달, Docker 환경변수 주입 3단계(/etc/environment → docker run -e → .env), IP/계정정보 항상 마스킹하는 습관 | [📄 보기](./springboot_day08.md) |
| Day 09 | `SpringPiniaProject_2` 신규 프로젝트 초기 스캐폴딩(Food/Recipe/Chef 재시작), MyBatis 어노테이션+XML 혼용 패턴, main.html에 Vue3+axios+vue-demi+Pinia CDN 연동, Dockerfile/.gitattributes 추가, application.yml DB정보 환경변수 유지 | [📄 보기](./springboot_day09.md) |
| Day 10 | Spring Security 로그인 골격 도입(SecurityConfig 필터체인/csrf-disable/remember-me/logout), MemberVO·AuthorityVO 신규, login.html/join.html 화면 추가, header.html에 sec:authorize 네임스페이스 준비 — AuthenticationManager 등 인증 Bean 3종은 아직 null(다음 작업 예정) | [📄 보기](./springboot_day10.md) |
| Day 11 | `SpringPiniaProject_2` Security 인증 3종 Bean 완성(AuthenticationManager/JdbcUserDetailsManager/PersistentTokenRepository), login.html form 추가로 실제 로그인 가능, Docker Compose+GitHub Actions 배포 파이프라인 재정비 / 신규 실습 `SpringSecurityProject_1`(인메모리 UserDetailsService)·`SpringSecurityProject_2`(DB연동 CustomUserDetailsService, failureHandler 연결 누락 버그 발견)·`LamdaProject`(람다식+Stream 문법) | [📄 보기](./springboot_day11.md) |
| Day 12 | 미니 쿠버네티스(Minikube) 배포 실습(kubectl/Docker 드라이버 설치, deployment.yaml로 Deployment+NodePort Service 작성, imagePullPolicy Never로 로컬 이미지 배포) / `SpringPiniaProject_2` 댓글 기능 신규(Comment VO/Mapper/Service/RestController), food 상세 화면 카카오맵 연동, Cookie 기반 최근 본 상품, header.html MANAGER 역할 추가 | [📄 보기](./springboot_day12.md) |
| Day 13 | `SpringPiniaProject_2` 댓글 수정/삭제 API 추가로 CRUD 완성(PUT/DELETE + Pinia 스토어 확장), 작성자 본인만 보이는 수정/삭제 버튼과 인라인 수정 textarea, MyBatis `@Param` import 오류·응답 Map 키 버그 수정 / Minikube 핵심 용어(클러스터·노드·파드·kubectl)와 도커 이미지 배포 흐름 개념 정리 | [📄 보기](./springboot_day13.md) |
| Day 14 | WebSocket+STOMP 채팅 3종 신규 실습 — `SpringWebSocket_1`(jQuery 기반 전체 공개 채팅, `/topic/public` 브로드캐스트) · `SpringWebSocket_2`(Pinia 기반 1:1 개인 채팅, `/queue/private/{id}` 지정 수신) · `SpringPiniaProject_2`(로그인 세션 연동으로 sender 위조 방지, 전체+1:1 채팅 통합, header.html 로그인 사용자 전용 채팅 메뉴) | [📄 보기](./springboot_day14.md) |
| Day 15 | `SpringPiniaProject_2` 채팅에 Spring Security 연동 — HttpSession 대신 `Principal`로 sender 확인, `/chat/join`+`Set<String>`으로 접속자 목록 브로드캐스트(연결 이벤트 리스너는 아직 미완성), Pinia `currentRoom`에 roomId 직접 저장하는 방식으로 리팩토링, `sendPublic`/`sendPrivate`/`send()` 메시지 전송 완성, `store.connect` 미호출 버그 수정 / Board 관련 빈 스캐폴딩 파일 신규 생성 / Jenkins 설치(apt 저장소 등록)+ngrok 터널링으로 로컬 서버 외부 노출+GitHub Webhook(`/github-webhook/`) 연동, GitHub Actions self-hosted runner 방식과 비교 | [📄 보기](./springboot_day15.md) |
| Day 16 | `SpringPiniaProject_2` 게시판(Board) JPA로 실제 구현(`BootBoard` Entity+`BootBoardRepository`, 목록 페이징/등록/상세+조회수), 댓글은 MyBatis로 별도 구현(`BoardCommentMapper`+XML, 세션 기반 작성자 세팅), 상세 화면에 Pinia 댓글 위젯 부분 마운트 / `@EnableScheduling`+`@EnableAspectJAutoProxy` 전역 설정 추가(AOP/스케줄링 골격), WebSocket `/sub`·`/pub` prefix 확장 / 댓글 API 경로 불일치 수정, 대댓글(답글) 기능 신규(group_id/group_step/group_tab/depth 조작), STOMP `/sub/notice/{id}`로 실시간 댓글 알림+Toast UI, `Jenkinsfile` 신규 작성(Declarative 문법으로 Git 연결 확인 파이프라인) / 우분투 서버에 Jenkins CI/CD 환경 전체 구축(ngrok 터널 전환, Docker 권한, Docker Hub Credentials, Pipeline Job, GitHub Webhook) | [📄 보기](./springboot_day16.md) |

---

## 💡 학습 포인트

- **매일 직접 코드를 작성**하고 주석 기반으로 개념을 정리
- **"다시 만들 수 있는지"** 를 기준으로 정리
- JSP MVC에서 직접 구현했던 구조가 Spring Boot에서 **어떻게 자동화되는지** 비교하며 학습
- JPA / MyBatis 병행 학습으로 각 기술의 장단점 체감

---

<div align="center">

`Spring Boot` `Java 21` `JPA` `Hibernate` `MyBatis` `Thymeleaf` `Oracle` `Vue.js` `Git`

</div>
