<h1 style="text-align: center; ">JPA 사용시 N+1 쿼리 문제</h1>
<p style="text-align: right;"> updated 2020.08 </p>

## N+1가 무엇인가

## 왜 일어나는가?

findAll같은 것들이 어떻게 동작하는지 알아야한다 em.createQuery JPQL을 사용한다.

- ID 쿼리가 아닌 JPQL은 persistence context를 체크하지 않고, 항상 DB에 쿼리를 날림
- 쿼리결과를 영속성 컨텍스트에 덮어씀.

1. EAGER_LOADING
2. LAZY_LOADING 에서 참조 엔티티 사용 할 때.

## 해결책

> N+1은 ORM의 한계 중 하나. 완전히 극복한 방법은 없다.
> 이해하지 않고 사용하게되면 어떤 쿼리를 실행할지 모르기 애플리케이션의 성능에 치명적이다.
> 적재적소에 맞게 해결책을 사용해야함.

### 1. fetch join 사용

- 명시적 EagerLoading, FetchType.LAZY 무의미.
- 카티잔 곱으로 조인됨
- ManyToOne, OneToOne 관계의 A, B Entity에서 A를 기준으로 조회 시 A의 갯수만큼 가져오게됨 (중복된 값이 프로퍼티로 숨음, 주 엔티티객체에는 중복이 없음)
  - 페이징 하는데 문제가 없음
- 하지만 OneToMany, ManyToMany 관계의 A, B Entity에서 A를 기준으로 조회 시 (A의 갯수) x (B의 갯수) 만큼의 엔티티 객체가 조회됨.(중복된 값이 드러남)

  - 페이징 불가
    -> 따라서 JPQL fetch join 구문에는 limit, offset구문이없음. fetch join + Pageable 사용시 full scan해서 메모리에서 페이징함.( 메모리 과적, full scan 오버헤드)

- @OneToMany에서 둘 이상의 컬렉션을 페치 할 수 없다. simultaneously fetch multiple bag 오류 뜸
- 즉, ManyToOne or OneToOne 관계에서의 사용에 최적화 되어있음 -> 다른 경우는 중복 제거 필요.
  - 중복 제거 방법 1. DISTINCT 사용
  - 중복 제거 방법 2. Set 사용

> fetch join vs join
> 그냥 join 은 JPQL통해서 모든 데이터를 가져오지만, 연관관계를 고려하지 않고 select 의 대상으로 지정한 엔티티만 리턴됨.
> 리턴되지 않은 컬렉션을 사용하는 시점에서 N+1 쿼리 문제 그대로 발생

### 2. EntityGraph

- @Query 작성시 미리 연결될 엔티티를 지정. -> 명시적인 EagerLoading이 됨.
- fetch join과 크게 다를바 없음.
- 단 outer join이 사용된다.

### 3. @BatchSize

- 주 엔티티와 연관된 데이터들을 where in 구문을 통해서 지정된 갯수(size)만큼 반복해서 가져옴.
- 최적화 하려면 대략적인 데이터 갯수를 알아야함.
- Lazy 로딩 시에 첫번째 쿼리에서 먼저 size만큼의 참조엔티티를 가져오고 size이상의 엘리먼트에 접근할 때 추가적인 where in JPQL 이 일어남

### 4. FetchMode.subselect

- 두번의 쿼리 (주 엔티티 쿼리, 참조엔티티 where in (주엔티티 id 쿼리))로 해결하는 방법임.
- lazy 시에는 프록시 객체 사용시점에 두번째 쿼리가 실행된다.

#### **ManyToOne에는 fetch join을, OneToMany에는 FetchType.SUBSELECT 혹은 @BatchSize를 사용한다**
