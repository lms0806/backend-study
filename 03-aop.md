# AOP

## 한 줄 정의

AOP(Aspect-Oriented Programming, 관점 지향 프로그래밍)는 여러 비즈니스 로직에 흩어지는 공통 관심사를 한곳으로 모으는 방법이다.

로깅, 트랜잭션, 권한 검사, 성능 측정처럼 핵심 기능은 아닌데 많은 메서드에 반복되는 코드를 관점(aspect)으로 분리한다.

## 왜 필요한가

주문 서비스의 `placeOrder()`마다 시작 로그, 권한 확인, 트랜잭션 시작/커밋, 소요 시간 측정을 직접 넣으면 비즈니스 코드가 묻힌다. 정책을 바꿀 때도 호출부를 전부 수정해야 한다.

AOP는 "어떤 지점에, 어떤 조건으로, 어떤 부가 로직을 끼울지"를 분리한다. Spring에서 `@Transactional`이 대표적이다. 개발자는 어노테이션만 붙이고, 실제 begin/commit/rollback은 트랜잭션 관점이 처리한다.

## 용어

| 용어 | 의미 |
| --- | --- |
| Aspect | 공통 관심사를 묶은 모듈. 포인트컷과 어드바이스의 조합. |
| Join point | 관점을 끼워 넣을 수 있는 지점. Spring AOP에서는 메서드 실행. |
| Pointcut | 조인 포인트 중 어디에 적용할지 고르는 표현식. |
| Advice | 그 지점에서 실행할 부가 로직. |
| Target | 원래의 비즈니스 객체. |
| Proxy | 타깃을 감싸 어드바이스를 먼저/나중에 실행하는 객체. 호출자는 프록시를 만난다. |
| Weaving | 관점을 대상 코드에 엮는 시점. Spring AOP는 런타임에 프록시로 엮는다. |

## 어드바이스 종류

- **Before**: 메서드 실행 전. 예: 파라미터 검증 로그.
- **AfterReturning**: 정상 반환 후. 예: 결과 크기 기록.
- **AfterThrowing**: 예외가 나온 뒤. 예: 에러 추적 ID 기록.
- **After**: 성공·실패와 관계없이 메서드가 끝난 뒤. finally에 가깝다.
- **Around**: 전후를 모두 감싼다. `ProceedingJoinPoint.proceed()`를 호출해야 원래 메서드가 실행된다. 트랜잭션, 시간 측정, 캐시처럼 실행 여부를 가로채야 할 때 쓴다.

Around 안에서 `proceed()`를 호출하지 않으면 타깃 메서드는 실행되지 않는다. 두 번 호출하면 타깃도 두 번 실행된다.

## Spring AOP는 프록시 기반이다

Spring은 AspectJ처럼 클래스 바이트코드를 컴파일 시점에 바꾸는 것이 기본이 아니다. 빈을 만들 때 프록시를 만들어 컨테이너에 넣는다.

- 인터페이스가 있으면 JDK 동적 프록시(인터페이스 구현체)를 쓸 수 있다.
- 구체 클래스만 있으면 CGLIB가 하위 클래스를 만들어 메서드를 오버라이드한다.
- `spring.aop.proxy-target-class=true`이면 인터페이스가 있어도 CGLIB를 쓴다. Spring Boot의 기본에 가깝다.

프록시는 **바깥에서 호출된 public 메서드**에만 동작한다.

### 자기 호출 문제

```java
@Service
public class OrderService {
    public void create() {
        save(); // this.save(). 프록시를 거치지 않는다.
    }

    @Transactional
    public void save() { }
}
```

`create()` 안의 `save()`는 같은 객체의 내부 호출이라 프록시를 타지 않고, `@Transactional`이 적용되지 않는다. 분리된 다른 빈을 통해 호출하거나, 자기 자신을 주입받은 프록시로 호출해야 한다. `AopContext.currentProxy()`는 노출 설정을 켠 경우에만 쓸 수 있고, 설계가 프록시에 묶이므로 남용하지 않는다.

private 메서드는 프록시가 가로챌 수 없다. CGLIB도 상속으로 오버라이드할 수 있는 메서드만 대상이다. final 메서드·클래스도 프록시 적용이 막힌다.

## 포인트컷 예시

```java
@Aspect
@Component
public class LoggingAspect {

    @Around("execution(* com.example.app.service..*(..))")
    public Object log(ProceedingJoinPoint joinPoint) throws Throwable {
        long start = System.nanoTime();
        try {
            return joinPoint.proceed();
        } finally {
            long elapsedMs = (System.nanoTime() - start) / 1_000_000;
            // 메서드 시그니처와 elapsedMs를 로그로 남긴다.
        }
    }
}
```

`execution(* com.example.app.service..*(..))`는 해당 패키지와 하위 패키지의 모든 반환 타입, 모든 메서드, 모든 파라미터를 가리킨다. `@within`, `@annotation`으로 특정 어노테이션이 붙은 클래스·메서드만 좁힐 수 있다.

여러 어드바이스의 순서는 `@Order` 또는 `Ordered`로 정한다. 숫자가 낮을수록 바깥쪽에서 먼저 들어온다. 트랜잭션과 로깅이 같이 있으면 순서가 로그에 커밋 전후가 어떻게 보이는지를 바꾼다.

## AOP로 풀기 좋은 것 / 아닌 것

좋은 후보: 트랜잭션 경계, 감사 로그, 메서드 실행 시간, 반복되는 권한 체크, 캐시 조회.

비즈니스 분기의 핵심 규칙은 관점으로 숨기지 않는 편이 낫다. 호출부만 봐서는 무슨 일이 일어나는지 알 수 없게 되기 때문이다. 포인트컷이 너무 넓으면 예상하지 못한 메서드까지 감싸 성능과 디버깅이 어려워진다.

## 자주 나오는 질문

1. `@Transactional`을 붙였는데 롤백이 안 된다. 같은 클래스 내부 호출이거나, 프록시를 타지 않는 private/final이거나, 체크 예외가 기본 롤백 대상이 아닌 경우를 먼저 의심한다.
2. Spring AOP와 AspectJ의 차이는? Spring AOP는 스프링 빈의 메서드 실행만 프록시로 다룬다. AspectJ는 필드 접근, 객체 생성 등 더 많은 조인 포인트와 컴파일/로드 타임 위빙이 가능하다.
3. 프록시 객체와 타깃 객체는 같은 인스턴스인가? 호출자가 주입받는 것은 프록시다. 타깃은 프록시 안에 있다.
