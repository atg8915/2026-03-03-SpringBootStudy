# Spring Boot Day 08

Recipe MyBatis 전환, CDN 방식 Pinia, 환경변수 기반 Docker 배포

> 주의: 이 문서에서 IP, DB 계정, 컨테이너 ID 등은 모두 `<...>` 형태로 마스킹 처리했습니다. 실제 값은 각자 환경에 맞게 채워 넣으세요.

## 0. 핵심 빠른 참조

| 구분 | 지금까지 (Day04~07) | Day 08 (오늘) |
|------|----------------------|----------------|
| Recipe 데이터 처리 | JPA(`RecipeRepository extends JpaRepository`) | MyBatis Mapper로 전환 (`RecipeMapper` + `recipe-mapper.xml`) |
| Recipe 모델 | `@Entity` (Recipe, RecipeDetail) | 순수 VO(`RecipeVO`, `RecipeDetailVO`), `@Entity` 어노테이션 제거, DB 매핑은 XML이 담당 |
| Vue+Pinia 실행 방식 | VueStudy 레포처럼 `vue create`로 빌드하는 별도 프로젝트 | Thymeleaf 화면 안에서 CDN `<script>`로 Vue/Pinia를 바로 불러와 사용 (빌드 도구 없이 동작) |
| 서버→클라이언트 값 전달 | `[[${값}]]` 인라인 표현식 | `th:inline="javascript"` + `/*[[${값}]]*/` 주석 조합으로 JS 변수에 값 심기 |
| DB 접속 정보 | yml에 `${DB_URL}` 등 환경변수 참조(Day07) | 그 환경변수를 어디서 채워주는지(`/etc/environment`, Docker `-e`, `.env`) 실습 |

---

## 1. Recipe 모듈 JPA → MyBatis 전환

### Entity에서 순수 VO로
```java
// 이전(Day04): @Entity + @Table + @Id
// 오늘: 어노테이션 없는 순수 VO (import는 남아있지만 실제로는 미사용)
@Data
public class RecipeVO {
	private int no;
	private String title, poster, chef, link;
	private int hit;
}
@Data
public class RecipeDetailVO {
	private int no;
	private String poster, title, chef, chef_poster, chef_profile, info1, info2, info3, content, foodmake;
}
```
- `RecipeDetailVO`는 Day05에서 정리했던 `RecipeDetail` Entity와 컬럼 구성이 완전히 동일함. 처리 방식만 JPA → MyBatis로 바뀜
- `ChefVO`, `FoodVO`도 같은 방식으로 VO 전환됨
- `FoodMapper`는 아직 내용 없이 껍데기만 생성된 상태임

### RecipeMapper의 어노테이션 SQL + XML SQL 혼합
```java
@Mapper
@Repository
public interface RecipeMapper {
	public List<RecipeVO> recipeListData(int start);   // ↔ XML의 <select id="recipeListData">
	public int recipeCount();                           // ↔ XML의 <select id="recipeCount">

	@Update("UPDATE recipe SET hit=hit+1 WHERE no=#{no}")   // 어노테이션으로 직접 SQL 작성
	public void hitIncrement(int no);

	@Select("SELECT * FROM recipeDetail WHERE no=#{no}")     // 어노테이션 방식
	public RecipeDetailVO recipeDetailData(int no);
}
```
- 목록/카운트처럼 조건(WHERE)이 복잡한 쿼리는 XML에 둠. 단순한 UPDATE/SELECT는 `@Update`/`@Select` 어노테이션으로 소스 코드 안에 바로 작성함
- 쿼리 복잡도에 따라 두 방식을 섞어 씀

### recipe-mapper.xml에서 재사용 SQL 조각(`<sql>`) 활용
```xml
<!-- recipe와 recipeDetail 양쪽에 모두 no가 존재하는 것만 노출 (상세 데이터 없는 목록 제외) -->
<sql id="where-no">
  WHERE no IN (SELECT no FROM recipe
               INTERSECT
               SELECT no FROM recipeDetail)
</sql>

<select id="recipeListData" resultType="com.sist.web.vo.RecipeVO" parameterType="int">
    SELECT no,poster,title,chef
    FROM recipe
    <include refid="where-no"/>
    ORDER BY no DESC
    OFFSET #{start} ROWS FETCH NEXT 12 ROWS ONLY
</select>

<select id="recipeCount" resultType="int">
    SELECT COUNT(*)
    FROM recipe
    <include refid="where-no"/>
</select>
<!-- 상세보기, 검색(Pinia)은 주석만 남아있고 아직 구현 전 -->
```
- `INTERSECT`로 두 테이블에 공통으로 존재하는 `no`만 필터링하는 서브쿼리를 `<sql id="where-no">`로 뺌
- 목록 조회와 카운트 조회 양쪽에서 `<include>`로 재사용해 쿼리 중복을 없앰
- ⚠️ **코드 확인 필요**: `RecipeServiceImpl.recipeListData()`에서 `start`를 `(page*ROWSIZE)-ROWSIZE`로 계산해놓고, 실제 Mapper 호출은 `rMapper.recipeListData(page)`로 `page`를 그대로 넘기고 있음. XML의 `#{start}`와 의도가 어긋나 있어서, 다시 볼 때 `start`를 넘기도록 고치는 게 맞아 보임. 오늘 자료에 그대로 있던 부분이라 원본 그대로 기록해둠

### Service/RestController 반환 구조는 이전 패턴 그대로
```java
// RecipeServiceImpl
rMapper.hitIncrement(no);              // 상세 조회 시 조회수 먼저 증가
return rMapper.recipeDetailData(no);   // 증가 반영된 값 재조회 없이 바로 반환(주의: 증가 전 값을 반환할 수 있음)

// pages[] 계산은 Day04~05와 동일한 BLOCK=10 공식 재사용
```
```java
// RecipeRestController: /recipe/list_vue, /recipe/detail_vue
// → Day05 정리 때 "SpringBoot 쪽에 추가 구현 필요"로 남겨뒀던 바로 그 두 API가 오늘 실제로 구현됨
```
> Vue 레포(VueStudy) Day01에서 정리했던 "list_vue/detail_vue가 백엔드에 없다"는 메모가 오늘 자료로 해소됨. 두 레포 진행 상황이 여기서 맞춰짐

### RecipeController는 화면 진입만 담당
```java
@GetMapping("/recipe/list")
public String recipe_list() { return "recipe/list"; }

@GetMapping("/recipe/detail")
public String recipe_detail(@RequestParam("no") int no, Model model) {
    model.addAttribute("no", no);   // Vue가 읽어갈 값 하나만 Model에 태움
    return "recipe/detail";
}
```

---

## 2. CDN 방식 Vue + Pinia (빌드 없이 Thymeleaf 화면에 바로 삽입)

```html
<script src="https://unpkg.com/vue@3.3.4/dist/vue.global.js"></script>
<script src="https://unpkg.com/axios/dist/axios.min.js"></script>
<script src="https://unpkg.com/vue-demi"></script>
<script src="https://unpkg.com/pinia@2.1.7/dist/pinia.iife.prod.js"></script>
<script src="/vue/axios.js"></script>              <!-- 프로젝트 내부 정적 리소스 -->
<script src="/vue/recipe/recipeStore.js"></script>  <!-- Pinia store 파일도 static 리소스로 서빙 -->
```
```javascript
const {createApp, onMounted, ref} = Vue     // 전역 Vue 객체에서 구조분해
const {createPinia} = Pinia
const recipeApp = createApp({
    setup(){
        const store = useRecipeStore()       // recipeStore.js에서 정의된 함수
        onMounted(()=>{ store.recipeListData() })
        return { store }
    }
})
recipeApp.use(createPinia())
recipeApp.mount(".container")
```
| 구분 | VueStudy 레포(빌드형) | 오늘(CDN형) |
|------|------------------------|---------------|
| 설치 | `npm install` 필요 | 설치 없이 `<script>` 태그로 즉시 사용 |
| 파일 구조 | `.vue` SFC(Single File Component) | HTML 안에 `<script>` 블록으로 직접 작성 |
| 라우팅 | `vue-router`로 SPA 라우팅 | Thymeleaf Controller가 페이지별 URL을 담당(`/recipe/list`, `/recipe/detail`), Vue는 각 페이지 안에서만 동작 |
| 용도 | 완전한 SPA 프론트엔드 프로젝트 | 서버 렌더링 페이지에 부분적으로 반응형 UI를 끼워 넣는 용도 (Day05의 Thymeleaf+Vue 혼합 패턴의 CDN 버전) |

### th:inline="javascript"로 서버 값을 JS에 안전하게 전달
```html
<script th:inline="javascript">
const NO = /*[[${no}]]*/ 0   // 주석 안에 EL을 넣고, 주석 밖의 0은 Thymeleaf 파싱 전 기본값(placeholder)
</script>
```
```javascript
onMounted(()=>{
    store.no = NO
    store.recipeDetailData()
})
```
- Day05에서 썼던 `const chef_name='[[${chef}]]'`는 값을 문자열 따옴표로 감싸는 방식임. `th:inline="javascript"` + `/*[[${}]]*/` 주석 방식은 이와 달리 숫자/객체 등 타입을 그대로 유지해서, 문자열이 아닌 값에 더 적합함
- 화면단 페이징: `store.range`, `store.move(i)`처럼 페이지 배열 생성과 이동 로직 자체를 Pinia store 안으로 옮김. 화면(HTML)은 `store.xxx`를 호출만 하는 구조로 단순화됨
- Day04의 `range()` 함수는 컴포넌트에 있었음. 오늘은 store로 이동함

---

## 3. Docker 배포 시 환경변수 주입 3단계 실습

### ① `/etc/environment`에 직접 등록 (시스템 전역)
```bash
sudo nano /etc/environment
# 아래 3줄 추가
DB_URL="jdbc:oracle:thin:@<서버IP>:1521:XE"
DB_USERNAME="<DB계정>"
DB_PASSWORD="<DB비밀번호>"
```
```bash
sudo ./gradlew clean build              # 전체 빌드
sudo ./gradlew clean build -x test      # 테스트 제외하고 빌드 (Day05 워크플로우에서도 사용했던 옵션)
sudo systemctl status docker            # 도커 서비스 상태 확인
```

### ② `docker run -e`로 컨테이너 실행 시 값 하나씩 주입
```bash
docker run -d -p 8080:8080 \
  -e DB_URL="jdbc:oracle:thin:@<서버IP>:1521:XE" \
  -e DB_USERNAME="<DB계정>" \
  -e DB_PASSWORD="<DB비밀번호>" \
  --name pinia-app <이미지ID>
```
- 기존 컨테이너 정리는 지금까지와 동일하게 `docker stop` → `docker rm` 순서로 진행함

### ③ `.env` 파일로 값 분리 + `docker run`에서 변수명만 참조 (오늘의 핵심)
```bash
sudo nano .env
```
```env
DB_URL=jdbc:oracle:thin:@<서버IP>:1521:XE
DB_USERNAME=<DB계정>
DB_PASSWORD=<DB비밀번호>
```
```bash
sudo cat .env      # 값 확인

docker run -d -p 8080:8080 \
  -e DB_URL -e DB_USERNAME -e DB_PASSWORD \
  --name pinia-app pinia-app
```
> 포인트: `-e DB_URL`처럼 값을 직접 안 쓰고 변수명만 적으면, 현재 쉘 세션에 로드돼 있는 환경변수 값을 그대로 컨테이너에 전달함. `docker run` 명령 자체에는 실제 비밀번호가 노출되지 않음. 단 `.env`를 `source` 하거나 `export`한 뒤 실행한다는 전제가 붙음

### 로그 확인
```bash
docker logs -f pinia-app   # 실시간 로그(에러 포함) 출력, -f로 계속 tail
```

### ①→③ 방식 비교
| 단계 | 저장 위치 | 노출 위험 | 특징 |
|------|-----------|-----------|------|
| ① `/etc/environment` | 시스템 전역 파일 | 서버 접근 가능한 사람에게 노출 | 서버 자체에서 실행하는 jar에 적용, 재부팅해도 유지 |
| ② `docker run -e "값"` | 쉘 히스토리, `docker inspect` 결과 | 명령어 자체에 평문 노출, `history` 명령으로 노출 가능 | 즉시 테스트용으로는 간단 |
| ③ `.env` + `-e 변수명` | `.env` 파일(별도 `.gitignore` 필요) | 파일 권한 관리로 최소화 가능 | 명령어에는 값이 안 보임, Git에 실수로 올리지 않도록 `.gitignore` 필수 |

> 오늘 배운 교훈(사용자 메모): "아이피 및 API는 항상 가릴 것". `.env`나 `/etc/environment`에 넣는 것과 별개로, **학습 정리/커밋/캡처 어디에도 실제 IP·계정정보·API 키를 그대로 남기지 않기**. `.env` 파일 자체도 `.gitignore`에 반드시 추가해서 레포에 올라가지 않도록 확인.

---

## 4. 다시 만들 때 체크리스트

```text
[MyBatis 전환]
① Entity(@Entity) → 순수 VO로 변경, DB 매핑 책임을 XML/Mapper로 이동
② 복잡한 조건(WHERE, 서브쿼리)은 XML의 <select>+<sql id="">+<include>로 재사용
③ 단순 UPDATE/SELECT는 Mapper 인터페이스에 @Update/@Select로 바로 작성
④ Service에서 계산한 값(start 등)이 실제 Mapper 호출 인자와 일치하는지 항상 재확인

[CDN Vue+Pinia]
⑤ 빌드 도구 없이 쓸 때는 vue.global.js + pinia.iife.prod.js(+vue-demi) CDN을 순서대로 로드
⑥ Pinia store 파일(recipeStore.js)도 정적 리소스 경로(/vue/...)로 분리해서 여러 화면에서 재사용
⑦ 서버 값을 JS로 넘길 때: 문자열이면 [[${}]] + 따옴표, 숫자/원본 타입 유지가 필요하면 th:inline="javascript" + /*[[${}]]*/
⑧ 페이지네이션 로직(range/move)은 컴포넌트가 아니라 store 안에 두고 화면은 호출만

[Docker 환경변수 배포]
⑨ 운영 서버: /etc/environment에 등록 → 시스템 전역에서 사용
⑩ 테스트: docker run -e KEY="값"으로 즉시 주입 (단, 쉘 히스토리에 값이 남는 점 주의)
⑪ 권장: .env 파일 작성 → 쉘에 로드 → docker run -e KEY(값 생략)로 변수명만 전달
⑫ .env는 반드시 .gitignore에 추가, 커밋/문서화 시 실제 IP·계정·키는 전부 마스킹 처리
⑬ docker logs -f 컨테이너명 으로 배포 후 에러 여부 바로 확인하는 습관
```

---

