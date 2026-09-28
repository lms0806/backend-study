# IoC

## 한 줄 정의

IoC(Inversion of Control, 제어의 역전)는 객체의 생성과 생명주기, 의존 관계 연결을 개발자 코드가 아니라 외부 컨테이너가 맡는 방식이다.

"필요한 객체를 내가 `new`로 만들어서 쓴다"에서 "필요한 객체는 컨테이너가 준비해 두고, 나는 받아서 쓴다"로 제어 방향이 바뀐다.

## 직접 제어와 비교

제어가 코드에 있는 경우:

```java
public class OrderService {
    private final PaymentClient paymentClient = new PaymentClient();
    private final OrderRepository orderRepository = new JpaOrderRepository();
}
```

`OrderService`가 구현체를 알고, 생성 시점을 정하고, 교체 방법도 스스로 가진다. 테스트에서 결제 클라이언트를 가짜로 바꾸려면 이 클래스를 고쳐야 한다.

제어가 컨테이너에 있는 경우:

```java
@Service
public class OrderService {
    public OrderService(PaymentClient paymentClient, OrderRepository orderRepository) {
        this.paymentClient = paymentClient;
        this.orderRepository = orderRepository;
    }
}
```

`OrderService`는 "어떤 구현이 들어오는가"를 모른다. 스프링 컨테이너가 구현체를 고르고, 필요한 시점에 만들고, 생성자로 넘긴다.

## 역전되는 것

- **생성 시점**: 언제 `new` 할지는 컨테이너 설정과 스코프가 결정한다.
- **구현체 선택**: 인터페이스 뒤에 어떤 클래스를 둘지는 설정·컴포넌트 스캔·프로필이 결정한다.
- **생명주기**: 초기화 콜백과 소멸 콜백을 컨테이너가 호출한다.
- **부가 기능 결합**: 트랜잭션 프록시처럼 객체 조립 과정에서 감싼다.

의존성 주입(DI)은 IoC를 실현하는 대표적인 방법이다. IoC가 원칙이고, DI는 그 원칙을 의존 객체를 외부에서 넣어 주는 방식으로 구현한 것이다. 서비스 로케이터(객체가 컨테이너에 직접 물어봐서 꺼내는 방식)도 넓은 의미의 IoC에 들어가지만, 스프링이 권하는 기본 스타일은 DI다.

## 스프링에서 컨테이너의 역할

스프링의 IoC 컨테이너는 `BeanFactory`와 그 하위인 `ApplicationContext`다. 실무에서 쓰는 것은 `ApplicationContext`다.

하는 일:

1. 설정(`@Configuration`, 컴포넌트 스캔, XML 등)을 읽어 빈 정의를 모은다.
2. 빈을 생성하고 의존성을 채운다.
3. 초기화 콜백을 호출하고, 필요하면 AOP 프록시로 감싼다.
4. 이름 또는 타입으로 빈을 찾아 준다.
5. 컨테이너가 닫힐 때 소멸 콜백을 호출한다.

`ApplicationContext`는 `BeanFactory`의 빈 생성·조회에 더해 메시지 국제화, 이벤트 발행(`ApplicationEventPublisher`), 환경 설정(`Environment`)을 포함한다. 그래서 "스프링 컨테이너"라고 하면 보통 애플리케이션 컨텍스트를 말한다.

진입점은 Spring Boot의 `SpringApplication.run()`이다. 이 호출이 컨텍스트를 띄우고, 스캔된 `@Component`와 자동 설정을 빈으로 등록한 뒤 웹 서버를 시작한다.

## 얻음과 비용

얻는 것:

- 클래스가 구현체가 아니라 역할(인터페이스)에 의존해 교체가 쉽다.
- 테스트에서 테스트용 구현이나 Mock을 넣기 쉽다.
- 생성, 트랜잭션, 라이프사이클이 한 정책으로 모인다.

비용:

- 실행 흐름이 코드의 `new`만 따라가서는 안 보인다. 어떤 빈이 주입됐는지 설정과 컨테이너를 같이 봐야 한다.
- 컨테이너 없이 그 클래스를 그냥 `new`하면 주입·AOP·라이프사이클이 빠진다.

## 자주 나오는 질문

1. IoC와 DI의 차이는? IoC는 제어를 프레임워크로 넘긴다는 원칙이다. DI는 의존 객체를 외부에서 주입해 그 원칙을 구현하는 기법이다.
2. 라이브러리와 프레임워크 차이와 어떻게 연결되나? 라이브러리는 내 코드가 호출한다. 프레임워크는 내 코드를 호출한다. 스프링이 컨트롤러와 빈 메서드를 대신 호출하는 구조가 IoC다.
3. `new`를 쓰면 항상 잘못인가? 값 객체, 메서드 안에서만 쓰고 버리는 짧은 수명의 객체, 컨테이너가 알 필요가 없는 객체는 `new`가 맞다. 서비스·리포지토리처럼 교체와 공통 정책이 필요한 협력 객체를 컨테이너에 맡긴다.
