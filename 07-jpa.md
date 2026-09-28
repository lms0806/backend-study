# JPA

## 한 줄 정의

JPA(Jakarta Persistence API, 예전 이름 Java Persistence API)는 자바 객체와 관계형 테이블을 매핑하는 표준 ORM API다. 구현체로 Hibernate가 가장 많이 쓰이고, Spring Data JPA는 그 위의 저장소 추상화다.

개발자는 SQL로 행을 읽고 객체를 조립하는 대신, 엔티티를 저장·조회·수정한다. 실제 SQL은 영속성 컨텍스트와 구현체가 만든다.

## 구성

| 개념 | 역할 |
| --- | --- |
| Entity | `@Entity`가 붙은 클래스. 테이블과 매핑된다. `@Id`가 필수다. |
| EntityManager | 엔티티를 저장하고 조회하는 영속성 작업의 API. |
| Persistence Context | 엔티티를 영속 상태로 보관하는 1차 저장소. 트랜잭션 범위와 맞물린다. |
| EntityManagerFactory | 보통 애플리케이션에 하나. 만드는 비용이 커서 공유한다. EntityManager는 더 가볍고 스레드 안전하지 않다. |

Spring Data JPA의 `JpaRepository`를 쓰면 `EntityManager`를 직접 다루지 않아도 된다. 트랜잭션이 시작될 때 영속성 컨텍스트가 열리고, 끝날 때 flush와 commit이 일어난다.

## 엔티티 상태

- **비영속(new)**: `new Order()` 직후. 컨텍스트와 무관하다.
- **영속(managed)**: `persist` 또는 조회로 컨텍스트에 들어간 상태. 더티 체킹 대상이다.
- **준영속(detached)**: 트랜잭션·컨텍스트가 닫힌 뒤. 변경해도 DB에 반영되지 않는다.
- **삭제(removed)**: `remove` 호출 후 commit 때 DELETE가 나간다.

스프링 웹에서 트랜잭션은 보통 서비스 메서드 경계다. 컨트롤러나 뷰에서 지연 로딩을 건드리면 컨텍스트가 이미 닫혀 `LazyInitializationException`이 난다. 필요한 데이터는 트랜잭션 안에서 로딩을 끝내거나, 조회 전용 DTO로 가져온다.

## 1차 캐시와 동일성

같은 영속성 컨텍스트에서 같은 ID를 두 번 조회하면 DB를 다시 타지 않고 같은 인스턴스를 돌려준다.

```java
Order a = em.find(Order.class, 1L);
Order b = em.find(Order.class, 1L);
// a == b
```

이 동일성 덕분에 컬렉션에 넣거나 변경을 추적하기 쉽다. 캐시는 컨텍스트 수명까지만 유지된다. 다른 트랜잭션의 최신 값을 항상 보는 전역 캐시가 아니다. 전역에 가깝게 쓰려면 2차 캐시를 따로 켜야 하고, 무효화 정책을 설계해야 한다.

## 쓰기지연과 더티 체킹

`persist`를 호출한 순간 INSERT가 바로 나가지 않을 수 있다. SQL은 보통 커밋 직전 flush 때 모아서 실행된다. 영속 엔티티의 필드를 바꾸면 `update()`를 호출하지 않아도, flush 때 스냅샷과 비교해 변경된 컬럼의 UPDATE를 만든다. 이것이 더티 체킹이다.

JPQL 벌크 수정·삭제는 영속성 컨텍스트를 거치지 않는다. 실행 뒤 메모리의 엔티티는 옛 값일 수 있어 `flush`와 `clear`를 고려한다.

## 로딩과 N+1

`@ManyToOne`, `@OneToOne`의 기본 fetch는 EAGER다. `@OneToMany`, `@ManyToMany`의 기본은 LAZY다. 컬렉션을 무조건 EAGER로 두면 목록 조회마다 연관 전부를 가져와 쿼리와 메모리가 커진다. 기본은 LAZY로 두고, 유스케이스마다 fetch join이나 `@EntityGraph`로 필요한 만큼만 당긴다.

N+1은 목록 1번 조회 뒤, 각 원소의 지연 연관을 건드릴 때 추가 쿼리가 N번 나가는 문제다.

- fetch join: 한 번의 SQL로 가져온다. 컬렉션 fetch join을 여러 개 걸거나 페이지와 섞으면 행이 불어나고 메모리에서 페이징되는 함정이 있다.
- 배치 크기(`@BatchSize`, `default_batch_fetch_size`): LAZY를 유지하면서 IN 절로 묶는다.
- DTO 조회: 화면에 필요한 컬럼만 JPQL 생성자로 조회한다. 엔티티 그래프가 필요 없는 읽기에 적합하다.

## 연관관계의 주인

외래 키가 있는 쪽이 주인이다. `@ManyToOne` 쪽의 `@JoinColumn`이 주인인 경우가 많다. 반대편 `@OneToMany(mappedBy="...")`는 읽기 전용 거울이다. 주인 쪽을 바꾸지 않으면 FK가 갱신되지 않는다. 양방향이면 편의 메서드에서 양쪽 참조를 같이 맞춘다.

## Spring Data JPA에서 알아 둘 것

- 메서드 이름 쿼리(`findByStatusAndCreatedAtAfter`)는 조건이 길어지면 이름이 시그니처가 된다. 복잡해지면 `@Query`나 QueryDSL로 옮긴다.
- 저장은 `save`. 신규면 persist, 아니면 merge에 가깝게 동작한다. 이미 영속인 엔티티는 필드만 바꿔도 더티 체킹으로 충분하다.
- OSIV(`spring.jpa.open-in-view`, Boot 기본 true)는 요청이 끝날 때까지 영속성 컨텍스트를 연다. 뷰에서의 지연 로딩은 편하지만, 컨트롤러 밖에서도 쿼리가 나갈 수 있고 커넥션 점유 시간이 길어진다. 팀에서 끄기로 했다면 트랜잭션 안에서 로딩을 끝내야 한다.

## 자주 나오는 질문

1. JPA와 MyBatis를 어떻게 나누나? 객체 그래프를 저장·수정하고 더티 체킹이 이득인 도메인 쓰기는 JPA가 잘 맞는다. 복잡한 통계 SQL, DB 특화 힌트, 대규모 읽기 전용 쿼리는 MyBatis나 네이티브 쿼리가 드러내기 쉽다.
2. `equals`/`hashCode`를 모든 필드로 만들면? 더티 체킹이나 컬렉션 소속 중에 해시가 바뀐다. 식별자(비즈니스 키 또는 영속 후 ID) 기준으로 두고, 아직 ID가 없는 신규 엔티티를 `Set`에 넣을 때의 동작을 따로 정한다.
3. 지연 로딩은 추가 SELECT인가? 프록시/컬렉션 래퍼에 처음 접근할 때 SELECT가 나간다. 영속성 컨텍스트가 없으면 예외다.
