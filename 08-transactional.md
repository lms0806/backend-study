# Transactional

## 한 줄 정의

`@Transactional`은 메서드(또는 클래스)의 실행을 하나의 트랜잭션 경계로 묶는 스프링 어노테이션이다.

구현은 트랜잭션 AOP 프록시다. 메서드에 들어오기 전에 트랜잭션을 열고, 정상 종료면 커밋하고, 롤백 규칙에 맞는 예외면 롤백한다.

## 기본 동작

```java
@Service
public class TransferService {
    @Transactional
    public void transfer(Long fromId, Long toId, long amount) {
        // 출금과 입금이 둘 다 반영되거나, 둘 다 반영되지 않는다.
    }
}
```

클래스에 붙이면 그 클래스의 public 메서드 전체에 적용된다. 메서드 어노테이션이 클래스보다 우선한다.

스프링 트랜잭션 매니저(`PlatformTransactionManager` 구현, JPA에서는 `JpaTransactionManager`)가 실제 JDBC 커넥션의 commit/rollback을 호출한다. `@EnableTransactionManagement`가 필요한데, Spring Boot는 자동으로 켠다.

## 프록시라서 생기는 한계

- public 메서드를 다른 빈이 호출할 때만 적용된다.
- 같은 클래스의 내부 호출(`this.method()`)에는 적용되지 않는다.
- private 메서드에는 적용되지 않는다.
- 자가 호출로 전파(propagation)를 나누려 해도 프록시를 타지 않으면 나뉘지 않는다.

자세한 이유는 [AOP](./03-aop.md)의 자기 호출을 보면 된다.

## 롤백 규칙

기본값:

- `RuntimeException`과 `Error`: 롤백
- 체크 예외(`Exception`의 자손 중 RuntimeException이 아닌 것): 커밋

체크 예외에도 롤백하려면 `rollbackFor`를 지정한다.

```java
@Transactional(rollbackFor = Exception.class)
public void place() throws IOException {
}
```

예외를 catch하고 다시 던지지 않으면 프록시는 정상 종료로 보고 커밋한다. catch 후 롤백이 필요하면 다시 던지거나, `TransactionAspectSupport.currentTransactionStatus().setRollbackOnly()`를 호출한다. 후자는 트랜잭션 API에 코드가 묶이므로, 실패를 예외로 전달하는 쪽이 읽기 쉽다.

## 전파 (propagation)

이미 트랜잭션이 있을 때 새 `@Transactional` 메서드가 어떻게 참여하는지를 정한다.

| 전파 | 동작 |
| --- | --- |
| `REQUIRED` (기본값) | 있으면 참여, 없으면 새로 연다. |
| `REQUIRES_NEW` | 항상 새 트랜잭션. 바깥 트랜잭션은 잠시 멈췄다가 안쪽이 커밋된 뒤 재개된다. 안쪽 커밋은 바깥 롤백과 무관하게 남는다. |
| `NESTED` | 바깥 트랜잭션 안에 savepoint를 둔다. JDBC savepoint가 될 뿐 별도 트랜잭션이 아니다. JPA 기본 경로에서는 기대를 확인해야 한다. |
| `SUPPORTS` | 있으면 참여, 없으면 트랜잭션 없이 실행. |
| `NOT_SUPPORTED` | 있으면 잠시 중단하고 트랜잭션 없이 실행. |
| `MANDATORY` | 반드시 기존 트랜잭션이 있어야 한다. 없으면 예외. |
| `NEVER` | 트랜잭션이 있으면 예외. |

`REQUIRES_NEW`는 "이력 로그는 본 작업이 롤백돼도 남긴다" 같은 요구에 쓴다. 안쪽 트랜잭션이 독립 커밋되므로 락과 커넥션을 하나 더 쓴다. 바깥이 잡은 같은 행을 안쪽에서 다시 쓰면 교착이나 자기 자신과의 락 대기가 날 수 있다.

## readOnly

```java
@Transactional(readOnly = true)
public Order get(Long id) {
    return orderRepository.findById(id).orElseThrow();
}
```

Hibernate는 flush 모드를 MANUAL에 가깝게 바꿔 불필요한 더티 체킹·flush를 줄인다. JDBC 드라이버나 DB에 따라 읽기 전용 힌트가 전달되기도 한다. 읽기 메서드에 붙여 두었다가 그 안에서 저장이 일어나면 flush가 생략되어 변경이 빠질 수 있다. 쓰기가 있으면 `readOnly = true`를 붙이지 않는다.

## 격리 수준과 타임아웃

`isolation`은 이 트랜잭션의 격리 수준을 지정한다. 기본값 `ISOLATION_DEFAULT`는 DB 기본값을 따른다. 수준별 이상 현상은 [트랜잭션 격리 수준](./09-transaction-isolation.md)에서 다룬다.

`timeout`은 초 단위다. 초과하면 트랜잭션이 롤백 대상이 된다. 오래 잡는 트랜잭션은 커넥션 풀과 락을 붙잡으므로, 외부 HTTP 호출을 트랜잭션 안에 넣지 않는 것이 원칙이다. 순서는 "트랜잭션 밖에서 외부 호출 → 짧은 트랜잭션으로 DB 반영"이 안전하다.

## 트랜잭션 범위 설계

한 트랜잭션은 일관성이 필요한 쓰기 묶음까지만 짧게 연다.

- 조회 여러 번과 무관한 메일 발송, 파일 업로드, 결제사 HTTP는 커밋 뒤로 뺀다.
- 커밋 전에 이벤트를 발행하면 리스너가 롤백된 데이터를 볼 수 있다. 커밋 이후가 필요하면 `TransactionalEventListener(phase = AFTER_COMMIT)`을 쓴다.
- 컨트롤러에 `@Transactional`을 붙이면 경계가 뷰 렌더링까지 늘어나기 쉽다. 서비스(또는 유스케이스) 계층에 둔다.

## 자주 나오는 질문

1. `@Transactional`을 붙였는데 INSERT가 두 메서드에서 각각 커밋된다. 호출이 자기 자신의 내부 호출이거나, 프록시가 아닌 대상(다른 스레드, `new`)일 수 있다.
2. 체크 예외인데 데이터가 들어갔다. 기본 롤백 규칙이다. `rollbackFor`가 필요하다.
3. 클래스와 메서드에 모두 있으면? 메서드 설정이 이긴다. 클래스의 `readOnly = true`를 쓰기 메서드에서 `readOnly = false`로 덮는 패턴이 흔하다.
