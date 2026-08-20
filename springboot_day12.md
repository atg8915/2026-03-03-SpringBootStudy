# 📘 Spring Boot Day 12 — 미니 쿠버네티스(Minikube) 배포 실습 + SpringPiniaProject_2 댓글 기능 신규 구현

## 0. 핵심 빠른 참조 — Day11 대비 오늘 바뀐 점

| 구분 | Day11까지 | Day12(오늘) |
|------|-----------|--------------|
| 배포 방식 | `docker run` / `docker-compose up -d` | Minikube 클러스터에 `kubectl apply`로 Pod·Service 배포 |
| 로컬 클러스터 | 없음 | Minikube(Docker 드라이버) + kubectl 신규 설치 |
| 매니페스트 | 없음 | `deployment.yaml`(Deployment 2개 replicas + NodePort Service) 작성 |
| `SpringPiniaProject_2` 댓글 기능 | 없음 | `CommentVO`/`CommentMapper`/`CommentService`/`CommentRestController` 신규 + Pinia 스토어 뼈대 |
| food 상세 화면 | 텍스트 위주 정보 나열 | 정보 테이블 + 카카오맵(Kakao Maps SDK) 좌표 마커 연동 |
| 최근 본 상품 | 없음 | `food_detail_before` 라우트에서 Cookie 저장 후 상세로 리다이렉트 |
| header.html 권한 표시 | ADMIN/USER만 분기 | MANAGER 역할 추가, `sec:authentication` 대신 세션 값(`session.username`)으로 이름 출력 |

---

## 1. 미니 쿠버네티스(Minikube) 배포 실습

우분투 환경에서 Minikube로 로컬 쿠버네티스 클러스터를 띄우고, 그동안 Docker로 단독 실행하던 스프링부트 앱을 Pod/Service 형태로 배포하는 과정을 정리함.

### 1-1. Minikube / kubectl 설치

```bash
# Minikube 설치
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube

# kubectl 설치
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```
- Minikube는 로컬 PC(또는 서버) 한 대 안에 가상 쿠버네티스 클러스터를 구성해주는 도구임. 실제 운영에서 쓰는 멀티 노드 클러스터를 그대로 쓰기엔 자원이 부담되므로, 학습·개발 단계에서는 Minikube로 클러스터 API·오브젝트 개념을 먼저 익히는 방식
- kubectl은 클러스터에 명령을 내리는 CLI 도구. Minikube가 "클러스터 자체"라면 kubectl은 "그 클러스터를 조작하는 리모컨"에 해당함
- 두 바이너리 모두 `/usr/local/bin`에 설치해 PATH에서 바로 실행되도록 함

### 1-2. Docker 구동 + Minikube 클러스터 기동

```bash
sudo systemctl start docker
sudo systemctl enable docker

sudo usermod -aG docker <사용자명>
newgrp docker

sudo apt install -y conntrack
sudo apt install -y util-linux-extra

minikube start --driver=docker --force

kubectl version --client
minikube status
kubectl get nodes
```
- Minikube는 여러 드라이버(VM, Docker 등)를 지원하는데, 여기서는 `--driver=docker`로 Docker 컨테이너 위에 클러스터를 올리는 방식을 씀. 별도 VM 없이 기존 Docker 엔진을 그대로 재사용하는 셈
- `usermod -aG docker <사용자명>` + `newgrp docker`는 매번 `sudo` 없이 docker 명령을 쓰기 위한 그룹 권한 설정. `newgrp`으로 그룹 변경을 즉시 반영시킴
- `conntrack`은 쿠버네티스 네트워킹(kube-proxy)이 연결 추적에 사용하는 리눅스 유틸리티라 Minikube 구동 전에 미리 설치해둬야 함
- `kubectl get nodes`로 클러스터에 노드가 잡혔는지 확인하는 게 설치 검증의 마지막 단계

### 1-3. 스프링부트 앱을 Docker 이미지로 빌드 + 단독 컨테이너로 먼저 검증

```bash
# 프로젝트 클론 후
sudo chmod +x gradlew
sudo ./gradlew clean build -x test

docker build -t myapp .

touch .env
sudo nano .env
```
```text
DB_URL=<DB_URL>
DB_USERNAME=<DB_USERNAME>
DB_PASSWORD=<DB_PASSWORD>
```
```bash
docker run --name myapp --env-file .env -d -it -p 9090:9090 myapp
```
- `docker run --name myapp -d -it -p 9090:9090 myapp`처럼 `.env` 없이 바로 띄우면 DB 접속정보가 컨테이너 안에 전달되지 않아 애플리케이션이 정상 기동하지 않음. `application.yml`이 `${DB_URL}` 같은 환경변수 참조로 작성돼 있으므로, 컨테이너 실행 시점에 `--env-file .env`로 값을 채워줘야 함(Day08부터 이어온 원칙과 동일)
- 쿠버네티스 매니페스트를 작성하기 전에, 먼저 Docker 단독 실행으로 이미지 자체가 정상 동작하는지 확인하는 순서로 진행함

### 1-4. `deployment.yaml` 작성 — Deployment + Service

```bash
mkdir ~/k8s
cd ~/k8s
touch deployment.yaml
sudo nano ./deployment.yaml
```
```yaml
apiVersion: apps/v1  # Deployment를 정의할 때 사용하는 표준 API 버전
kind: Deployment      # 애플리케이션의 배포를 관리하는 오브젝트
metadata:
  name: myapp-deployment
spec:
  replicas: 2          # 동일한 Pod를 2개 실행(로드 밸런싱 가능)
  selector:
    matchLabels:
      app: myapp        # 이 Deployment가 관리할 Pod를 라벨로 선택
  template:
    metadata:
      labels:
        app: myapp       # Pod에 붙는 라벨, selector와 반드시 일치해야 함
    spec:
      containers:
        - name: myapp
          image: myapp:latest
          imagePullPolicy: Never  # Docker Hub에서 받지 말고 Minikube에 이미 있는 이미지를 쓰라는 의미
          ports:
            - containerPort: 9090
          env:
            - name: DB_URL
              value: "<DB_URL>"
            - name: DB_USERNAME
              value: "<DB_USERNAME>"
            - name: DB_PASSWORD
              value: "<DB_PASSWORD>"
---
apiVersion: v1
kind: Service          # 클러스터 내외에서 Pod들을 접근 가능하게 해주는 오브젝트
metadata:
  name: myapp-service
spec:
  selector:
    app: myapp          # app: myapp 라벨을 가진 Pod로 트래픽 전달
  ports:
    - port: 80           # 클러스터 내부에서 접근할 서비스 포트
      targetPort: 9090    # Pod 내부 컨테이너가 열어둔 포트
  type: NodePort          # 클러스터 외부에서 노드 IP + 포트로 접속 가능하게 노출
```
- Deployment는 "몇 개의 Pod를 어떤 이미지로 유지할지"를 선언하는 오브젝트이고, Service는 "그 Pod들을 어떻게 네트워크로 노출할지"를 선언하는 별개의 오브젝트임. 하나의 YAML 파일에 `---`로 구분해 같이 작성해도 되고, 파일을 분리해도 됨
- `replicas: 2`로 동일한 Pod가 2개 뜨는데, `selector.matchLabels`와 `template.metadata.labels`가 같은 `app: myapp` 값이어야 Deployment가 자기가 만든 Pod를 제대로 추적함
- `imagePullPolicy: Never`가 핵심 포인트. 기본값대로 두면 쿠버네티스가 Docker Hub 같은 외부 레지스트리에서 이미지를 받아오려 시도하는데, 로컬에서 빌드한 이미지를 그대로 쓰려면 "외부에서 당겨오지 말라"고 명시해야 함
- `type: NodePort`는 ClusterIP(클러스터 내부 전용)보다 한 단계 더 열어서, 클러스터 밖에서도 노드의 IP와 할당된 포트로 접근할 수 있게 해줌

### 1-5. 이미지 로드 + 클러스터 적용 + 서비스 확인

```bash
docker ps -a   # 기존 컨테이너 확인 후 불필요한 것은 정리

kubectl apply -f ~/k8s/deployment.yaml

minikube image load myapp:latest

kubectl delete pod -l app=myapp
kubectl get pods
kubectl get svc

minikube service myapp-service
```
- `kubectl apply -f`로 매니페스트를 클러스터에 반영하지만, Minikube는 자체 Docker 데몬을 따로 갖고 있어서 로컬에서 빌드한 이미지를 그대로 못 보는 경우가 있음. `minikube image load myapp:latest`로 이미지를 Minikube 내부로 옮겨 넣어야 `imagePullPolicy: Never` 설정이 실제로 그 이미지를 찾을 수 있음. 이미지 로드는 메모리를 꽤 먹는 작업이라, VM 램 용량을 넉넉히 잡아두는 편이 나음
- 이미지를 새로 로드한 뒤에는 기존 Pod가 옛날 이미지를 물고 있으므로 `kubectl delete pod -l app=myapp`으로 지워야 Deployment가 새 이미지로 Pod를 재생성함(`-l app=myapp`은 라벨 기준 선택)
- `kubectl get pods`/`kubectl get svc`로 Pod 상태와 Service에 할당된 클러스터 IP·포트를 확인
- `minikube service myapp-service`를 실행하면 NodePort로 열린 서비스에 브라우저로 바로 접속할 수 있는 URL을 열어줌

---

## 2. SpringPiniaProject_2 — 댓글 기능 신규 구현

### 2-1. Comment 계층 구조 — VO / Mapper / Service / RestController

```java
@Data
public class CommentVO {
	private int no,fno,page;
	private String id,name,msg,dbday;
	private Date regdate;
}
```
```java
@Mapper
@Repository
public interface CommentMapper {
	public List<CommentVO> commentListData(@Param("start")int start,@Param("fno")int fno);
	public int commentRowCount(int fno);
	public void commentInsert(CommentVO vo);
}
```
```xml
<select id="commentListData" resultType="com.sist.web.vo.CommentVO" parameterType="int">
  SELECT no,fno,id,name,TO_CHAR(regdate, 'yyyy-mm-dd hh24:mi:ss') as dbday
  FROM piniaComment
  WHERE fno=#{fno}
  ORDER BY no DESC
  OFFSET #{start} ROWS FETCH NEXT 10 ROWS ONLY
</select>
```
```java
@RestController
@RequiredArgsConstructor
public class CommentRestController {
	private final CommentService cService;

	@GetMapping("/comment/list_vue")
	public ResponseEntity<Map> comment_list(@RequestParam("page") int page, @RequestParam("fno") int fno) { ... }

	@PostMapping("/comment/insert_vue")
	public ResponseEntity<Map> comment_insert(@RequestBody CommentVO vo, HttpSession session) {
		String id=(String)session.getAttribute("userid");
		String name=(String)session.getAttribute("username");
		vo.setId(id);
		vo.setName(name);
		cService.commentInsert(vo);
		...
	}
}
```
- 기존 Food/Recipe와 같은 Mapper → Service → RestController 3계층 패턴을 댓글 기능에도 그대로 적용함
- 댓글 작성자 정보(`id`, `name`)는 클라이언트가 보낸 값이 아니라 `HttpSession`에 저장된 로그인 세션 값을 그대로 가져다 씀. 클라이언트가 임의로 다른 사람 이름을 넣어 등록하는 걸 막는 효과가 있음
- 목록 조회는 `OFFSET ... FETCH NEXT`로 페이징 처리하고, `fno`(음식 번호)로 특정 게시물에 달린 댓글만 필터링함
- `CommentServiceImpl`은 Mapper 호출을 그대로 위임하는 얇은 계층으로, 아직 추가 비즈니스 로직은 없는 상태

### 2-2. food 상세 화면 리뉴얼 + 카카오맵 연동

```html
<div id="map" style="width:100%;height:350px;"></div>
<script>
  var mapContainer = document.getElementById('map'),
      mapOption = { center: new kakao.maps.LatLng(33.450701, 126.570667), level: 3 };
  var map = new kakao.maps.Map(mapContainer, mapOption);
  var geocoder = new kakao.maps.services.Geocoder();
  geocoder.addressSearch('[[${vo.address}]]', function(result, status) {
    if (status === kakao.maps.services.Status.OK) {
      var coords = new kakao.maps.LatLng(result[0].y, result[0].x);
      var marker = new kakao.maps.Marker({ map: map, position: coords });
      ...
    }
  });
</script>
```
```html
<script type="text/javascript" src="//dapi.kakao.com/v2/maps/sdk.js?appkey=<API키>&libraries=services"></script>
```
- 상세 화면을 텍스트 나열 대신 정보 테이블(주소/전화/음식종류/가격대/영업시간/주차/테마)로 재구성함
- 카카오맵 SDK는 `appkey` 쿼리 파라미터로 발급받은 API 키를 요구함. `main.html`에 `<script>` 태그로 직접 심는 방식이라, 실제 배포 시에는 키가 클라이언트 소스에 그대로 노출된다는 점을 인지하고 있어야 함(카카오맵 키는 도메인 제한 설정으로 방어하는 방식이 일반적)
- `services.Geocoder`로 주소 문자열을 좌표로 변환(지오코딩)한 뒤, 변환된 좌표에 마커와 인포윈도우를 얹는 흐름. `vo.address`는 서버에서 내려준 값을 Thymeleaf 인라인(`[[${...}]]`)으로 JS 변수 자리에 그대로 꽂아 씀

### 2-3. 최근 본 상품 — Cookie 기반 라우팅 분리

```java
@GetMapping("food/detail_before")
public String food_detail_before(
	@RequestParam("no") int no,
	HttpServletResponse response,
	RedirectAttributes ra
)
{
	Cookie cookie=new Cookie("food_"+no, String.valueOf(no));
	cookie.setPath("/");
	cookie.setMaxAge(60*60*24);
	response.addCookie(cookie);
	ra.addAttribute("no",no);
	return "redirect:/food/detail";
}
```
- 상세 화면으로 바로 이동하지 않고 `food_detail_before`를 한 단계 거치도록 라우트를 분리함. 이 단계에서 Cookie를 심고, 실제 화면 렌더링은 `redirect:/food/detail`로 넘김
- 하나의 메서드 안에서 쿠키 추가(`response.addCookie`)와 모델 데이터 전달(`Model.addAttribute`)을 동시에 처리할 수 없어서, 리다이렉트 시 파라미터를 넘기는 `RedirectAttributes`를 별도로 사용함
- 쿠키 이름을 `food_` + 게시물 번호로 만들어, 상품별로 개별 쿠키가 쌓이는 구조. 유효기간은 하루(`60*60*24`초)로 설정함

### 2-4. SecurityConfig — 인증/인가 흐름 학습 주석 정리

```text
사용자 → /member/login → /member/login_process
  → Security FilterChain
    → UsernamePasswordAuthenticationFilter
        (.usernameParameter("userid") / .passwordParameter("userpwd")로 폼 값 추출)
    → AuthenticationManager → AuthenticationProvider → JdbcUserDetailsManager
        (DB 조회: springmember / authority)
    → UserDetails 생성 → BCryptPasswordEncoder로 비밀번호 비교
    → 성공 시 LoginSuccessHandler / 실패 시 LoginFailHandler
    → SecurityContext → Session에 저장 → 인증 완료
```
- `/member/login_process` 요청은 Controller가 아니라 Security의 `UsernamePasswordAuthenticationFilter`가 가로채서 처리한다는 점이 핵심. Controller 메서드를 따로 만들지 않아도 로그인 처리가 동작하는 이유가 여기 있음
- 인증(Authentication, 로그인/사용자 확인)과 인가(Authorization, 권한 확인)를 구분해서 정리해둠. 필터 체인은 인증까지만 담당하고, 이후 URL별 접근 제어(`hasRole` 등)가 인가 영역
- `LoginSuccessHandler`가 인증 성공 후 `MemberMapper`로 회원 상세정보를 다시 조회해서 세션에 `userid`/`username`/`sex`를 저장하도록 확장됨. 인증 자체는 Security가 처리하지만, 화면에 쓸 부가 정보는 별도로 세션에 채워 넣는 구조

### 2-5. header.html — MANAGER 역할 추가 + 세션 기반 이름 표시

```html
<li sec:authorize="hasRole('MANAGER')">
  <a href="#">
    <span th:text="${session.username}"></span>(부관리자) 로그인되었습니다
  </a>
</li>
```
- 기존 ADMIN/USER 두 역할에 MANAGER(부관리자)를 추가함
- 로그인한 사용자 이름 표시를 `sec:authentication="name"`(Security 인증 객체에서 직접 꺼내는 방식)에서 `th:text="${session.username}"`(세션에 저장해둔 값을 꺼내는 방식)으로 바꿈. `LoginSuccessHandler`에서 세션에 `username`을 채워 넣은 것과 짝을 이루는 변경

---

## 3. 비교표 — 오늘까지 다룬 배포 방식 3단계

| 구분 | `docker run`(Day02~) | `docker-compose`(Day03~) | Kubernetes/Minikube(Day12, 오늘) |
|------|----------------------|---------------------------|-----------------------------------|
| 배포 단위 | 컨테이너 1개 직접 실행 | `docker-compose.yml`로 컨테이너 그룹 관리 | Pod(컨테이너 묶음) + Deployment(Pod 관리) |
| 복제/이중화 | 없음(컨테이너 1개) | 없음(서비스당 보통 1개) | `replicas: 2`로 동일 Pod 여러 개 유지 |
| 환경변수 주입 | `--env-file .env` | `env_file: .env` | Deployment YAML의 `env:` 목록 |
| 재기동 방식 | `docker run` 재실행 | `docker compose down/pull/up -d` | `kubectl delete pod` → Deployment가 자동 재생성 |
| 외부 노출 | `-p 포트:포트` | `ports:` 매핑 | Service(`type: NodePort`)로 별도 오브젝트 분리 |
| 학습 목적 | 컨테이너 기본 개념 | 여러 컨테이너를 정의 파일로 관리 | Pod/Deployment/Service 등 쿠버네티스 오브젝트 모델 이해 |

---

## 4. 다시 만들 때 체크리스트

```text
[Minikube 배포]
① Docker 실행 → usermod로 docker 그룹 권한 부여 → conntrack 설치 → minikube start --driver=docker
② 이미지는 먼저 docker run 단독 실행으로 정상 동작 확인 후 쿠버네티스로 옮길 것
③ deployment.yaml: replicas/selector.matchLabels/template.labels의 app 값을 전부 일치시킬 것
④ 로컬 빌드 이미지를 쓰려면 imagePullPolicy: Never 필수
⑤ kubectl apply 후 minikube image load로 이미지를 Minikube 내부에 반영
⑥ 이미지 갱신 시 kubectl delete pod -l app=<라벨>로 기존 Pod를 지워야 새 이미지로 재생성됨
⑦ Service는 type: NodePort로 열어야 클러스터 밖에서 접근 가능, minikube service <서비스명>으로 접속 URL 확인

[SpringPiniaProject_2 — 댓글 기능]
⑧ 댓글 작성자 정보는 클라이언트 입력이 아니라 HttpSession에서 꺼내 서버가 채워 넣을 것
⑨ 쿠키를 심는 로직과 모델 데이터를 함께 넘기는 로직은 한 메서드에서 동시 처리 불가 → RedirectAttributes로 분리
⑩ 카카오맵 등 외부 SDK API 키는 소스에 그대로 노출되므로, 노트/저장소에 옮길 때는 반드시 마스킹할 것
⑪ Security 인증 객체(sec:authentication)와 세션 값(session.xxx) 중 어느 쪽을 화면 표시에 쓸지는 LoginSuccessHandler에서 세션에 뭘 채워 넣었는지에 따라 결정됨
```
