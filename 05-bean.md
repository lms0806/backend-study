# Bean

## 한 줄 정의

Bean은 스프링 IoC 컨테이너가 생성하고, 관리하고, 필요한 곳에 넣어 주는 객체다.

평범한 `new`로 만든 객체와 달리, 빈은 컨테이너의 등록 정보(빈 정의)에 따라 스코프, 의존성, 초기화, 소멸, AOP 프록시가 적용된다.

## 등록 방법

**컴포넌트 스캔**

클래스에 스테레오타입 어노테이션을 붙이면 스캔 대상이 된다.

| 어노테이션 | 용도 |
| --- | --- |
| `@Component` | 일반적인 스프링 관리 컴포넌트 |
| `@Service` | 비즈니스 로직. `@Component`의 특수화 |
| `@Repository` | 저장소. 영속 계층 예외를 스프링의 `DataAccessException`으로 바꾸는 번역 기능이 붙는다 |
| `@Controller`, `@RestController` | 웹 요청을 받는 계층 |

`@Service`와 `@Component`의 런타임 차이는 거의 없고, 역할을 드러내는 표시에 가깝다. `@Repository`와 `@RestController`는 추가 기능이 있다.

**자바 설정**

```java
@Configuration
public class AppConfig {
    @Bean
    public Clock clock() {
        return Clock.systemUTC();
    }
}
```

서드파티 클래스처럼 어노테이션을 붙일 수 없는 객체, 생성 과정이 복잡한 객체는 `@Bean` 메서드로 등록한다. `@Configuration` 클래스의 `@Bean` 메서드 호출은 컨테이너가 가로채서 같은 빈을 재사용한다(싱글톤인 경우).

빈 이름을 생략하면 메서드 이름 또는 클래스명을 기반으로 정해진다. `@Qualifier`나 이름 주입으로 구분해야 하면 명시적인 이름이 낫다.

## 스코프

| 스코프 | 인스턴스 |
| --- | --- |
| `singleton` (기본값) | 컨테이너에 하나. 모든 주입이 같은 객체를 받는다. |
| `prototype` | 요청할 때마다, 주입 시점마다 새 객체. 컨테이너는 만든 뒤 소멸 콜백을 관리하지 않는다. |
| `request` | HTTP 요청마다 하나. 웹 애플리케이션. |
| `session` | HTTP 세션마다 하나. |
| `application` | 서블릿 컨텍스트마다 하나. |

싱글톤 빈은 필드에 요청별 상태를 두면 동시 요청이 그 상태를 공유한다. 싱글톤에는 불변 의존성만 두고, 요청 데이터는 메서드 파라미터나 지역 변수로 다룬다.

싱글톤이 프로토타입 빈을 생성자 주입으로 받으면, 주입은 싱글톤 생성 때 한 번이라 프로토타입도 그 시점에 만들어진 하나가 유지된다. 매번 새 프로토타입이 필요하면 `ObjectProvider<T>`나 `jakarta.inject.Provider<T>`로 조회 시점을 늦춘다.

## 생명주기

싱글톤 빈 기준 순서는 다음과 같다.

1. 생성자 호출. 이 시점에 생성자 주입이 끝난다.
2. 필드·세터 주입. `@Autowired`가 생성자 이외에도 있다면 이 단계.
3. `BeanPostProcessor`의 postProcessBeforeInitialization. 여기서 AOP 프록시 준비가 일어난다.
4. `@PostConstruct` 또는 `InitializingBean.afterPropertiesSet()`, 커스텀 init 메서드.
5. `BeanPostProcessor`의 postProcessAfterInitialization. 실제 프록시로 감싸 반환하는 지점이다.
6. 사용.
7. 컨테이너 종료 시 `@PreDestroy`, `DisposableBean.destroy()`.

`@PostConstruct`에서는 의존성이 이미 채워져 있다. 생성자에서 다른 빈의 초기화가 끝났다고 가정하면 안 된다. 생성 순서는 의존 관계로 결정되며, 순환이 있으면 세터/필드 주입에서만 제한적으로 풀리고 생성자 주입은 실패한다.

## 조회와 충돌

같은 타입의 빈이 둘이면 주입이 실패한다. 해결은 다음 중 하나다.

- `@Primary`로 기본 빈을 지정한다.
- `@Qualifier("name")`으로 이름을 지정한다.
- 파라미터 이름을 빈 이름과 맞춘다. 컴파일 시 파라미터 이름이 유지되어야 한다.
- 필요한 쪽만 `@Profile`로 활성화한다.

없는 빈을 필수 주입하면 컨텍스트 기동에 실패한다. 선택 의존은 `ObjectProvider`나 `Optional` 파라미터로 둔다. 필드에 `required = false`를 쓰는 방식은 생성자 주입보다 드러나기 어렵다.

## 등록됐다고 바로 쓸 수 있는 것은 아니다

컨테이너 안에 있는 동안만 스프링의 기능이 붙는다. `@Transactional`, `@Cacheable`은 빈 메서드를 프록시로 부를 때 동작한다. 테스트에서 `new OrderService()`를 하면 빈이 아니다.

지연 초기화(`@Lazy`, `spring.main.lazy-initialization=true`)를 켜면 기동은 빨라지지만, 빈 생성 오류가 첫 호출 때로 밀린다. 기동 시 실패를 선호하면 기본값(즉시 초기화)이 맞다.

## 자주 나오는 질문

1. `@Component`와 `@Bean`의 차이는? `@Component`는 클래스 자신을 스캔으로 등록한다. `@Bean`은 설정 클래스가 반환한 객체를 등록한다. 내 코드는 전자, 외부 라이브러리 객체는 후자가 자연스럽다.
2. 싱글톤은 JVM에 하나인가? 스프링 싱글톤은 해당 `ApplicationContext` 안에서 하나다. 컨텍스트가 둘이면 인스턴스도 둘이다. GoF 싱글톤(클래스 로더당 하나, private 생성자)과는 범위가 다르다.
3. 빈은 기본적으로 스레드 안전한가? 컨테이너가 동시 접근을 막아 주지 않는다. 싱글톤을 동시에 여러 요청이 호출한다. 상태를 필드에 두지 않는 것으로 안전하게 만든다.
