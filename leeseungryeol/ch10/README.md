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

<h2>동기 이벤트 처리 문제</h2>

환불 처리 로직일 경우 외부 서비스의 영향을 받는다.</br>
그러나 외부 환불 서비스가 느려진다면 이는 내 시스템의 성능 저하로 연결된다.</br></br>

무조건 트랜잭션을 롤백하는 것이 아니라 구매 취소 자체는 처리하고 환불만 재처리하거나 수동을 처리할 수도 있다.</br></br>

외부 시스템과의 연동을 동기로 처리할 때 발생하는 성능과 트랜잭션 범위 문제를 해소하는 방법은 이벤트를 비동기로 처리하거나 이벤트와 트랜잭션을 연계하는 것이다.</br>


<h2>비동기 이벤트</h2>

'A하면 이어서 B하라' 중에 'A하면 최대 언제까지 B하라' 로 바꿀 수 있는 요구사항은 이벤트를 비동기로 처리하는 방식으로 구현할 수 있다.</br>

방법
1. 로컬 핸들러를 비동기로 실행하기
2. 메시지 큐를 사용하기
3. 이벤트 저장소와 이벤트 포워더 사용하기
4. 이벤트 저장소와 이벤트 제공 API 사용하기


<h3>로컬 핸들러 비동기 실행</h3>

이벤트 핸들러를 별로 스레드로 실행</br>

- @EnableAsync 애너테이션을 사용해서 비동기 기능 활성화
- 이벤트 핸들러 메서드에 @Async 애너테이션을 붙인다.


<h3>메시징 시스템을 이용한 비동기 구현</h3>

카프카나 래빗 MQ를 사용해서 메시징 큐 사용

![image](https://github.com/JSON-loading-and-unloading/DDD-start/assets/106163272/9d3b71a4-1368-4332-b0d1-ca2e2cbb65c6)

필요할 경우 도메인 기능과 메시지 큐에 이벤트를 저장하는 절차를 한 트랜잭션으로 묶어야한다.</br>
이때 글로벌 트랜잭션이 필요한다.</br>
글로벌 트랜잭션을 사용할 경우</br>
안전하게 이벤트를 메시지 큐에 전달할 수 있는 장점이 있지만,</br>
글로벌 트랜잭션으로 인해 전체 성능이 떨어지는 단점도 있다.</br></br>


<h3>이벤트 저장소를 이용한 비동기 처리</h3>

![99919198-58ce7300-2d5f-11eb-897a-d5f5de43a851](https://github.com/JSON-loading-and-unloading/DDD-start/assets/106163272/06700c27-9616-4d6f-957e-3e78d7968dbd)

이 방법은 이벤트를 일단 DB에 저장한 뒤에 별도 프로그램을 이용해서 이벤트 핸들러에 전달하는 것이다.</br></br>

위 포워더는 주기적으로 이벤트 저장소에서 이벤트를 가져와 이벤트 핸드러를 실행한다.</br>
포워더는 별도 스레드를 이용하기 때문에 이벤트 발행과 처리가 비동기로 처리된다.</br></br>

DB는 도메인 상태와 동일하게 사용한다. 즉, 도메인의 상태변화와 이벤트 저장이 로컬 트랜잭션으로 처리된다.</br>

![99919237-8a473e80-2d5f-11eb-859a-df1b033d1058](https://github.com/JSON-loading-and-unloading/DDD-start/assets/106163272/ed101b8d-e7af-4c76-807d-7c7ef12ad61f)

다른 방법으로는 이벤트를 외부에 제공하는 API를 사용하는 것이다.</br></br>

이 둘의 차이점은 이벤트를 전달하는 방식에 있다.</br>
포워더 방식이 포워더를 이용해서 이벤트를 외부에 전달한다면, API 방식은 외부 핸들러가 API 서버를 통해 이벤트 목록을 가져간다.</br>

<h2>이벤트 적용 시 추가 고려사항</h2>

1. 이벤트 소스를 EventEntry에 추가할지 여부이다.</br>
   -> 이벤트 발생 주체에 대한 정보를 갖지 않으면, 특정 주체가 발생시킨 이벤트만 조회하는 기능을 구현할 수 없다. 이 기능을 구현하려면 이벤트에 발생 주체 정보를 추가해야한다.</br></br>

2. 포워더에서 전송 실패를 얼마나 허용할 것이냐에 대한 것이다.</br>
  -> 이벤트가 계속 실패하면 재시도가 무한으로 반복되므로 최대 몇 번까지 재시도를 할 지 정해줘야한다.</br></br>

3. 이벤트에 대한 것이다.</br>
   -> 이벤트 저장소를 사용하는 방식은 이벤트 발생과 이벤트 저장을 한 트랜잭션으로 처리하기 때문에 트랜잭션에 성공하면 이벤트가 저장소에 보관된다는 것을 보장할 수 있다. 반면 로컬 핸들러를 이용해서 이벤트를 비동기로 처리할 경우 이벤트 처리에 실패하면 이벤트를 유실하게 된다.</br></br>

4. 이벤트 순서에 대한 것이다.</br>
   -> 이벤트 발생 순서대로 외부 시스템에 전달해야 할 경우, 이벤트 저장소를 사용하는 것이 좋다. 반면 메시징 시스템은 사용 기술에 따라 이벤트 발생 순서와 메시지 전달 순서가 다를 수도 있다.</br></br>

5. 이벤트 재처리에 대한 것이다.</br>
   -> 동일한 이벤트를 다시 처리해야 할 때 이벤트를 어떻게 할지 결정해야 한다.</br>
   1. 마지막으로 처리한 이벤트의 순번을 기억해 두었다가 이미 처리한 순번의 이벤트가 도착하면 해당 이벤트를 처리하지 않고 무시한다.</br>
   2. 이벤트 멱등으로 처리한다.</br>


<h3>이벤트 처리와 DB 트랜잭션 고려</h3>

이벤트를 처리할 때는 DB 트랜잭션을 함께 고려해야 한다.</br></br>

이벤트 처리를 동기로 하든 비동기로 하든 이벤트 처리 실패와 트랜잭션 실패를 함께 고려해야 한다.</br>
트랜잭션 실패와 이벤트 처리 실패를 모두 고려하면 복잡해지므로 경우의 수를 줄이면 도움이 된다.</br></br>

경우의 수를 줄이는 방법은 트랜잭션이 성공할 때만 이벤트 핸들러를 실행하는 것이다.</br></br>

스프링은 @TransactionalEventListener 애너테이션을 지원한다.</br>
-> 이 애너테이션은 스프링 트랜잭션 상태에 따라 이벤트 핸들러를 실행할 수 있게 한다.</br></br>

phase 속성 값으로 TransactionPhase.AFTER_COMMIT을 지정한다. 트랜잭션 커밋에 성공한 뒤에 핸들러 메서드를 실행한다.</br>
반대로 중간에 에러가 발생해서 트랜잭션이 롤백 되면 핸들러 메서드를 실행하지 않는다.</br></br>

위 방법을 사용하면서 트랜잭션이 성공할 때만 이벤트가 DB에 저장되므로, 트랜잭션은 실패했는데 이벤트 핸들러가 실행되는 상황은 발생하지 않는다.</br>

