# 📘 Spring Boot Day 04 — Recipe/Chef 목록 + 페이지 배열 방식 + Docker Compose 실행

## 0. 핵심 빠른 참조

| 구분 | Day 03까지 | Day 04 (변경점) |
|------|-----------|------------------|
| 페이지 정보 전달 | `curpage/totalpage/startPage/endPage` 각각 개별 model 속성 | **`pages[]` 배열 하나**로 통합 (`pages[0]`=curpage, `pages[1]`=totalpage, `pages[2]`=startPage, `pages[3]`=endPage) |
| 페이지 크기 | Service 내부에 고정값(12) | **`getPageData(page, rowsize)`처럼 rowsize를 매개변수로 전달** → 목록(12건)/쉐프(20건) 다른 크기 재사용 |
| 화면 전환 | main_html로 `food/list`, `goods/list` 등 | 동일 패턴을 **레시피 앱(recipe/chef)에 재사용** — main.html/header.html 구조 그대로 |
| Docker | Dockerfile로 직접 build/tag/push/run | **docker-compose.yml**로 이미지 pull+실행을 한 번에 (`up -d` / `down`) |

| 계층 | 이번 자료 핵심 |
|------|----------------|
| Entity | `Recipe`(레시피 목록), `Chef`(쉐프 프로필, PK가 문자열 `chef`) |
| Repository | `findByTitleContains`, `findByChefContains` 등 메서드 규칙 검색, `findAll(Pageable)`로 페이징 |
| Service | `recipeListData`/`chefListData` — Pageable+Sort로 페이징, `getPageData(page,rowsize)`로 공용 페이지 계산 |
| Controller | `/main/main`(레시피 목록=홈), `/recipe/chef_list`(쉐프 목록) — 둘 다 `main/main` 뼈대로 forward |

---

## 1. 계층 구조 비유 (주석 그대로 정리)

```text
요청 ==== DispatcherServlet ===== @Controller
             |                         |
             ---------------------------
               | 연동 (필요한 데이터나 내장객체 => 매개변수)

1. Repository / Mapper => 데이터베이스만 연동   → "재료"
2. Service              => 조립(Repository가 받은 값 가공) → "주방"
3. Controller           => 조립된 데이터만 받아서 HTML 전송 → "서빙"
```

### Controller 매개변수 정리 (재복습, 이번 파일 주석 기준)
| 매개변수 | 용도 |
|----------|------|
| `@RequestParam` | 단일 값 (`getParameter()`에 대응) |
| `@ModelAttribute` | 커맨드 객체(VO 단위) 받기 |
| `@RequestBody` | JSON → VO 변환 (`@RestController`에서 사용, 자바스크립트 연동) |
| `Model` | 서버 → 화면 전송 객체 (request에 대응) |
| `RedirectAttributes` | `sendRedirect` 시 값 전송 |
| `HttpSession` / `HttpServletRequest` / `HttpServletResponse` | 내장 객체, 세션·쿠키 처리 |
| `Principal` | 보안(Security)에서 로그인 사용자 정보, session 대체 |
| `@RequestParam(value="page",required=false)` | null 허용 → 검색/페이지 파라미터에 사용 |

---

## 2. Entity — Recipe / Chef

```java
@Entity
@Data
public class Recipe {
	@Id
	private int no;
	private String title,poster,chef,link;
	private int hit;
}

@Entity
@Data
public class Chef {
	@Id
	private String chef;   // PK가 문자열(쉐프 이름)
	private String poster;
	private String mem_cont1,mem_cont3,mem_cont7,mem_cont2;  // 메뉴/소개 콘텐츠 슬롯 4개
}
```
- `Chef`는 다른 Entity들과 달리 **PK가 `int no`가 아니라 `String chef`** — 쉐프 이름 자체를 기본키로 사용
- Repository 주석에 `// JOIN Recipe = Chef => @Query` 메모 → 추후 Recipe와 Chef를 JOIN하는 쿼리 작성 예정

---

## 3. Repository — 검색 메서드 규칙 재정리

```java
public interface RecipeRepository extends JpaRepository<Recipe, Integer> {
    public List<Recipe> findByTitleContains(String title);
    public List<Recipe> findByChefContains(String chef);
}
```

| 메서드 패턴 | 생성되는 SQL |
|-------------|-------------|
| `findByName(String name)` | `WHERE name=?` (equals) |
| `findByTitleStartsWith(String title)` | `WHERE title LIKE 'title%'` |
| `findByTitleEndsWith(String title)` | `WHERE title LIKE '%title'` |
| `findByTitleContains(String title)` | `WHERE title LIKE '%title%'` |
| `findByOrderByTitleDesc()` | `ORDER BY title DESC` |
| `findAll(Pageable, Sort)` / `count()` / `save()` / `delete()` | 기본 제공 |

```java
public interface ChefRepository extends JpaRepository<Chef, String> {
	// findAll, count 기본 제공만 사용 (전용 검색 메서드는 아직 없음)
}
```

---

## 4. Service — Pageable 목록 조회 + 공용 페이지 계산

### 목록 조회 (Recipe / Chef 동일 패턴)
```java
// Recipe: 12건씩, no 오름차순
Pageable pg = PageRequest.of(page-1, 12, Sort.by(Sort.Direction.ASC,"no"));
Page<Recipe> pList = rDao.findAll(pg);
/*
 *  실제 SQL문장
 *  SELECT * FROM recipe ORDER BY no ASC OFFSET ? ROWS FETCH NEXT ? ROWS ONLY
 *  JPA => 중심이 객체 단위 (@Entity) → 객체===Column(메소드) = ORM (C#의 LINQ와 유사한 개념)
 */
List<Recipe> list = new ArrayList<>();
if (pList != null && pList.hasContent()) list = pList.getContent();

// Chef: 20건씩 (정렬 조건 없이)
Pageable pg = PageRequest.of(page-1, 20);
Page<Chef> pList = cDao.findAll(pg);
```

### getPageData(page, rowsize) — 페이지 크기를 매개변수로 뺀 공용 메서드
```java
int totalpage = (int)(Math.ceil(rDao.count()/(double)rowsize));
int startPage = ((page-1)/10*10)+1;
int endPage   = ((page-1)/10*10)+10;
if (endPage > totalpage) endPage = totalpage;
int[] pages = {page, totalpage, startPage, endPage};
return pages;
```
- Day02~03에서는 `12`(행 크기)가 메서드 내부에 고정돼 있었는데, 이번엔 **`rowsize`를 인자로 받아** 레시피(12건)와 쉐프(20건)에 같은 메서드를 재사용
- 반환값은 개별 변수가 아니라 **배열 하나(`pages[]`)** 로 통일 → Controller/화면에서 `pages[0]~pages[3]`로 접근

---

## 5. Controller — main_html 패턴 재사용

```java
@GetMapping("/main/main")
public String main_main(@RequestParam(value="page",required=false) String page, Model model) {
    if (page == null) page = "1";
    List<Recipe> list = rService.recipeListData(Integer.parseInt(page));
    int[] pages = rService.getPageData(Integer.parseInt(page), 12);
    model.addAttribute("pages", pages);
    model.addAttribute("list", list);
    model.addAttribute("main_html", "main/home");   // templates/main/home.html
    return "main/main";
}

@GetMapping("/recipe/chef_list")
public String recipe_chef(@RequestParam(value="page",required=false) String page, Model model) {
    if (page == null) page = "1";
    List<Chef> list = rService.chefListData(Integer.parseInt(page));
    int[] pages = rService.getPageData(Integer.parseInt(page), 20);
    model.addAttribute("pages", pages);
    model.addAttribute("list", list);
    model.addAttribute("main_html", "recipe/chef");
    return "main/main";
}
```
- Day03에서 정리한 **레이아웃 include 패턴**(`main.html` + `th:include="${main_html}"`)을 그대로 재사용 — 레시피 앱에서도 동일한 뼈대 구조 유지
- `/main/main`을 **홈(레시피 목록)** 진입점으로 사용

---

## 6. 화면 — pages[] 배열 기반 페이징 (Thymeleaf)

```html
<!-- home.html (레시피 목록) -->
<div class="col-sm-3" th:each="vo:${list}">
  <img th:src="${vo.poster}" th:title="${vo.title}">
  <p>[[${vo.chef}]]</p>
</div>

<!-- 페이징: curpage 대신 pages[0], totalpage 대신 pages[1] ... -->
<li th:if="${pages[2]>1}"><a th:href="@{/main/main(page=${pages[2]-1})}">&laquo;</a></li>
<li th:each="i:${#numbers.sequence(pages[2],pages[3])}"
    th:class="${i==pages[0]?'active':''}">
  <a th:href="@{/main/main(page=${i})}">[[${i}]]</a>
</li>
<li th:if="${pages[3]<pages[1]}"><a th:href="@{/main/main(page=${pages[3]+1})}">&raquo;</a></li>
```

```html
<!-- chef.html (쉐프 목록) -->
<table class="table" th:each="vo:${list}">
  <tr>
    <td rowspan="2"><img th:src="${vo.poster}" class="img-circle"></td>
    <td colspan="4" style="color:orange;">[[${vo.chef}]]</td>
  </tr>
  <tr>
    <!-- mem_cont1~mem_cont2 를 아이콘 4개와 나란히 출력 -->
    <td><img src="/icon/m1.png">[[${vo.mem_cont1}]]</td>
    <td><img src="/icon/m2.png">[[${vo.mem_cont3}]]</td>
    <td><img src="/icon/m3.png">[[${vo.mem_cont7}]]</td>
    <td><img src="/icon/m4.png">[[${vo.mem_cont2}]]</td>
  </tr>
</table>
<!-- 페이징 링크는 /recipe/chef_list(page=...) 로 동일 구조 재사용 -->
```

### header.html — 드롭다운 메뉴 추가
```html
<a class="dropdown-toggle" data-toggle="dropdown" href="#">레시피 <span class="caret"></span></a>
<ul class="dropdown-menu">
  <li><a href="#">레시피 검색</a></li>
  <li><a href="/recipe/chef_list">쉐프</a></li>
</ul>
<li><a href="#">실시간 채팅(Pinia)</a></li>  <!-- 다음 학습 예고: Pinia 실시간 채팅 -->
```
- `find.html`은 아직 빈 껍데기(레시피 검색 화면, 미구현 상태)

---

## 7. Docker Compose로 실행하기

### docker-compose.yml
```yaml
version: "3"
services:
  app:
    image: atg8915/recipe-app
    ports:
      - "8080:8080"
```

### 실행/종료 명령
```bash
# Spring 프로젝트 폴더에서
sudo nano docker-compose.yml     # 위 내용 작성

sudo docker compose up -d        # 백그라운드로 컨테이너 실행
sudo docker compose down         # 컨테이너 중지+제거
```

| 이전 방식 (Day02~03) | Compose 방식 (Day04) |
|----------------------|------------------------|
| `docker build` → `tag` → `push` → `pull` → `docker run -p 8080:8080 이미지` | `docker-compose.yml`에 이미지·포트 선언 → `docker compose up -d` 한 줄 |
| 컨테이너 이름/포트를 매번 `run` 명령에 옵션으로 지정 | yml 파일에 고정 선언 → 재실행 시 명령어 반복 불필요 |
| 종료: `stop` + `rm` 두 단계 | 종료: `docker compose down` 한 번 |

---

## 8. 다시 만들 때 체크리스트

```text
[Entity/Repository]
① Entity: PK 타입이 반드시 int일 필요 없음 (Chef처럼 String PK도 가능)
② Repository: 검색은 findByXxxContains 등 메서드 규칙, 목록은 findAll(Pageable)

[Service]
③ 목록 조회: Pageable(page-1, rowsize, Sort) → Page<T> → hasContent() 체크 후 getContent()
④ 페이지 계산: getPageData(page, rowsize)로 rowsize를 매개변수화 → 여러 목록(12건/20건 등)에 재사용
⑤ 반환은 개별 변수 대신 pages[]="{page,totalpage,startPage,endPage}" 배열 하나로 통일

[Controller]
⑥ main_html 방식 유지: model.addAttribute("main_html","경로") → return "main/main"
⑦ page 파라미터는 required=false로 null 허용, null이면 "1"로 기본값 처리

[화면]
⑧ 페이징 HTML은 pages[0]~pages[3] 인덱스로 접근 (curpage/totalpage 이름 대신)
⑨ 새 목록 화면 추가 시 기존 main.html/header.html 레이아웃 그대로 재사용

[배포]
⑩ docker-compose.yml에 image+ports 선언
⑪ 실행: docker compose up -d / 종료: docker compose down
```

