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

환불 서비스를 구현한다고 할 때, 보통 결제 시스템은 외부에 존재하므로 RefundService는 외부에 있는 결제 시스템이 제공하는 환불 서비스를 호출한다.

문제 발생
1. 외부 서비스가 정상이 아닐 경우 트랜잭션 처리를 어떻게 해야 할지 애매하다. 환불 기능을 실행하는 과정에서 익셉션이 발생하면 트랜잭션을 롤백 해야 할까? 아니면 커밋해야 할까?
2. 성능 문제, 외부 시스템의 응답 시간이 길어지면 그 만든 대기 시간도 길어진다.

사실 Order는 주문을 표현하는 도메인 객체인데 결제 도메인의 환별 관련 로직이 뒤섞이게 된다.
이것은 환불 기능이 바뀌면 Order도 영향을 받게 된다는 것을 의미한다.

추가적으로 주문을 취소한 뒤에 환불뿐만 아니라 취소했다는 내용을 통지해야 한다면 트랜잭션 처리가 더 복잡해진다.

이는 주문 바운디드 컨텍스트와 결제 바운디드 컨텍스트간의 강결합 때문이다.
이를 해결하기 위해 이벤트를 사용한다.

<h2>이벤트 개요</h2>

이벤트라는 용어는 '과거에 벌어진 어떤 것'을 의미한다.

<h3>이벤트 관련 구성요소</h3>

1. 이벤트 생성 주체는 엔티티, 밸류, 도메인 서비스와 같은 도메인 객체이다. 이벤트 생성 주체는 이벤트를 생성해서 디스패터에 이벤트를 전달한다.
2. 이벤트 핸들러는 이벤트 생성 주체가 발생한 이벤트에 반응한다.
3. 이벤트 디스패처는 이벤트 생성 주체와 이벤트 핸들러를 연결해 주는 것이다.


<h3>이벤트의 구성</h3>

- 이벤트 종류 : 클래스 이름으로 이벤트 종류를 표현
- 이벤트 발생 시간
- 추가 데이터: 주문번호, 신규 배송지 정보등 이벤트와 관련된 정보

```
public class ShippingInfoChangedEvent {

    private String orderNumber;
    private long timestamp;
    private ShippingInfo newShippingInfo;

    // 생성자, getter
}
```
클래스 이름을 보면 과거 시제를 사용한다.


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
이 코드에서는 Events.raise()는 디스패치를 통해 이벤트를 전파하는 기능을 제공한다.


