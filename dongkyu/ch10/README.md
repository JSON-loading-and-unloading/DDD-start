# Chapter10 이벤트

## 10.1 시스템 간 강결합 문제

- 두 바운디드 컨텍스트 간의 강결합이 존재하면 여러 문제가 발생하게 된다.

### 주문을 취소하면 환불 처리를 해야하는 예시

```java
public class CancelOrderService {
	private final RefundService refundService;

	@Transactional
	public void cancel(OrderNo orderNo) {
		// ...
		refundService.refund(order.getPaymentId());
		// ...
	}
}
```

- 이 경우 외부 서비스가 정상이 아닐 경우 트랜잭션 처리를 어떻게 해야 할지 애매하다.
- 환불을 처리하는 외부 시스템 응답 시간이 길어지면 대기 시간도 길어지기에 성능에 직접적인 영향을 받게 된다.
- 추가 요구 사항이 있을 경우 더 복잡해진다.
    - erx) 주문 취소 시 환불 뿐만 아니라 취소했다는 내용을 통지해야 한다는 추가 요구 사항
    - 로직이 더 비대해지고 트랜잭션 처리가 복잡해진다.

## 10.2 이벤트 개요

- 바운디드 컨텍스트 간 강결합은 이벤트를 사용해 해결할 수 있다.
- ‘~할 때’, ‘~가 발생하면’, ‘만약 ~하면’과 같은 요구사항은 도메인 상태 변경과 관련된 경우가 많고 이는 이벤트를 이용해 구현할 수 있다.

### 10.2.1 이벤트 관련 구성요소

- **이벤트**
- **이벤트 생성 주체**
    - 엔티티, 벨류, 도메인 서비스와 같은 도메인 객체
    - 로직을 실행해 상태가 바뀌면 이벤트를 발생 시킨다.
- **이벤트 디스패처(퍼블리셔)**
    - 이벤트 생성 주체와 이벤트 핸들러를 연결시킨다.
    - 구현 방식에 따라 이벤트 생성과 처리를 동기나 비동기로 실행한다.
- **이벤트 핸들러(구독자)**
    - 이벤트 생성 주체가 발생시킨 이벤트에 반응해 원하는 기능을 실행한다.

### 10.2.2 이벤트의 구성

- 이벤트 종류
    - 클래스 이름으로 이벤트 종류를 표현
    - ex) `ShippingInfoChangedEvent`
- 이벤트 발생 시간
- 추가 데이터
    - 이벤트와 관련된 정보
    - ex) 주문 번호, 신규 배송지 정보 등
    - 이벤트와 관련 없는 정보를 담을 필요는 없다.

### 10.2.3 이벤트 용도

- 첫 번째 용도는 **트리거**다.
    - 도메인 상태가 바뀔 때 다른 후처리가 필요할 때
    - ex) ‘주문 취소’를 트리거로 ‘환불 처리’
    - ex) ‘예매 완료’를 트리거로 ‘SMS 통지’
- 두 번째 용도는 **서로 다른 시스템 간의 동기화**이다.
    - ex) ‘배송지 변경’ 시 외부 배송 서비스에 바뀐 배송지 정보를 전송해야 한다.

### 10.2.4 이벤트 장점

- 이벤트를 사용하면 **서로 다른 도메인 로직이 섞이는 것을 방지**할 수 있다.

    ```java
    public class Order {
    	public void cancel() {
    		verifyNotYetShipped();
    		this.state = OrderState.CANCELED;
    		Events.raise(new OrderCanceldEvent(number.getNumber()));
    	}
    }
    ```

    - ‘주문’ 도메인에서 ‘결제’ 도메인으로의 의존도 제거했다.
- 이벤트를 사용하면 **기능 확장도 용이**하다.
    - 이벤트 생성 주체 코드는 건드릴 필요 없이 이벤트 핸들러만 추가하면 된다.

## 10.3 이벤트, 핸들러, 디스패처 구현

- 이벤트 관련 코드는 다음과 같다.
    - **이벤트 클래스**: 이벤트를 표현
    - **디스패처**: 스프링이 제공하는 `ApplicationEventPublisher`를 이용
    - **Events**: 이벤트를 발행하고`ApplicationEventPublisher`를 사용
    - **이벤트 핸들러**: 스프링이 제공하는 기능을 사용하여 이벤트를 수신 처리한다.

### 10.3.1 이벤트 클래스

```java
public class OrderCancedEvent {
	private String orderNumber;

	// 생성자, getter ...
}
```

- 이벤트 자체를 위한 상위 타입은 존재하지 않는다.
    - 모든 이벤트가 공통으로 갖는 프로퍼티가 있다면 상위 클래스를 만들어 상속하게 할 수도 있다.
- 이벤트 클래스 이름을 결정할 때 과거 시제를 사용하면 된다.
    - ex) `OrderCancedEvent`
- 이벤트 처리에 필요한 최소한의 데이터를 포함해야 한다.

### 10.3.2 Events 클래스와 ApplicationEventPublisher

- 이벤트를 발생시킬 때 스프링이 제공하는 `ApplicationEventPublisher`를 사용하면 된다.
    - `applicationEventPublisher.publishEvent(…)`
- 도메인 엔티티 객체 내부에서 이벤트를 발생시키도록 구현할 수도 있다.

    ```java
    public class Order {
    	public void cancel() {
    		verifyNotYetShipped();
    		this.state = OrderState.CANCELED;
    		Events.raise(new OrderCanceldEvent(number.getNumber()));
    	}
    }
    ```

    - `Events` 클래스는 `ApplicationEventPublisher`를 사용해 이벤트를 발생시키도록 구현한 것이다.

    ```java
    import org.springframework.context.ApplicationEventPublisher;
    
    public class Events {
    	private static ApplicationEventPublisher publisher;
    	
    	static void setPublisher(ApplicationEventPublisher publisher) {
    		Events.publisher = publisher;
    	}
    
    	public static void raise(Object event) {
    		if (publisher != null) {
    			publisher.publisherEvent(event);
    		}
    	}
    }
    ```

    - `Events` 클래스의 초기화는 아래와 같이 구현한다.

    ```java
    @Configuration
    public class EventsConfig {
    	// ApplicationContext는 ApplicationEventPublisher를 상속하고 있다.
    	@AutoWired
    	private ApplicationContext applicationContext;
    
    	@Bean
    	public InitializingBean eventsInitializer() {
    		return () -> Events.setPublisher(applicationContext);
    	}
    }
    ```

    - `InitializingBean` 인터페이스는 스프링 빈 객체를 초기화할 때 사용하는 인터페이스다.

### 10.3.3 이벤트 발생과 이벤트 핸들러

- 이벤트 처리는 스프링이 제공하는 `@EventListener` 애너테이션을 사용해 구현한다.

    ```java
    @EventListener(OrderCancedEvent.class)
    public void handle(OrderCanceldEvent event) {
    	refundService.refund(event.getOrderNumber());
    }
    ```


### 10.3.4 흐름 정리

1. 도메인 기능을 실행
2. 도메인 기능은 `Events.raise()`를 이용해 이벤트 발생시킴
3. `Events.raise()`는 `ApplicationEventPublisher`를 이용해 이벤트 출판
4. `ApplicationEventPublisher`는 `@EventListener` 메서드를 찾아 실행
- 이렇게만 처리하면 이벤트 발생과 이벤트 처리가 같은 트랜잭션 범위 내에서 실행된다.

## 10.4 동기 이벤트 처리 문제

- 이벤트를 이용해 강결합은 해결했지만 **외부 서비스에 영향 받는 문제는 남아있다.**
    - 동기적으로 이벤트가 처리되기에 외부 서비스의 응답 시간이 길어지면 성능에 문제가 발생하는 것은 아직 해결하지 못했다.
    - 같은 트랜잭션으로 묶이기에 외부 서비스에서 예외가 발생했을 때 이벤트 발생 로직도 롤백할지, 커밋할지 애매한 문제도 남아있다.
- 위 문제를 해소하는 방법은 두 가지가 있다.
    - 이벤트를 비동기로 처리
    - 이벤트와 트랜잭션을 연계

## 10.5 비동기 이벤트 처리

- 구현할 것들 중 ‘A하면 이어서 B 하라’라는 요구사항은 실제로 ‘A 하면 최대로 언제까지 B 하라’인 경우가 많다.
  - ex) 회원 가입 후 가입 축하 이메일을 바로 받을 필요는 없고 주문을 취소한 뒤 결제 취소가 수십 초 내에 이루어질 수도 있다.
- **이벤트를 비동기로 처리하는 방식으로 구현할 수 있는데,** 네 가지 방식이 있다.
  - 로컬 핸들러로 비동기로 실행하기
  - 메시지 큐를 사용하기
  - 이벤트 저장소와 이벤트 포워더 사용하기
  - 이벤트 저장소와 이벤트 제공 API 사용하기

### 10.5.1 로컬 핸들러 비동기 실행

- 스프링이 제공하는 `@Async` 애너테이션을 사용하면 이벤트 핸들러가 별도 스레드로 실행된다.
  - `@EnableAsync` 애너테이션으로 비동기 기능을 활성화 후 이벤트 핸들러 메서드에 `@Async` 애너테이션을 붙인다.

    ```java
    @SpringbootApplication
    @EnableAsync
    public class ShopApplication {
    		
    	public static void main(String[] args) {
    		SpringApplication.run(ShopApplication.class, args);
    	}
    }
    ```

    ```java
    @Async
    @EventListener(OrderCanceldEvent.class)
    public void handle(OrderCanceledEvent event) {
    	// ...
    }
    ```


### 10.5.2 메시징 시스템을 이용한 비동기 구현

- 카프카나 래빗MQ와 같은 **메시징 시스템을 이용해 비동기 이벤트를 처리할 수 있다.**
  1. 이벤트가 발생
  2. 이벤트 디스패처가 이벤트를 메시지 큐에 전송
  3. 메시지 큐는 이벤트를 메시지 리스너에게 전달
  4. 메시지 리스너가 이벤트 핸들러를 이용해 이벤트 처리
- 필요하다면 이벤트를 발생시키는 도메인 기능과 큐를 이벤트에 저장하는 절차를 한 트랜잭션으로 묶어야 한다.
  - 이를 위해선 글로벌 트랜잭션이 필요
  - 이벤트를 메시지 큐에 안전하게 전달할 수 있는 장점이 있다.
  - 글로벌 트랜잭션으로 인해 전체 성능이 떨어지는 단점이 있다.
- 메시지 큐를 사용하면 보통 이벤트 발생 주체와 이벤트 핸들러가 별도 프로세스(JVM)에서 동작한다.
  - 동일 JVM에서 이벤트를 주고 받을 수 있지만 시스템을 더 복잡하게 만들 뿐이다.
- 래빗MQ는 글로벌 트랜잭션과 함께 클러스터와 고가용성을 지원하기에 안정적인 메시지 전달의 장점이 있다.
- 카프카는 글로벌 트랜잭션을 지원하진 않지만 다른 메시징 시스템에 비해 높은 성능을 보여준다.

### 10.5.3 이벤트 저장소를 이용한 비동기 처리

- **이벤트를** **DB에 저장한 뒤 별도 프로그램을 이용해 이벤트 핸들러에 전달**하는 방법도 있다.
  1. 이벤트가 발생하면 핸들러는 스토리지에 이벤트를 저장
  2. 별도 스레드에서 동작하는 프로그램이 주기적으로 저장소에서 이벤트를 가져와 핸들러를 실행
- 도메인 상태와 이벤트 저장소로 동일한 DB를 사용한다.
  - 도메인 변화와 이벤트 저장이 로컬 트랜잭션으로 처리된다.
- 핸들러가 이벤트 처리에 실패해도 다시 이벤트 저장소에서 이벤트를 읽으면 된다.
- 이벤트 저장소에서 이벤트를 읽는 별도 프로그램으로 **포워더**와 **외부 API**가 있다.
  - 포워더 방식은 포워드를 이용해 이벤트를 외부에 전달한다.
  - 포워더 방식은 이벤트를 어디까지 처리했는지 추적하는 역할이 포워더 자신에게 있다.
  - API 방식은 외부 핸들러가 API를 통해 이벤트 목록을 가져간다.
  - API 방식은 외부 핸들러가 자신이 어디까지 이벤트를 처리했는지 기억해야 한다.

## 10.6 이벤트 적용 시 추가 고려 사항

1. **이벤트에 이벤트 소스를 추가할 것인지** 고려해야 한다. 
   - ‘Order가 발생시킨 이벤트만 조회하기’처럼 특정 주체에 대한 이벤트만 읽고 싶으면 이벤트에 발생 주체 정보를 추가해야 한다.
2. **포워더에서 전송 실패를 얼마나 허용할 것인지** 고려해야 한다.
   - 포워더는 이벤트 전송에 실패하면 실패한 이벤트부터 다시 읽어 전송을 시도한다.
   - 특정 이벤트에서 계속 전송이 실패하면 나머지 이벤트는 계속 전송할 수 없게 된다.
   - 실패한 이벤트의 재전송 횟수 제한을 두면 해결할 수 있다.
   - 처리에 실패한 이벤트를 별도 저장소에 저장하면 실패 이유 분석이나 후처리에 도움이 된다.
3. **이벤트 손실**에 대해서 고려해야 한다.
   - 이벤트 저장소를 사용하는 방식은 이벤트 발생과 저장을 한 트랜잭션으로 처리하기에 손실을 막을 수 있다.
   - 반면 로컬 핸들러를 이용한 비동기 처리에선 이벤트 처리에 실패하면 이벤트를 유실하게 된다.
4. **이벤트 순서**에 대해서 고려해야 한다.
   - 이벤트 발생 순서대로 외부 시스템에 전달해야 한다면 이벤트 저장소를 사용하는 것이 좋다.
   - 반면 메시징 시스템은 기술에 따라 발생 순서와 전달 순서가 다를 수도 있다.
5. **이벤트 재처리**에 대해 고려해야 한다.
   - 동일한 이벤트를 다시 처리해야 할 때 어떻게 할지 결정해야 한다.
   - 마지막 처리한 이벤트 순번을 기억해 두었다가 이미 처리한 순번 이벤트가 도착하면 무시하는 방법이 가장 쉽다.
   - 이벤트 핸들러가 멱등성을 가진다면 이벤트 중복 발생이나 중복 처리에 대한 부담이 없다.

### 10.6.1 이벤트 처리와 DB 트랜잭션 고려

이벤트를 사용할 때 동기로 하든 비동기로 하든 이벤트 처리 실패와 트랜잭션 실패를 함께 고려해야 한다.

- 이벤트 발생과 처리를 모두 동기로 처리할 경우
  - 이벤트 핸들러가 외부 시스템을 호출하여 작업을 정상 완료 후 DB에 완료 처리 업데이트를 하는데 실패할 수가 있다.
  - 이 경우 작업은 완료 됐지만 DB에는 완료되지 않은 상태로 남게 된다.
- 이벤트 처리를 비동기로 처리할 경우
  - 이벤트 발생 주체에서 비동기 핸들러를 호출 후 DB에 완료 업데이트를 커밋하지만 외부 시스템이 실패할 수가 있다.
  - 이 경우 작업은 완료되지 않았지만 DB에는 완료된 상태로 남게 된다.
- **트랜잭션이 성공할 때만 이벤트 핸들러를 실행하는 방법으로 해결**할 수 있다.
  - `@TransactionalEventListener`를 사용하면 트랜잭션 커밋 여부에 따라 핸들러 실행 여부를 결정할 수 있다.
  - 트랜잭션 성공 시에만 핸들러를 실행하게 되면 트랜잭션 실패는 고려하지 않고 이벤트 처리 실패만 고려하면 된다.
    
