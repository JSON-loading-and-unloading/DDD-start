<h1>이벤트</h1>

<h2>시스템 간 강결합 문제</h2>

```
public class Order {

    public void cancel(RefundService refundService) {

        // 주문 로직
        verifyNotYetShopped();
        this.state = OrderState.CANCELED;

        // 결제 로직
        this.refundStatus = State.REFUND_STARTED;
        try {
            refundSvc.refund(getPaymentId());
            this.refundStatus = State.REFUND_COMPLETED;
        } catch(Exception e) {
            ...
        }

    }
}

```

환불 서비스를 구현한다고 할 때, 보통 결제 시스템은 외부에 존재하므로 RefundService는 외부에 있는 결제 시스템이 제공하는 환불 서비스를 호출한다.</br></br>

문제 발생</br>
1. 외부 서비스가 정상이 아닐 경우 트랜잭션 처리를 어떻게 해야 할지 애매하다. 환불 기능을 실행하는 과정에서 익셉션이 발생하면 트랜잭션을 롤백 해야 할까? 아니면 커밋해야 할까?</br>
2. 성능 문제, 외부 시스템의 응답 시간이 길어지면 그 만든 대기 시간도 길어진다.</br></br>

사실 Order는 주문을 표현하는 도메인 객체인데 결제 도메인의 환별 관련 로직이 뒤섞이게 된다.</br>
이것은 환불 기능이 바뀌면 Order도 영향을 받게 된다는 것을 의미한다.</br></br>

추가적으로 주문을 취소한 뒤에 환불뿐만 아니라 취소했다는 내용을 통지해야 한다면 트랜잭션 처리가 더 복잡해진다.</br></br>

이는 주문 바운디드 컨텍스트와 결제 바운디드 컨텍스트간의 강결합 때문이다.</br>
이를 해결하기 위해 이벤트를 사용한다.</br></br>

<h2>이벤트 개요</h2>

이벤트라는 용어는 '과거에 벌어진 어떤 것'을 의미한다.</br>

<h3>이벤트 관련 구성요소</h3>

1. 이벤트 생성 주체는 엔티티, 밸류, 도메인 서비스와 같은 도메인 객체이다. 이벤트 생성 주체는 이벤트를 생성해서 디스패터에 이벤트를 전달한다.</br>
2. 이벤트 핸들러는 이벤트 생성 주체가 발생한 이벤트에 반응한다.</br>
3. 이벤트 디스패처는 이벤트 생성 주체와 이벤트 핸들러를 연결해 주는 것이다.</br>


<h3>이벤트의 구성</h3>

- 이벤트 종류 : 클래스 이름으로 이벤트 종류를 표현</br>
- 이벤트 발생 시간</br>
- 추가 데이터: 주문번호, 신규 배송지 정보등 이벤트와 관련된 정보</br>

```
public class ShippingInfoChangedEvent {

    private String orderNumber;
    private long timestamp;
    private ShippingInfo newShippingInfo;

    // 생성자, getter
}
```
클래스 이름을 보면 과거 시제를 사용한다.</br>


```
public class Order {

  public void changeShippingInfo(ShippingInfo newShippingInfo) {
      verifyNotYetShipped();
      setShippingInfo(newShippingInfo);
      Events.raise(new ShippingInfoChangedEvent(number, newShippingInfo));
  }
  ...
}
```
이 코드에서는 Events.raise()는 디스패치를 통해 이벤트를 전파하는 기능을 제공한다.</br>

```
public class ShippingInfoChangedHandler implement EventHandler<ShppingInfoChangedEvent> {

  @Override
  public void handle(ShppingInfoChangedEvent evt) {
    shippingInfoSynchronizer.sync(
      evt.getOrderNumber(),
      evt,getNewShippingInfo();
    )
  }
}

```
핸들러는 디스패치로부터 이벤트를 전달받아 수행한다.</br>
이때 이벤트를 발생시키기에 필요한 데이터를 가져오기도 한다.</br>

<h3>이벤트 용도</h3>

![99668194-bee19e80-2ab0-11eb-9efb-658288beeccb](https://github.com/JSON-loading-and-unloading/DDD-start/assets/106163272/cdbd298b-cf73-49a7-860a-542dd34eebd2)

1. 트리거 -> 도메인의 상태가 바뀔 때 다른 후처리가 필요하면 실행하기 위한 트리거로 이벤트를 사용할 수 있다.
2. 데이터 동기화 -> 바뀐 데이터를 다른 시스템에 전달하여 데이터를 동기화 시킨다.

<h3>이벤트 장점</h3>

![99668375-fe0fef80-2ab0-11eb-8e78-7bcbd78e84c8](https://github.com/JSON-loading-and-unloading/DDD-start/assets/106163272/5a5994fa-d84c-439a-86cd-e5ac6bba2617)

그림과 같이 서로 다른 도메인에 로직이 섞이는 것을 방지할 수 있다.</br>

<h2>이벤트, 핸들러, 디스패치 구현</h2>

이벤트 클래스 : 이벤트를 표현</br>
디스패터 : 스프링 제공 ApplicationPublisher를 이용</br>
Events: 이벤트를 발행한다. 이벤트 발행을 위해 ApplicationPublisher를 사용</br>
이벤트 핸들러 : 이벤트를 수신해서 처리. 스프링이 제공하는 기능을 사용</br>

<h3>이벤트 클래스</h3>
이벤트 자체를 위한 상위 타입은 존재하지 않는다.</br>
이벤트 클래스의 이름을 결정할 때는 과거 시제를 사용해야 한다는 점만 유의한다.</br>

```
public class OrderCanceledEvent extends Event {
		// 이벤트는 핸들러에서 이벤트를 처리하는 데 필요한 데이터를 포함한다.
    private String orderNumber;
    
    public OrderCanceledEvent(String number) {
        super();
        this.orderNumber = number;
    }

    public String getOrderNumber() { return orderNumber; }
}

```


```
public abstract class Event {
    
    private long timestamp;

    public Event() {
        this.timestamp = System.currentTimeMillis();
    }

    public long getTimestamp() {
        return timestamp;
    }   
}
```

모든 이벤트가 공통으로 갖는 프로퍼티가 존재한다면 관련 상위 클래스를 만들 수도 있다.</br>

<h3>Events 클래스와 ApplicationPublisher</h3>

```
public class Events {
    
    // EventHandler 목록을 보관하는 ThreadLocal 변수를 생성한다.
    private static ThreadLocal<List<EventHandler<?>>> handlers = 
        new ThreadLocal<>();

    // 이벤트를 처리 중인지 여부를 판단하는 ThreadLocal 변수를 생성한다.
    private static ThreadLocal<Boolean> publishing = 
        new ThreadLocal<Boolean>() {
            @Override
            protected Boolean initialValue() {
                return Boolean.FALSE;
            }   
        };

    // 파라미터로 전달받은 이벤트를 처리한다.
    public static void raise(Object event) {
        // 이벤트를 처리 중이면 진행하지 않는다.
        if (publishing.get()) return;
    }}
```

raise()는 ApplicationPublisher가 제공하는 publishEvent() 메서드를 이용해서 이벤트를 발생시킨다.</br>


```
public class Order {
    
    public void cancel() {
        verifyNotYetShipped();
        this.state = OrderState.CANCELED;
        
        Events.raise(new OrderCanceledEvent(number.getNumber()));
    }
}

```


```
@EventListiner(OrderCanceldEvent.class)
~


```

publicEvent() 메서드를 실행할 때, OrderCanceledEvent 타입 객체를 전달하면 OrderCanceledEvent.class 값을 갖는 @EventListiner 애너테이션을 붙인</br>
메서드를 찾아 실행한다.</br>

![ddd_start_10_02](https://github.com/JSON-loading-and-unloading/DDD-start/assets/106163272/73523efa-48d7-487a-be02-f69383372a3d)

위 시퀀스 다이어그램을 통해 이해할 수 있다.</br>
