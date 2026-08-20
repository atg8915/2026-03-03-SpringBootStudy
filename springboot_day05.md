# 📘 Spring Boot Day 05 — 검색·쉐프별 레시피·상세보기 (Day04 대비 추가) + Git Actions 자동 배포

> 이 문서는 [Day 04](./springboot_day04.md)에서 다룬 Recipe/Chef 목록 코드를 기반으로, **오늘 새로 추가된 부분만** 비교해서 정리했습니다.

## 0. 핵심 빠른 참조 — Day04 대비 추가된 것

| 구분 | Day 04 (어제) | Day 05 (오늘, 추가/변경) |
|------|--------------|---------------------------|
| 검색 기능 | 없음 (목록만) | `findByTitleContains`/`findByChefContains`에 **`page` 매개변수 추가** → Pageable 검색으로 확장 |
| 검색 결과 페이징 | 없음 | **`getPageDataFind(mode,page,rowsize,fd)`** 신설 — `mode`로 제목검색(1)/쉐프검색(2) 구분, `countByTitleContains`/`countByChefContains`로 검색 결과 개수 계산 |
| 상세보기 | 없음 (Recipe만 있고 상세 엔티티 없음) | **`RecipeDetail` 엔티티 + `RecipeDetailRepository` 신설** — 조리순서(`foodmake`), 쉐프 프로필 등 상세 전용 테이블 분리 |
| 화면 렌더링 방식 | Thymeleaf 서버 렌더링만 사용 | **Thymeleaf + Vue 혼합 패턴 등장** — 뼈대는 Thymeleaf가 그리고, 목록은 Vue+axios가 비동기로 채움 |
| REST 컨트롤러 | 없음 (Recipe는 서버 렌더링만) | **`RecipeRestController` 신설** (`restcontroller` 패키지) — `/recipe/find_vue`, `/recipe/recipe_chef_vue` 두 개의 검색 API |
| 배포 방식 | Docker Compose 수동 `up -d` | **Git Actions + self-hosted runner**로 `main` 브랜치 push 시 자동 빌드·배포 |

---

## 1. Service 계층 — Day04 대비 변경/추가 (`RecipeServiceImpl`)

### 기존 메서드 시그니처 변경
```java
// Day04: 검색 메서드가 페이징 없이 List만 반환
findByTitleContains(String title)
findByChefContains(String chef)

// Day05: page 매개변수 추가 → Pageable 검색으로 변경
@Override
public List<Recipe> findByTitleContains(String title, int page) {
    final int ROWSIZE = 12;
    Pageable pg = PageRequest.of(page-1, ROWSIZE, Sort.by(Sort.Direction.ASC,"no"));
    /*
     *  SELECT * FROM recipe
     *  WHERE title LIKE '%데이터%'
     *  ORDER BY no ASC OFFSET ? ROWS FETCH NEXT ? ROWS ONLY
     */
    Page<Recipe> pList = rDao.findByTitleContains(title, pg);
    return (pList != null && pList.hasContent()) ? pList.getContent() : new ArrayList<>();
}
// findByChefContains(chef, page) 도 동일 패턴
```

### 신규: 검색 결과 전용 페이지 계산
```java
@Override
public int[] getPageDataFind(int mode, int page, int rowsize, String fd) {
    int count = (mode == 1) ? rDao.countByTitleContains(fd) : rDao.countByChefContains(fd);
    int totalpage = (int)(Math.ceil(count/12.0));
    int startPage = ((page-1)/10*10)+1;
    int endPage   = ((page-1)/10*10)+10;
    if (endPage > totalpage) endPage = totalpage;
    return new int[]{page, totalpage, startPage, endPage};
}
```
- Day04의 `getPageData(page,rowsize)`는 **전체 목록**을 세던 계산임. 이건 검색 조건에 걸리는 건수(count)를 기준으로 별도 계산함
- 메서드 하나를 재사용하면서 `mode` 값으로 제목검색(1)/쉐프검색(2)을 분기해 카운트 쿼리만 다르게 호출함

### 신규: 상세보기 + 전체 건수
```java
@Override
public int recipeCount() { return rDao.recipeCount(); }   // 홈 화면에 전체 레시피 수 표시용

@Override
public RecipeDetail findByNo(int no) { return rdDao.findByNo(no); }  // RecipeDetailRepository 사용
```

---

## 2. Entity/Repository 신규 — `RecipeDetail`

```java
/*
 *  NO / POSTER / TITLE / CHEF / CHEF_POSTER / CHEF_PROFILE
 *  INFO1~3 (조리시간/난이도/인분 등 추정) / CONTENT(CLOB) / FOODMAKE(CLOB, 조리순서)
 */
@Entity
@Table(name="recipedetail")   // Recipe(목록용)와 별도 테이블
@Data
public class RecipeDetail {
	@Id
	private int no;
	private String poster,title,chef,chef_poster,chef_profile,info1,info2,info3,content,foodmake;
}

public interface RecipeDetailRepository extends JpaRepository<RecipeDetail, Integer>{
	public RecipeDetail findByNo(int no);
}
```
- 목록용 `Recipe`에는 제목·포스터·쉐프·조회수만 두고 상세용 `RecipeDetail`에 조리순서와 쉐프 소개까지 담아 **테이블/엔티티 자체를 분리**함. 목록 조회 성능을 위해 상세 컬럼(CLOB 등)을 목록 쿼리에서 제외함

---

## 3. RecipeRestController 신규 (`restcontroller` 패키지)

```java
@RestController
@RequiredArgsConstructor
public class RecipeRestController {
	private final RecipeService rService;

	// 제목 검색
	@RequestMapping("/recipe/find_vue")   // 반드시 비동기(axios)로만 호출
	public ResponseEntity<Map> recipe_find(@RequestParam("page") int page, @RequestParam("fd") String fd) {
		List<Recipe> list = rService.findByTitleContains(fd, page);
		int[] pages = rService.getPageDataFind(1, page, 12, fd);   // mode=1 (제목)
		return ResponseEntity.ok(Map.of("list", list, "pages", pages));
	}

	// 쉐프별 레시피 검색
	@RequestMapping("/recipe/recipe_chef_vue")
	public ResponseEntity<Map> recipe_chef(@RequestParam("page") int page, @RequestParam("chef") String chef) {
		List<Recipe> list = rService.findByChefContains(chef, page);
		int[] pages = rService.getPageDataFind(2, page, 12, chef);  // mode=2 (쉐프)
		return ResponseEntity.ok(Map.of("list", list, "pages", pages));
	}
}
```
- 예외 발생 시 공통적으로 `ResponseEntity.status(INTERNAL_SERVER_ERROR).build()` 반환. Day02의 GoodsRestController와 동일한 예외 처리 패턴을 그대로 재사용함

---

## 4. RecipeController 신규 매핑 4종

```java
// ① 홈 화면에 전체 건수 추가 (Day04 대비 count 하나 늘어남)
@GetMapping("/main/main")
public String main_main(...) {
    ...
    int count = rService.recipeCount();
    model.addAttribute("count", count);
    ...
}

// ② 검색 화면 (뼈대만 서버가 그리고, 실제 목록은 Vue가 채움)
@GetMapping("/recipe/find")
public String recipe_find(Model model) {
    model.addAttribute("main_html", "recipe/find");
    return "main/main";
}

// ③ 쉐프별 레시피 화면 (쉐프 이름을 Thymeleaf로 화면에 심어주고 Vue가 그 값으로 API 호출)
@GetMapping("/recipe/chef_recipe")
public String chef_recipe(@RequestParam("chef") String chef, Model model) {
    model.addAttribute("chef", chef);
    model.addAttribute("main_html", "recipe/chef_recipe");
    return "main/main";
}

// ④ 상세보기 (조리순서 파싱 로직 포함)
@GetMapping("/recipe/detail")
public String recipe_detail(@RequestParam("no") int no, Model model) {
    RecipeDetail vo = rService.findByNo(no);
    model.addAttribute("vo", vo);

    List<String> mList = new ArrayList<>();  // 조리순서 설명
    List<String> iList = new ArrayList<>();  // 각 단계 이미지
    String[] makes = vo.getFoodmake().split("\n");   // 줄바꿈으로 단계 분리
    for (String s : makes) {
        StringTokenizer st = new StringTokenizer(s, "^");  // '^' 구분자로 [설명]^[이미지] 분리
        mList.add(st.nextToken());
        iList.add(st.nextToken());
    }
    model.addAttribute("mList", mList);
    model.addAttribute("iList", iList);
    model.addAttribute("main_html", "recipe/detail");
    return "main/main";
}
```
- 핵심은 파싱 로직임. `foodmake` 컬럼 하나(CLOB)에 `설명^이미지경로` 형식으로 줄마다 저장돼 있어서 `\n`으로 1차 분리 → `^`로 2차 분리해서 두 개의 List로 나눔
- 화면에서는 인덱스 동기화 패턴을 씀. `th:each="m,stat:${mList}"`로 돌면서 `iList[stat.index]`로 같은 순서의 이미지를 매칭함

---

## 5. 화면 — Thymeleaf + Vue 혼합 패턴 (신규 기법)

### chef_recipe.html — Thymeleaf 값 → Vue data로 주입
```html
<script type="text/javascript">
const chef_name='[[${chef}]]'   // ← Thymeleaf 인라인으로 서버 값을 JS 변수에 심음
</script>
...
<script>
let findApp=Vue.createApp({
  data(){ return { find_list:[], ..., chef:chef_name } },  // Vue data 초기값으로 사용
  mounted(){ this.dataRecv() },
  methods:{
    async dataRecv(){
      await axios.get('http://localhost:8080/recipe/recipe_chef_vue',
        { params:{ page:this.curpage, chef:this.chef } })
        .then(response=>{
          this.find_list=response.data.list
          this.curpage=response.data.pages[0]
          this.totalpage=response.data.pages[1]
          this.startPage=response.data.pages[2]
          this.endPage=response.data.pages[3]
        })
    },
    change(no){ location.href="/recipe/detail?no="+no }  // 상세 이동은 서버 렌더링 페이지로
  }
}).mount(".container")
</script>
```
> 이번에 배운 핵심 패턴임. Controller가 `model.addAttribute("chef", chef)`로 넘긴 값을 화면의 `<script>` 안에서 `[[${chef}]]`로 꺼내 **JS 변수로 굳혀서** Vue의 `data()` 초기값에 연결함. Thymeleaf의 서버 렌더링과 Vue의 클라이언트 렌더링을 한 페이지 안에서 이어붙임

### find.html — 검색어 입력 + Vue 반응형 상태
```html
<input type="text" v-model="fd" ref="fdRef" @keydown.enter="find()">
<button @click="find()">검색</button>
```
```javascript
find(){
  if(!this.fd){ this.$refs.fdRef.focus(); return }  // 빈 입력이면 포커스만 주고 종료
  this.curpage=1
  this.dataRecv()
}
```
- `v-model`로 입력값을 실시간 바인딩함. DOM 포커스 제어는 `$refs`가 맡음. Vue Composition 없이 Options API 그대로 사용함

### detail.html — 조리순서 인덱스 동기화 출력
```html
<table class="table" th:each="m,stat:${mList}">
  <tr>
    <td width="80%">[[${m}]]</td>
    <td width="20%"><img th:src="${iList[stat.index]}" style="width:170px;height:120px"></td>
  </tr>
</table>
```
- `th:each="m,stat:${mList}"`의 `stat.index`로 다른 리스트(iList)의 같은 순번 항목을 함께 출력함. Day01에서 정리했던 `status.index/count/first/last` 문법을 여기서 실전 활용함

---

## 6. Git Actions + Self-hosted Runner 배포 (완전히 새로운 내용)

### Runner 설치 (Ubuntu 서버에서)
```bash
mkdir actions-runner && cd actions-runner

curl -o actions-runner-linux-x64-2.336.0.tar.gz -L \
  https://github.com/actions/runner/releases/download/v2.336.0/actions-runner-linux-x64-2.336.0.tar.gz

# (선택) 해시 검증
echo "04cf0be1aff4c3ec3554466c39124ca250e3effd8873bb7e8d68535aa9505d5d  actions-runner-linux-x64-2.336.0.tar.gz" | shasum -a 256 -c

tar xzf ./actions-runner-linux-x64-2.336.0.tar.gz

# 러너 등록 (GitHub 저장소 URL + 토큰은 저장소 Settings > Actions > Runners에서 발급)
./config.sh --url https://github.com/atg8915/SpringThymeLeafDockerProject --token <발급받은 토큰>

# 실행
./run.sh
```
- 워크플로우 파일에서는 `runs-on: self-hosted`로 지정하면 이 러너가 실행됨

### GitHub에서 워크플로우 생성
- 저장소 Actions 탭 → **"set a workflow yourself"** 클릭 → `.github/workflows/deploy.yml` 형태로 생성

### deploy.yml — main push 시 자동 빌드+배포
```yaml
name: Deploy Git Action
on:
  push:
    branches:
      - main            # main 브랜치에 push할 때마다 배포 시작

jobs:
  deploy:
    runs-on: self-hosted   # 등록해둔 Ubuntu 서버에서 실행
    steps:
      - uses: actions/checkout@v4          # 소스코드 다운로드 (clone과 동일)
      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: '21'
      - run: chmod +x gradlew              # gradlew 실행 권한 부여
      - run: ./gradlew clean build -x test # 빌드 (테스트 제외)
      - run: |                              # 기존 8080 포트 점유 프로세스 종료
          PID=$(lsof -t -i:8080 || true)
          if [ -n "$PID" ]; then
             kill -15 "$PID" || true
          fi
        env:
          RUNNER_TRACKING_ID: ""
      - run: |                              # 새 jar 실행 (nohup + disown = 세션 종료 후에도 유지)
          nohup java -jar build/libs/SpringThymeLeafDockerProject-0.0.1-SNAPSHOT.jar > app.log 2>&1 &
          disown
        env:
          RUNNER_TRACKING_ID: ""
```

### 배포 흐름 요약
```text
로컬에서 git push (main 브랜치)
   |
GitHub Actions 트리거
   |
self-hosted 러너(Ubuntu 서버)가 작업 수신
   |
checkout → JDK21 세팅 → gradlew 빌드 → 기존 프로세스 종료 → 새 jar 백그라운드 실행
   |
서버 무중단(수동 개입 없이) 재배포 완료
```

### ⚠️ 오늘의 교훈
> **Git Actions 연동 시 서버가 자꾸 꺼지는 문제 발생 → 원인은 러너(RUNNER) 이름과 워크플로우 안의 jar 파일명이 정확히 일치하지 않았기 때문.**
> `nohup java -jar build/libs/<프로젝트명>-<버전>-SNAPSHOT.jar` 의 파일명이 실제 빌드 산출물 이름과 한 글자라도 다르면 실행이 실패함. 러너 등록 시 지정한 이름이 워크플로우와 어긋나도 배포 자체가 안 됨.
> → **RUNNER 이름과 jar 파일명(빌드 산출물 경로)을 항상 정확히 맞춰야 한다.**

---

## 7. 다시 만들 때 체크리스트 (Day05 추가분 기준)

```text
[검색 기능]
① Repository: findByXxxContains(값, Pageable) 오버로드 + countByXxxContains(값) 추가
② Service: getPageDataFind(mode, page, rowsize, fd)로 검색 전용 페이지 계산 분리
③ RestController(restcontroller 패키지): /recipe/find_vue, /recipe/recipe_chef_vue 두 API로 검색 결과 JSON 반환

[상세보기]
④ 상세 전용 Entity/Repository를 목록용과 분리 (RecipeDetail ↔ Recipe)
⑤ CLOB 컬럼에 구분자(\n, ^)로 여러 값을 저장한 경우 Controller에서 split/StringTokenizer로 파싱 후 별도 List 2개로 분리
⑥ 화면에서는 th:each의 stat.index로 두 List를 같은 순번끼리 매칭 출력

[Thymeleaf+Vue 혼합]
⑦ Controller가 model에 담은 값을 <script>의 [[${값}]]으로 꺼내 JS 변수로 고정 → Vue data() 초기값에 연결
⑧ 목록/페이징은 그대로 Vue+axios 패턴(Day02~04) 재사용, 상세 이동은 location.href로 서버 렌더링 페이지 이동

[Git Actions 배포]
⑨ Ubuntu에 self-hosted runner 설치 및 등록 (config.sh → run.sh)
⑩ .github/workflows/deploy.yml 작성: on push(main) → checkout → JDK셋업 → gradlew build → 기존 프로세스 kill → nohup으로 새 jar 실행
⑪ jar 파일명, 러너 이름을 워크플로우 스크립트와 정확히 일치시키기 (틀리면 서버 다운)
```

---

