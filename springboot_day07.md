# 📘 Spring Boot Day 07 — 메서드 규칙 vs JPQL vs QueryDSL (Emp/Dept 검색 3방식 비교)

## 0. 핵심 빠른 참조 — 검색 쿼리 3가지 방식

| 구분 | 1. 메소드 규칙 | 2. JPQL (`@Query`) | 3. QueryDSL |
|------|----------------|----------------------|-------------|
| 원리 | 메서드 이름으로 SQL 자동 생성 | JPA가 제공하는 객체중심 SQL 문장 | 타입 안정성을 갖춘 쿼리 빌더(Q-class) |
| 장점 | SQL을 몰라도 사용 가능, 메서드명으로 가독성 좋음 | 표준 기술이라 별도 라이브러리 불필요, 엔티티(객체) 기반 | **타입 안정성** — 컴파일 시점에 오류 확인 가능 |
| 단점 | 메서드명이 길어질 수 있음(`findBySalGreaterThanAndEnameLikeAndJobLikeOrderByHiredateDesc`), **동적 쿼리 불가** | 문자열 기반이라 **오타 시 에러 처리가 어려움**, 동적 쿼리 어려움 | **초기 설정이 어려움**(Q-class 생성 필요), 문법이 까다로움 |
| 사용처 | 단순 조회 (`findByEmpno(int empno)`처럼 조회 목적 명확한 경우) | 복잡하지 않은 SQL / 수정·삭제 시 주로 사용 | **복잡한 조인 / 필터링 / 페이징** |
| 실무 비율 메모 | MyBatis와의 사용 빈도 비율 **8:2**로 언급 | — | — |

> 정리 결론(`EmpJpqlRepository` 상단 주석): 단순 조회는 메소드 규칙(Getter만 있는 interface로 SELECT *), 복잡한 동적 쿼리·검색은 QueryDSL/MyBatis, 수정·삭제·정적 쿼리는 JPQL. **용도별로 3가지를 섞어 쓰는 설계**임

---

## 1. Emp ↔ Dept 엔티티 관계 (`@ManyToOne`)

```java
@Entity
@Table(name="EMP")
@Getter @Setter
public class Emp {
	@Id
	private Integer empno;
	private String ename;
	private String job;
	private Integer mgr;       // column에 null 가능 → Integer(래퍼) 사용
	private Date hiredate;
	private int sal;
	private Integer comm;

	@ManyToOne
	@JoinColumn(name="deptno")
	private Dept dept;         // FK를 객체 참조로 매핑
}
```
```text
// Emp.java 상단 주석
1. DQL => SELECT => 검색 : 메소드 규칙
          findByName(String name) => eq
2. DML => INSERT / UPDATE / DELETE
             |         |        |
             ---------       -------
                | save      | delete
```
- `Dept`는 `deptno`(PK) / `dname` / `loc` 만 가진 단순 엔티티
- `Emp.dept`처럼 FK 컬럼을 **객체 참조**로 매핑함
- 그래서 SQL의 `JOIN`이 `emp.dept.dname` 같은 점 표기로 자동 처리됨

---

## 2. Q-class 생성 방법

```text
1. Window < Show View < Other < Gradle < Gradle Tasks
2. ./gradlew clean compileJava
3. project 폴더에서 Gradle > Refresh
4. project < Clean 설정

Q-class : 데이터베이스를 검색할 때 사용하는 Java 코드(자동 생성)
```
- 위 절차와 Q-class 설명은 `EmpQueryRepository` 상단 주석에 정리해둔 것임
- `QEmp.java`, `QDept.java`는 직접 작성하지 않음. 빌드 시 `@Generated("com.querydsl.codegen.DefaultEntitySerializer")`로 자동 생성되는 파일임
- 생성 안 될 때 체크리스트: `compileJava` 재실행 → Gradle Refresh → Clean 순서로 시도

### QueryFactoryConfig에서 Bean 등록
```java
@Configuration
public class QueryFactoryConfig {
	@PersistenceContext
	private EntityManager em;

	@Bean
	public JPAQueryFactory jpaQueryFactory() {
		return new JPAQueryFactory(em);
	}
}
```
- QueryDSL을 쓰려면 `JPAQueryFactory`를 Bean으로 등록해서 Repository에 주입받아야 함

---

## 3. `EmpMethodRepository` 메소드 규칙 SQL 총정리

```java
Emp findByEmpno(int empno);                       // WHERE empno=?
List<Emp> findByEname(String ename);               // WHERE ename=?  (equals)
List<Emp> findByEnameStartsWith(String ename);      // WHERE ename LIKE '?%'  (index 적용됨)
List<Emp> findByEnameEndsWith(String ename);        // WHERE ename LIKE '%?'
List<Emp> findByEnameContains(String ename);        // WHERE ename LIKE '%?%'

List<Emp> findBySalGreaterThanEqual(int sal);       // WHERE sal>=?
List<Emp> findBySalLessThanEqual(int sal);          // WHERE sal<=?
List<Emp> findBySalBetween(int min,int max);        // WHERE sal BETWEEN ? AND ?

List<Emp> findByJobAndSalGreaterThan(String job,int sal);  // WHERE job=? AND sal>?
List<Emp> findByJobOrEname(String job,String ename);        // WHERE job=? OR ename=?

List<Emp> findByDeptDname(String dname);            // 부서명으로 emp 검색 (JOIN 자동)
List<Emp> findByDeptLoc(String loc);                 // 근무지로 emp 검색
List<Emp> findByDeptDnameContains(String loc);       // 부서명 LIKE

List<Emp> findByOrderBySalDesc();                    // ORDER BY sal DESC
List<Emp> findTop3ByOrderBySalDesc();                // WHERE rownum<=3 ORDER BY sal DESC (Top-N)
List<Emp> findDistinctByJob(String job);             // 중복 제거

List<Emp> findByCommIsNull();                        // WHERE comm ISNULL
List<Emp> findByCommIsNotNull();                      // WHERE comm ISNOTNULL

List<Emp> findByDeptDeptnoIn(List<Integer> deptnos);  // IN (List.of(10,20,30) 형태로 전달)
List<Emp> findByJobNot(String job);                    // WHERE NOT job=?
```

---

## 4. JPQL `@Query` — `EmpJpqlRepository`

```java
@Query("SELECT e FROM Emp e") // Emp는 테이블이 아니라 Entity 객체명 → 반드시 별칭(e) 사용
public List<Emp> empListData();

@Query("SELECT e FROM Emp e WHERE e.empno=:empno")
public Emp empDetailData(@Param("empno") int empno);

@Query("SELECT e FROM Emp e WHERE e.ename LIKE CONCAT(:ename,'%')")   // A%
public List<Emp> empEnameStartsLike(@Param("ename") String ename);

@Query("SELECT e FROM Emp e WHERE e.ename LIKE CONCAT('%',:ename)")   // %A
public List<Emp> empEnameEndsLike(@Param("ename") String ename);

@Query("SELECT e FROM Emp e WHERE e.ename LIKE CONCAT('%',:ename,'%')") // %A%
public List<Emp> empLikeData(@Param("ename") String ename);

@Query("SELECT e FROM Emp e WHERE e.sal BETWEEN :min AND :max")
List<Emp> findBySalBetween(@Param("min") int min,@Param("max") int max);

@Query("SELECT e FROM Emp e JOIN e.dept d WHERE d.dname=:dname")
List<Emp> findByDeptDname(@Param("dname") String dname);

@Query("SELECT e FROM Emp e JOIN e.dept d WHERE d.dname LIKE CONCAT('%',:dname,'%')")
List<Emp> findByDeptDnameContains(@Param("dname") String loc);

@Query("SELECT DISTINCT e FROM Emp e WHERE e.job = :job")
List<Emp> findDistinctByJob(@Param("job") String job);

@Query("SELECT e FROM Emp e WHERE e.comm IS NULL")
List<Emp> findByCommIsNull();

@Query("SELECT e FROM Emp e WHERE e.job!=:job")
List<Emp> findByJobNot(@Param("job") String job);

@Query("SELECT e FROM Emp e WHERE e.dept.deptno IN :deptnos")
List<Emp> findByDeptDeptnoIn(@Param("deptnos") List<Integer> deptnos);
```
- 파라미터 바인딩은 `:이름` + `@Param("이름")` 조합
- `JOIN e.dept d`처럼 JPQL에서도 별칭이 필수임
- 주석에 남은 미완성 예시: `findTop3ByOrderBySalDesc()`용 `@Query("SELECT e FROM Emp e ORDER BY e.sal DESC")`
  - Top-N은 JPQL만으로는 제한이 애매해서 주석처리된 채 남아있음
  - 대신 QueryDSL의 `.limit()`로 해결

---

## 5. QueryDSL 쿼리 빌더 (`EmpQueryRepository`)

### 기본 구조
```java
QEmp emp = QEmp.emp;   // Q-class 인스턴스 얻기
return (Emp) queryFactory.from(emp)
                   .where(emp.empno.eq(empno))
                   .fetchOne();
```
```text
from(테이블 : Q-class 객체)
where(조건)
orderBy(컬럼.desc())
groupBy(컬럼)
having(조건)
```

### 비교연산자 (주석 표)
| 연산자 | 메서드 | 예시 | SQL |
|--------|--------|------|-----|
| `=` | `eq()` | `emp.sal.eq(3000)` | `sal=3000` |
| `<` | `lt()` | `emp.sal.lt(3000)` | `sal<3000` |
| `>` | `gt()` | `emp.sal.gt(3000)` | `sal>3000` |
| `<=` | `loe()` (LessThanEqual) | `emp.sal.loe(3000)` | `sal<=3000` |
| `>=` | `goe()` (GreaterThanEqual) | `emp.sal.goe(3000)` | `sal>=3000` |
| `!=` | `ne()` | `emp.sal.ne(3000)` | `sal<>3000` |

### 주요 패턴
```java
// LIKE 3종
emp.ename.startsWith(ename)   // A%
emp.ename.endsWith(ename)     // %A
emp.ename.contains(ename)     // %A%

// BETWEEN
.where(emp.sal.between(min, max))

// AND / OR (두 가지 표현 가능)
.where(emp.job.eq(job).and(emp.sal.gt(sal)))
.where(emp.job.eq(job), emp.sal.gt(sal))     // and 대신 콤마(,)로도 가능
.where(emp.job.eq(job).or(emp.sal.gt(sal)))

// 정렬 / Top-N
.orderBy(emp.sal.desc())
.orderBy(emp.sal.desc()).limit(3)             // Top-3

// DISTINCT (컬럼 선택 조회)
queryFactory.select(emp.sal).distinct().from(emp).fetch()

// NULL 체크
.where(emp.comm.isNull())     // / isNotNull()

// IN
.where(emp.dept.deptno.in(deptnos))

// NOT
.where(emp.job.ne(job))

// JOIN (부서명 조건)
QDept dept = QDept.dept;
queryFactory.from(emp).join(emp.dept, dept).where(dept.dname.eq(dname)).fetch()
queryFactory.from(emp).join(emp.dept, dept).where(dept.dname.contains(dname)).fetch()
```
> QueryDSL은 항상 `QEmp.emp` 같은 Q-class를 먼저 선언하고 `queryFactory.from(Q객체)`로 시작함. JPQL이 문자열로 쿼리를 쓴다면 QueryDSL은 메서드 체이닝으로 조건을 조립함

---

## 6. EmpController에서 세 방식 호출 실험

```java
@Controller
@RequiredArgsConstructor
public class EmpController {
	private final EmpMethodRepository eDao;   // 방식1
	private final EmpJpqlRepository eDao2;    // 방식2
	private final EntityManager em;           // JPQL 직접 실행용
	private final EmpQueryRepository eDao3;   // 방식3(QueryDSL)

	@GetMapping("/emp")
	public void emp_method() {
		// 아래 라인들을 하나씩 주석 해제하며 각 Repository의 결과를 콘솔로 확인하는 방식으로 학습
		// 예: eDao.findByEnameStartsWith("A") / eDao2.empLikeData("T") / eDao3.findByDeptDnameLike("개발팀")
		List<Emp> list = eDao3.findByDeptDnameLike("개발팀");
		System.out.println(list);
	}
}
```
- 화면 없이 콘솔 출력으로만 결과를 확인하는 실습임. 반환 타입이 `void`인 것도 이 때문
- `EntityManager`로 JPQL을 Repository 없이 직접 실행하는 방법도 주석으로 남아있음:
  ```java
  String jpql = "SELECT e FROM Emp e ORDER BY e.sal DESC";
  List<Emp> list = em.createQuery(jpql, Emp.class).setMaxResults(5).getResultList();
  ```

---

## 7. application.yml 민감정보를 환경변수로 분리

```yaml
spring:
  datasource:
    url: ${DB_URL}
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
    driver-class-name: oracle.jdbc.driver.OracleDriver
```
- Day01~06까지는 `url/username/password`를 yml에 직접 하드코딩했음
- 오늘부터는 `${환경변수명}` 형식으로 분리함
  - 레포에 DB 계정정보가 그대로 노출되지 않게 하는 보안 조치임
  - 실제 값을 OS 환경변수로 주입할지 Git Actions Secrets로 주입할지는 확인 필요

---

## 8. 다시 만들 때 체크리스트

```text
[QueryDSL 세팅]
① build.gradle에 QueryDSL 관련 의존성/플러그인 추가 확인
② ./gradlew clean compileJava 로 Q-class(QEmp, QDept 등) 생성
③ 생성 안 되면: Gradle Refresh → Project Clean 순으로 재시도
④ JPAQueryFactory를 @Configuration 클래스에서 @Bean으로 등록 (EntityManager 주입)

[Repository 3방식 선택 기준]
⑤ 단순 조회(PK/단일 컬럼 검색)는 메소드 규칙(findByXxx)
⑥ 정적이고 단순한 조건의 SQL, 수정/삭제는 JPQL(@Query)
⑦ 복잡한 동적 조건(여러 필터 조합, 조인, Top-N, distinct 등)은 QueryDSL

[QueryDSL 코드 패턴]
⑧ QEmp emp = QEmp.emp; 로 Q-class 선언 → queryFactory.from(emp).where(...).fetch()
⑨ 연산자는 eq/lt/gt/loe/goe/ne, LIKE는 startsWith/endsWith/contains
⑩ AND는 .and() 또는 where()에 조건 여러 개를 콤마로 나열해도 동일하게 동작
⑪ JOIN 필요 시 QDept dept = QDept.dept; → .join(emp.dept, dept)

[보안]
⑫ application.yml의 DB 계정정보는 ${DB_URL} 등 환경변수 참조로 분리, 실제 값은 서버 환경변수/Secrets로 관리
```

---

