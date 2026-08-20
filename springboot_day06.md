# Spring Boot Day 06 — 파일 업로드 3종 + DataBoard(첨부파일 게시판) + Git Actions Docker 배포 전환

## 0. 핵심 빠른 참조

| 구분 | 내용 |
|------|------|
| 업로드 방식 3종 | ① 순수 HTML form(단일/다중) ② Vue+axios+FormData(다중) ③ jQuery로 입력칸 동적 추가/삭제(다중) |
| 신규 게시판 | `DataBoard` — 첨부파일까지 저장하는 게시판, **MyBatis(Mapper) 기반**으로 처리 (지금까지의 Recipe/Food/Goods는 JPA Repository였음 — 방식 전환) |
| 파일 저장 위치 지정 | `UploadRestController`: `application.yml`의 `file.upload_dir` 값을 `@Value`로 주입 / `DataBoardController`: `request.getServletContext().getRealPath("/upload")`로 웹앱 내부 경로 사용 (같은 문제를 두 가지 방식으로 처리) |
| 중복 파일명 처리 | 동일 파일명 존재 시 `이름(순번).확장자` 형태로 변경 (업로드 컨트롤러마다 각자 구현) |
| **Git Actions 배포 방식 전환** | **Day05**: `gradlew build` → jar를 `nohup java -jar`로 직접 실행 → **Day06**: `gradlew build` → **Docker 이미지 빌드** → 기존 컨테이너 stop/rm → 새 컨테이너로 `docker run` |

---

## 1. 파일 업로드 화면 3종 비교

| 방식 | 파일 | 핵심 특징 |
|------|------|-----------|
| ① 순수 HTML | `upload.html` | `<form enctype="multipart/form-data">` 두 개(단일/다중), `<input type=file multiple>`로 다중 선택, 서버가 전체 페이지를 새로 렌더링(전통적 submit) |
| ② Vue+axios | `upload2.html` | `@change="handlerFile"`로 선택 파일을 `this.files`에 저장 → `FormData`에 담아 `axios.post`로 비동기 전송, 성공 시 `alert` |
| ③ jQuery 동적 행 추가 | `upload3.html` | `Add`/`Remove` 버튼으로 `<input type=file>` 행을 테이블에 동적 추가/삭제 후 한 번에 `<form>` submit |

### upload.html의 단일·다중 폼 2개
```html
<form method="post" action="/upload_ok" enctype="multipart/form-data">
  <input type="file" name="file">
  <button type="submit">업로드</button>
</form>
<form method="post" action="/multi-upload" enctype="multipart/form-data">
  <input type="file" name="files" multiple>
  <button type="submit">업로드</button>
</form>
```

### upload2.html의 Vue + FormData
```javascript
data(){ return { files:[] } },
methods:{
  handlerFile(e){ this.files = e.target.files },   // input change 이벤트로 파일 목록 저장
  submit(){
    const formData = new FormData()
    for (let i of this.files) formData.append('files', i)   // 여러 개를 같은 key('files')로 append
    axios.post('multi-upload', formData, {
      headers: { 'Content-Type': 'multipart/form-data' }
    }).then(() => alert("등 록 완 료 ! ! ! !"))
  }
}
```
- `FormData`에 반복 `append`하면 서버는 `List<MultipartFile> files`로 한 번에 받을 수 있음

### upload3.html에서 jQuery로 업로드 행 동적 관리
```javascript
let fileIndex = 0
$('#addBtn').on('click', function(){
  $('#upload-table tbody').append(
    '<tr id="m'+fileIndex+'"><td>Files:'+(fileIndex+1)+'</td>'
    +'<td><input type=file name="files"></td></tr>'
  )
  fileIndex++
})
$('#removeBtn').on('click', function(){
  if (fileIndex > -1) { $('#m'+(fileIndex-1)).remove(); fileIndex-- }
})
```
- `<input type=file name="files">`를 동적으로 여러 개 추가한 뒤 `<form>` submit 한 번으로 전체 전송. Vue 없이 jQuery만으로 다중 업로드 UX를 구현함

---

## 2. UploadRestController의 단일/다중 업로드 API

```java
@RestController
public class UploadRestController {
	@Value("${file.upload_dir}")   // application.yml의 file.upload_dir 값 주입
	private String uploadDir;
	private static int count = 1;  // 다중 업로드에서 같은 파일명 중복될 때 사용

	@PostMapping("/upload_ok")
	public String upload_ok(@RequestParam(value="file",required=false) MultipartFile file) throws Exception {
		File f = new File(uploadDir);
		if (!f.exists()) f.mkdir();          // 업로드 폴더 없으면 생성
		if (file.isEmpty()) return "파일이 없습니다!!";

		String oname = file.getOriginalFilename();
		File files = new File(uploadDir+"/"+oname);
		String newName = oname;
		if (files.exists()) {                // 동명 파일 존재 시 이름(count).확장자로 변경
			String name = oname.substring(0, oname.lastIndexOf('.'));
			String ext  = oname.substring(oname.lastIndexOf("."));
			newName = name+"("+count+")"+ext;
			count++;
		}
		Path savePath = Paths.get(uploadDir, newName);
		Files.copy(file.getInputStream(), savePath);   // 실제 파일 저장
		return "업로드 성공 :"+oname+",변경:"+newName;
	}

	@PostMapping("/multi-upload")
	public String multi_upload(@RequestParam(value="files",required=false) List<MultipartFile> files) throws Exception {
		for (MultipartFile file : files) {
			if (file.isEmpty()) return "파일이 존재하지 않는다";
			String oname = file.getOriginalFilename();
			File f = new File(uploadDir+"/"+oname);
			if (f.exists()) {                // while로 중복 안 날 때까지 순번 증가
				String name = oname.substring(0, oname.lastIndexOf('.'));
				String ext  = oname.substring(oname.lastIndexOf("."));
				int cnt = 1;
				while (f.exists()) {
					f = new File(uploadDir+"/"+name+"("+cnt+")"+ext);
					cnt++;
				}
			}
			Files.copy(file.getInputStream(), Paths.get(uploadDir, f.getName()));
		}
		return "다중 업로드 완료";
	}
}
```
- 단일 업로드는 클래스 필드 `count`로, 다중 업로드는 요청마다 만드는 지역변수 `cnt`로 중복을 처리함. 같은 문제를 컨트롤러 안에서 다르게 구현했으니 주의
- `count`는 모든 요청이 공유하므로 여러 사용자가 동시에 올리면 값이 꼬일 수 있음

---

## 3. 첨부파일 게시판 DataBoard (MyBatis 기반으로 전환)

### DataBoardVO 테이블 명세와 향후 커리큘럼 메모
```java
/*
 *  NO/NAME/SUBJECT/CONTENT/PWD/REGDATE/HIT/FILENAME/FILESIZE/FILECOUNT
 *
 *  향후 학습 예정 메모:
 *  1. Spring-Boot => Thymeleaf
 *  2. JPA + MyBatis => 동적 쿼리(JOIN/JPQL/QueryDSL)
 *  3. Spring Security + JWT
 *  4. 알림 => WebSocket + Stomp + 카프카
 *  5. JavaMail
 *  6. Front => Pinia(Vue)
 *  7. Spring AI
 *  => CI/CD(AWS) 무중단 배포(Blue/Green), Nginx => Jenkins
 *  ---------------
 *  SpringAI + React + TanStack-Query + TypeScript + NodeJS => NextJS
 */
@Data
public class DataBoardVO {
	private int no, hit, filecount;
	private String name, subject, content, pwd, filename, filesize, dbday;
	private Date regdate;
	private List<MultipartFile> files;   // 폼에서 받은 첨부파일 목록 (커맨드 객체에 파일 필드 포함)
}
```
- 이전 Entity들과 달리 `@Entity`가 아닌 순수 VO임. DataBoard는 JPA가 아니라 MyBatis Mapper로 SQL을 직접 다루고 `DataBoardServiceImpl`이 `DataBoardMapper`를 주입받음

### Service의 Mapper 위임 구조
```java
public interface DataBoardService {
	List<DataBoardVO> databoardListData(int start);
	int databoardTotalPage();
	void databoardInsert(DataBoardVO vo);
}
@Service @RequiredArgsConstructor
public class DataBoardServiceImpl implements DataBoardService {
	private final DataBoardMapper mapper;   // Repository 대신 Mapper
	// 각 메서드는 mapper.xxx() 호출만 그대로 위임
}
```

### DataBoardController — 목록/등록/파일 저장
```java
@GetMapping("/databoard/list")
public String databoard_list(@RequestParam(value="page",required=false) String page, Model model) {
    if (page == null) page = "1";
    int curpage = Integer.parseInt(page);
    int start = (curpage*10)-10;
    model.addAttribute("list", dService.databoardListData(start));
    model.addAttribute("curpage", curpage);
    model.addAttribute("totalpage", dService.databoardTotalPage());
    model.addAttribute("main_html", "databoard/list");
    return "main/main";
}

@PostMapping("/databoard/insert_ok")
public String databoard_insert_ok(@ModelAttribute("vo") DataBoardVO vo, HttpServletRequest request) throws Exception {
    // uploadDir을 웹앱 실제 경로에서 얻어옴 (yml 설정과는 다른 방식)
    String uploadDir = request.getServletContext().getRealPath("/upload");
    File dir = new File(uploadDir);
    if (!dir.exists()) dir.mkdir();

    List<MultipartFile> files = vo.getFiles();
    String filename = "", filesize = "";
    boolean bCheck = false;
    for (MultipartFile file : files) {
        if (file.isEmpty()) { bCheck = false; continue; }
        // 중복 파일명 처리 (이름(count).확장자)
        // ... Files.copy로 저장 ...
        filename += f.getName()+",";
        filesize += f.length()+",";
        bCheck = true;
    }
    if (bCheck) {
        filename = filename.substring(0, filename.lastIndexOf(","));  // 마지막 콤마 제거
        filesize = filesize.substring(0, filesize.lastIndexOf(","));
        vo.setFilename(filename); vo.setFilesize(filesize); vo.setFilecount(files.size());
    } else {
        vo.setFilename(""); vo.setFilesize(""); vo.setFilecount(0);
    }
    dService.databoardInsert(vo);
    return "redirect:/databoard/list";
}
```
- 여러 파일명을 콤마(,)로 이어붙여 한 컬럼 `FILENAME`에 저장함. 파일마다 행을 나누지 않고 문자열 하나로 관리하는 셈. Day05에서 `foodmake`를 구분자로 저장했던 방식과 같음
- `getRealPath("/upload")`는 Servlet 컨테이너 기준 실제 경로를 구함. yml 설정값을 읽는 `UploadRestController`의 `@Value("${file.upload_dir}")`와는 접근이 다름
- 업로드 폴더 위치를 정하는 같은 문제를 컨트롤러마다 다르게 처리함

---

## 4. application.yml에 업로드 설정 추가

```yaml
server:
  servlet:
    context-path: /
spring:
  servlet:
    multipart:
      enabled: true
      max-file-size: 100MB       # 파일 1개 최대 크기
      max-request-size: 100MB    # 요청 전체(여러 파일 합산) 최대 크기
  ...
file:
  upload_dir: C:/upload          # UploadRestController가 @Value로 읽는 경로
```
- `spring.servlet.multipart` 설정이 없으면 기본 업로드 용량 제한 1MB에 걸려 큰 파일 업로드가 실패함. 이번에 새로 추가함

---

## 5. Docker 사용 권한 설정 (Ubuntu)

```bash
sudo usermod -aG docker $USER
```
- 현재 사용자를 `docker` 그룹에 추가함. 이후로는 `sudo` 없이 `docker` 명령 실행 가능
- Git Actions self-hosted runner가 `sudo` 없이 매 스텝을 실행해야 하므로 이 설정이 없으면 워크플로우 내 `docker build/run` 명령이 권한 오류로 실패함

---

## 6. Git Actions 배포 방식 전환 — jar 직접 실행 → Docker 컨테이너 실행

| 구분 | Day 05 (jar 직접 실행) | Day 06 (Docker 컨테이너 실행) |
|------|--------------------------|-------------------------------|
| 빌드 이후 단계 | jar 실행 전 기존 8080 포트 프로세스를 `lsof`+`kill`로 종료 | Docker 이미지를 새로 빌드(`docker build`) |
| 실행 방식 | `nohup java -jar ...jar & disown` | 기존 컨테이너 `stop`+`rm` 후 `docker run --name mini-app -d -p 8080:8080` |
| 프로세스 격리 | 서버 OS에 자바 프로세스로 직접 상주 | 컨테이너 단위로 격리되어 실행, 이미지로 버전 관리 가능 |

### deploy.yml (Docker 버전)
```yaml
name: Deploy Git Action
on:
  push:
    branches:
      - main
jobs:
  deploy:
    runs-on: self-hosted
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: '21'
      - run: chmod +x gradlew
      - run: ./gradlew clean build -x test

      # Docker 이미지 생성
      - run: docker build -t mini-app .

      # 기존 컨테이너 중지/삭제 (없어도 에러 없이 넘어가도록 || true)
      - run: |
          docker stop mini-app || true
          docker rm mini-app || true

      # 새 컨테이너 실행
      - run: |
          docker run --name mini-app -d -p 8080:8080 mini-app
```
> Day05에서 `lsof`로 포트를 죽이고 `nohup java -jar`로 재실행하던 단계는 주석 처리되어 남아있음. 배포는 Docker 컨테이너 방식으로 완전히 대체됨

### 배포 흐름 요약
```text
git push (main)
   |
self-hosted 러너 작업 시작
   |
checkout → JDK21 → gradlew build
   |
docker build -t mini-app .        (새 이미지 생성)
   |
docker stop/rm mini-app (기존 컨테이너 정리, 실패해도 무시)
   |
docker run --name mini-app -d -p 8080:8080 mini-app  (새 컨테이너 기동)
```

---

## 7. 다시 만들 때 체크리스트

```text
[파일 업로드]
① application.yml에 spring.servlet.multipart(enabled/max-file-size/max-request-size) 먼저 설정
② 저장 경로는 @Value("${file.upload_dir}") 또는 request.getServletContext().getRealPath()로 결정 (프로젝트 전체에서 하나의 방식으로 통일 권장)
③ 중복 파일명은 "이름(순번).확장자"로 변경, while(f.exists())로 완전히 겹치지 않을 때까지 순번 증가
④ 화면은 목적에 맞게 선택: 단순 폼(HTML) / 비동기+미리 목록 표시(Vue+FormData) / 입력칸 동적 추가(jQuery)

[DataBoard(MyBatis)]
⑤ VO에 List<MultipartFile> files 필드 포함 → @ModelAttribute로 폼 전체 커맨드 객체 바인딩
⑥ 여러 파일명/용량은 콤마(,)로 이어붙여 한 컬럼에 저장, 마지막 콤마는 substring으로 제거
⑦ Service는 Mapper(MyBatis)에 위임만 — Repository(JPA)와 병행 사용 가능함을 기억

[배포 - Docker 전환]
⑧ Ubuntu에서 sudo usermod -aG docker $USER로 sudo 없이 docker 명령 가능하게 설정
⑨ deploy.yml: gradlew build → docker build → 기존 컨테이너 stop/rm(|| true로 실패 무시) → docker run -d -p
⑩ Dockerfile은 이전 Day(FROM eclipse-temurin-21-jdk, COPY build/libs/*.jar, ENTRYPOINT) 그대로 재사용
```

---

