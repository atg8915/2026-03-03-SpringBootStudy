# 📘 Spring Boot Day 17 — Kafka 알림 파이프라인 도입 + 실시간 랭킹 수집 (SpringPiniaProject_2) / AWS EC2 CI/CD 배포 서버 구축

## 0. 핵심 빠른 참조 — Day16 대비 오늘 바뀐 점

| 구분 | Day16 | Day17 |
|------|-------|-------|
| 댓글 알림 전송 | Controller에서 `SimpMessagingTemplate`로 직접 STOMP 전송 | Controller → Kafka Producer(`notice-topic`) → Kafka Consumer → STOMP 전송으로 한 단계 경유 |
| 실시간 랭킹 수집 | `RealFindWordTask`는 빈 스케줄러 골격만 존재 | Jsoup 크롤링(`DataCollection`)을 연결해 1분 주기로 실제 데이터 수집·출력 |
| 댓글 화면 | 테이블 안에 테이블을 중첩한 목록 UI | 카드형 리스트(`.comment-item`)로 리디자인, 답글 들여쓰기는 `marginLeft` 계산 방식으로 변경 |
| 배포 인프라 | 우분투 로컬 서버에 Jenkins/Docker 구성 | AWS EC2 인스턴스를 새로 띄워 별도 CI/CD 배포 서버 구축(자바21+Docker+Kafka, 실습 후 리소스 정리) |

---

## 1. Kafka 알림 파이프라인 도입

```java
// NoticeProducer.java
@Service
@RequiredArgsConstructor
public class NoticeProducer {
	private final KafkaTemplate<String, ChatMessage> kafkaTemplate;
	private static final String TOPIC="notice-topic";

	public void sendNotice(ChatMessage notice) {
		kafkaTemplate.send(TOPIC, notice.getReceiver(), notice);
	}
}
```

```java
// NoticeConsumer.java
@Service
@RequiredArgsConstructor
public class NoticeConsumer {
	private final SimpMessagingTemplate template;

	@KafkaListener(topics = "notice-topic", groupId = "notice-group")
	public void consumerNotice(ChatMessage notice) {
		String dest="/sub/notice/"+notice.getReceiver();
		template.convertAndSend(dest, notice.getMessage());
	}
}
```

```yaml
# application.yml
spring:
  kafka:
    bootstrap-servers: localhost:9092
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
    consumer:
      group-id: notification-group
      auto-offset-reset: earliest
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.springframework.kafka.support.serializer.JsonDeserializer
    properties:
      spring.json.trusted.packages: com.sist.web.vo
    listener:
      ack-mode: record
```

- `bootstrap-servers`로 접속할 Kafka 브로커 주소를 지정하고 producer/consumer 각각 key/value의 직렬화·역직렬화 방식을 별도로 설정함
- `spring.json.trusted.packages`를 지정하지 않으면 JSON 역직렬화 시 패키지 신뢰 오류가 발생 — VO가 속한 패키지(`com.sist.web.vo`)를 명시적으로 허용해야 함
- `auto-offset-reset: earliest`는 Consumer Group에 저장된 offset이 없을 때(최초 구독 시) 가장 오래된 메시지부터 읽도록 하는 옵션

```java
// BoardCommentRestController.java — 알림 전송 방식 교체
ChatMessage notice = new ChatMessage(
	vo.getId(), pvo.getId(),
	"[☠️댓글 알람]"+vo.getId()+"님이 댓글을 달았습니다!!"
);
noticeProducer.sendNotice(notice);
```

| 구분 | 직전 방식 (Day16) | 오늘 방식 (Kafka 경유) |
|------|--------------------|--------------------------|
| 전송 경로 | Controller → `SimpMessagingTemplate.convertAndSend()` (1단계) | Controller → `NoticeProducer`(Kafka 발행) → `NoticeConsumer`(Kafka 구독) → `SimpMessagingTemplate.convertAndSend()` (3단계) |
| 결합도 | Controller가 STOMP 전송 로직을 직접 소유 | Controller는 메시지 발행만 담당, 실제 STOMP 전송은 Consumer가 담당(관심사 분리) |
| 장점 | 구조가 단순 | 메시지 큐를 거치므로 Consumer가 다운돼도 메시지가 유실되지 않고, 알림 전용 서비스로 분리 확장 가능 |

- `ChatMessage` VO에 `@AllArgsConstructor`/`@NoArgsConstructor`를 추가함 — Kafka가 JSON을 객체로 역직렬화할 때 기본 생성자가 필요하기 때문(없으면 `NoticeConsumer`에서 역직렬화 실패)
- 기존 `template.convertAndSend(...)` 직접 호출 코드는 삭제하지 않고 주석 처리만 해둔 상태 — 롤백 가능성을 열어둔 것으로 보임

---

## 2. 실시간 랭킹 크롤링(Jsoup) — 스케줄러 완성

```java
// DataCollection.java
public static List<RealFindVO> dataCollect() {
	List<RealFindVO> list=new ArrayList<>();
	Document doc=Jsoup.connect("https://rank.ezme.net").get();
	Elements words=doc.select(".rank_word");
	Elements images=doc.select(".rank_img");
	for(int i=0;i<words.size();i++) {
		RealFindVO vo=new RealFindVO();
		vo.setRank(i+1);
		vo.setWord(words.get(i).text());
		vo.setImages(images.get(i).attr("data-pagespeed-lazy-src"));
		list.add(vo);
	}
	return list;
}
```

```java
// RealFindWordTask.java
@Component
public class RealFindWordTask {
	@Async
	@Scheduled(fixedRate = 60*1*1000)   // 기존 3분 → 1분 주기로 단축
	public void task() {
		List<RealFindVO> list = DataCollection.dataCollect();
		for(RealFindVO vo : list) {
			System.out.println("Rank:"+vo.getRank()+" Word:"+vo.getWord());
		}
	}
}
```

- Jsoup으로 외부 사이트(`rank.ezme.net`)의 HTML을 파싱해 `.rank_word`, `.rank_img` 클래스의 요소를 순서대로 추출하는 웹 크롤링 방식
- Day16에서 빈 껍데기였던 `RealFindWordTask.task()`에 실제 크롤링 로직이 연결됐지만 결과를 콘솔에 출력만 하고 DB 저장이나 화면 전달까지는 아직 이어지지 않은 중간 단계
- `MainClass`(임시 테스트용 단독 실행 클래스)로 크롤링 로직만 따로 떼어 먼저 검증해본 흔적이 남아있음
- Day16에서 지적됐던 "`@Async`는 `@EnableAsync` 없이는 동기로 실행된다"는 문제는 이번에도 그대로 남아있는 상태

---

## 3. 댓글 화면 UI 리디자인 — 테이블 목록 → 카드형 리스트

```html
<!-- 카드형 댓글 아이템 -->
<div class="comment-item"
     v-for="(rvo,index) in store.list" :key="index"
     :style="{marginLeft: (rvo.group_tab * 28) + 'px'}">
  <div class="comment-header">
    <div class="comment-user">
      <span v-if="rvo.group_tab>0" class="reply-indent">↳</span>
      <span class="comment-avatar">{{rvo.name ? rvo.name.charAt(0) : '?'}}</span>
      <span class="comment-name">{{rvo.name}}</span>
      <span class="comment-date">{{rvo.dbday}}</span>
    </div>
    <div class="comment-actions">
      <a class="comment-btn comment-reply" :class="{active: store.reReplyNo===rvo.no}"
         v-if="store.sessionId!==''" @click="store.toggleReply(rvo.no)">
        {{store.reReplyNo===rvo.no ? '취소' : '댓글'}}
      </a>
    </div>
  </div>
  <div class="comment-content"><pre>{{rvo.msg}}</pre></div>
</div>
```

- Day16까지 쓰던 `<table>` 중첩 목록(표 안에 표) 구조를 `.comment-item` 카드 리스트로 전면 교체
- 답글 들여쓰기 방식이 바뀜 — Day16까지는 `group_tab` 값만큼 `&nbsp;&nbsp;`를 반복 출력했지만 오늘은 `:style="{marginLeft: rvo.group_tab * 28 + 'px'}"`처럼 계산된 인라인 스타일로 들여쓰기를 표현
- 아바타(이름 첫 글자 원형 배지), 댓글 없음 상태 이모지(💬), 입력창 placeholder 등 UX 디테일이 추가됨
- 기존 테이블 기반 마크업은 삭제하지 않고 HTML 주석으로 통째로 남겨둠 — 리디자인 롤백에 대비한 것으로 보임

---

## 4. AWS EC2 기반 CI/CD 배포 서버 구축

Jenkins CI/CD 자체는 Day16(우분투 로컬 서버)에서 구축했고 오늘은 이를 실습하기 위해 AWS EC2에 별도 서버를 새로 띄우고 정리까지 진행함.

### 4-1. EC2 인스턴스 생성

- 이름: `cicdServer`, AMI: 우분투 26.04, 인스턴스 유형: small
- 키페어 신규 생성: 이름 `cicdKey`, 유형 ED25519, PuTTY용 형식으로 다운로드
- HTTP 트래픽 허용 체크
- 스토리지: 15GiB, 파일 시스템 없음으로 구성 후 인스턴스 시작
- 시작 후 인스턴스 세부 정보에서 퍼블릭 IP 확인(이하 `<서버IP>`로 표기)

### 4-2. 보안 그룹 인바운드 규칙 추가

- 인스턴스에 연결된 보안 그룹(launch-wizard)에서 인바운드 규칙 편집 → 규칙 추가
  - Oracle-RDS, Anywhere-IPv4
  - 사용자 지정 TCP 8080, Anywhere-IPv4
  - 사용자 지정 TCP 9090, Anywhere-IPv4 (Jenkins)
  - 사용자 지정 TCP 9092, Anywhere-IPv4 (Kafka)
- 이후 규칙 저장

### 4-3. PuTTY로 접속

- PuTTY 접속 정보에 `<서버IP>` 입력 후 세션 이름 저장
- SSH → Auth → Credentials에서 4-1에서 받은 키페어(.ppk) 등록
- 로그인 계정: `ubuntu`

### 4-4. 자바 21 설치 + 환경변수 등록

```
sudo apt update && sudo apt upgrade
sudo apt install openjdk-21-jdk -y
```

- 설치 경로 `/usr/lib/jvm/java-21-openjdk-amd64`를 확인해두고 `~/.bashrc`에 아래 내용 추가 후 `source ~/.bashrc`로 즉시 반영

```
export JAVA_HOME=/usr/lib/jvm/java-21-openjdk-amd64
export PATH=$PATH:$JAVA_HOME/bin
```

### 4-5. SSH 키 인증 등록

```
cd .ssh
cat authorized_keys
ssh-keygen -t ed25519 -C "<이메일>"
cat id_ed25519.pub >> authorized_keys
```

- `ssh-keygen`으로 새 키 쌍(공개키 `id_ed25519.pub` / 개인키 `id_ed25519`)을 생성하고 공개키를 `authorized_keys`에 추가하는 방식으로 키 기반 인증을 등록
- 개인키(`id_ed25519`)는 `-----BEGIN OPENSSH PRIVATE KEY-----`로 시작하는 텍스트 형태로 출력되는데, 이 값은 서버 접근 권한 그 자체이므로 노트나 채팅 등 어디에도 원문을 남기지 말고 로컬에만 안전하게 보관해야 함
- 원리: 개인키(`id_ed25519`, 로그인 시 제시하는 값)와 공개키(`id_ed25519.pub`, 서버의 `authorized_keys`에 등록해둔 값)가 한 쌍으로 일치해야 인증이 통과됨

### 4-6. Docker + Docker Compose 설치

```
sudo apt-get install ca-certificates curl gnupg lsb-release -y

curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" \
  | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io -y
sudo systemctl enable docker

sudo curl -L "https://github.com/docker/compose/releases/download/1.28.2/docker-compose-$(uname -s)-$(uname -m)" \
  -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
docker-compose --version

sudo usermod -aG docker ubuntu
newgrp docker
```

- Docker 공식 GPG 키를 시스템 키링에 등록하고 그 키로 서명된 저장소를 `apt` 소스 목록에 추가하는 표준 설치 순서
- `docker-compose` 바이너리는 다운로드만으로는 실행 권한이 없어 `chmod +x`를 반드시 별도로 줘야 함(빠뜨리면 `--version` 확인 단계에서 실행 실패)
- `usermod -aG docker ubuntu`로 그룹에 추가한 뒤에도 현재 세션에는 바로 반영되지 않아 `newgrp docker`로 그룹 갱신을 적용(또는 재로그인 필요)

### 4-7. Kafka 컨테이너 실행

```
docker run -d --name kafka -p 9092:9092 apache/kafka
```

- 별도의 ZooKeeper 없이 단일 컨테이너로 뜨는 `apache/kafka` 이미지 사용, 9092 포트를 4-2에서 열어둔 보안 그룹 규칙과 매핑

### 4-8. 프로젝트 배포

```
git clone <리포지토리 주소>
cd <프로젝트>
sudo chmod +x gradlew

export DB_URL="jdbc:oracle:thin:@<DB서버IP>:1521:XE"
export DB_USERNAME="<DB계정>"
export DB_PASSWORD="<DB비밀번호>"

sudo ./gradlew clean build -x test
cd build/libs
java -jar AWSMiniProject-0.0.1-SNAPSHOT.jar
```

- DB 접속 정보는 전부 환경변수로 주입 — `application.yml`의 `${DB_URL}`/`${DB_USERNAME}`/`${DB_PASSWORD}`와 매칭되는 구조(Day08에서 정리한 환경변수 주입 패턴과 동일)
- 환경변수 이름을 정확히 맞추는 게 중요함 — 비밀번호용 변수를 `DB_USERNAME`으로 잘못 선언하면 사용자명 변수만 덮어써지고 `DB_PASSWORD`는 비어있는 채로 빌드/실행되어 DB 인증이 실패함
- `gradlew clean build -x test`의 `-x test`는 테스트 태스크를 건너뛰고 빌드하는 옵션 — CI 환경에서 빌드 속도를 줄일 때 흔히 사용
- 실행 후 `9090` 포트(Jenkins) 또는 애플리케이션 포트로 정상 기동 여부 확인

### 4-9. 리소스 정리

- 실습 종료 후 AWS 대시보드에서 키페어 삭제 → 인스턴스 종료 → 보안 그룹 삭제 순으로 정리
- 인스턴스를 계속 켜두면 과금이 발생하므로, 학습·테스트 목적의 인스턴스는 확인이 끝나면 바로 정리하는 습관이 중요함

---

## 5. 다시 만들 때 체크리스트

```text
[Kafka 알림 파이프라인]
① Kafka로 주고받을 객체(ChatMessage)에는 기본 생성자가 필요 — @NoArgsConstructor 없으면 JSON 역직렬화 실패
② spring.kafka.properties.spring.json.trusted.packages를 지정해야 다른 패키지의 VO도 역직렬화 가능
③ Producer는 발행만, Consumer가 실제 후속 동작(STOMP 전송 등)을 담당하도록 역할을 분리하면 확장이 쉬움
④ auto-offset-reset=earliest는 신규 Consumer Group이 처음 구독할 때 과거 메시지부터 읽게 하는 옵션

[실시간 랭킹 크롤링]
⑤ Jsoup은 CSS 선택자(.class)로 HTML 요소를 그대로 추출 — 파싱 대상 사이트의 마크업이 바뀌면 선택자도 같이 깨짐
⑥ @Async가 실제로 동작하려면 @EnableAsync가 반드시 함께 있어야 함(Day16부터 이어지는 미해결 포인트)

[댓글 UI 리디자인]
⑦ 답글 들여쓰기는 반복 공백 대신 :style="{marginLeft: 값}"처럼 계산된 인라인 스타일로도 표현 가능
⑧ 리디자인 시 기존 마크업을 지우지 않고 주석으로 남겨두면 문제 발생 시 빠르게 되돌릴 수 있음

[AWS EC2 CI/CD 서버]
⑨ 보안 그룹 인바운드에 애플리케이션이 쓰는 포트(8080/9090/9092 등)를 미리 다 열어둬야 함
⑩ docker-compose 등 다운로드한 바이너리는 반드시 chmod +x로 실행 권한을 따로 부여해야 함
⑪ usermod -aG docker 적용 후에는 newgrp 또는 재로그인을 해야 그룹 변경이 현재 세션에 반영됨
⑫ 개인키(id_ed25519, .ppk 등)는 어떤 문서·채팅에도 원문 그대로 남기지 않는다 — 유출 시 서버 접근 권한이 그대로 넘어감
⑬ DB 접속 환경변수(URL/USERNAME/PASSWORD) 이름을 오타 없이 정확히 맞춰야 함 — 이름이 겹치면 뒤 값이 앞 값을 덮어씀
⑭ 학습·테스트용 인스턴스는 확인 후 키페어/인스턴스/보안 그룹까지 함께 정리해 불필요한 과금을 막는다
```
