<h1>도메인 모델 시작하기</h1>

<h2>도메인 모델 도출</h2>

요구사항</br>

 - 최소 한 종류 이상의 상품을 주문해야 한다
 - 한 상품을 한 개 이상 주문할 수 있다
 - 총 주문 금액은 각 상품의 구매 가격 합을 모두 더한 금액이다
 - 각 상품의 구매 가격 합은 상품 가격에 구매 개수를 곱한 값이다
 - 주문할 때 배송지 정보를 반드시 지정해야 한다
 - 배송지 정보는 받는 사람 이름, 전화번호, 주소로 구성된다
 - 출고를 하면 배송지를 변경할 수 있다
 - 출고 전에 주문을 취소할 수 있다
 - 고객이 결제를 완료하기 전에는 상품을 준비하지 않는다

기능 측면</br>

```
public class Order {
    public void changeShipped() { ... }
    public void changeShippingInfo(ShippingInfo newShippingInfo) { ... }
    public void cancel() { ... }
    public void completePayment() { ... }
}

```


 - 한 상품을 한 개 이상 주문할 수 있다
 - 각 상품의 구매 가격 합은 상품 가격에 구매 개수를 곱한 값이다

   주문 항목이 어떤 데이터로 구성되는지 알려준다.</br>

```
   public class OrderLine {
    private Product product; // 주문할 상품
    private int price; // 상품의 가격
    private int quantity; // 구매 개수
    private int amounts; // 구매 가격 합

    public OrderLine(Product product, int price, int quantity) {
        this.product = product;
        this.price = price;
        this.quantity = quantity;
        this.amounts = calculateMounts();
    }

    private int calculateMounts() {
        return price * quantity;
    }

    public int getAmounts() { ... }
    
  }

```


- 최소 한 종류 이상의 상품을 주문해야 한다
- 총 주문 금액은 각 상품의 구매 가격 합을 모두 더한 금액이다

Order과 OrderLine의 관계를 알 수 있다.</br>

```
public class Order {

  private List<OrderLine> orderLines;
  private Money totalAmounts;

  public Order(List<OrderLine> orderLines){
    setOrderLines(orderLines);
    }

  private void setOrderLines(List<OrderLine> orderLines){
    ...
     }

  private void calculateTotalAmounts(){
    int sum = orderLines.stream()
        .mapToInt(x -> x.getAmounts())
        .sum();
    this.totalAmounts = new Money(sum);

    }
  }

```

```
public class ShipingInfo {
 private String receiverName;
 private String receiverPhoneNumber;
 private String shipingAddress1;
 private String shipingAddress2;
 private String shipingZipcode;

 ...
  }

```

배송지 정보 주소를 가질 수 있다.</br>

```
public class Order {
 private List<OrderLine> orderLines;
 private int totalAmounts;
 private ShipingInfo shippingInfo;

    public Order(List<OrderLine> orderLines, ShippingInfo shippingInfo) {
        setOrderLines(orderLines);
        setShippingInfo(shippingInfo);
    }

 private void setShippingInfo(ShippingInfo shippingInfo) {
        if(shippingInfo == null)
            throw new IllegalArgumentException("no ShippingInfo");
    }
 }

```

ShippingInfo가 null이면 익셉션이 발생하여, 배송지 구현 필수 도메인 규칙 구현

 - 출고를 하면 배송지를 변경할 수 있다
 - 출고 전에 주문을 취소할 수 있다
 - 고객이 결제를 완료하기 전에는 상품을 준비하지 않는다

위 요구사항을 통해</br>

```
public enum OrderState {
    PAYMENT_WAITING, PREPARING, SHIPPED, DELIVERING, DELIVERY_COMPLETED;
}

```


```
public class Order {
 private OrderState state;

    public void changeShippingInfo(ShippingInfo newShippingInfo) {
        verifyNotYetShipped();
        setShippingInfo(newShippingInfo);
    }

 public void cancel() {
        verifyNotYetShipped();
        this.state = OrderState.CANCELED;

     }
 public void verifyNotYetShipped() {
        if( state != OrderState.PAYMENT_WATING && state != OrderState.PREPARING)
            throw new IllegalStateException("aleady shipped");
     }
 }

```

배송지 변경이나 주문 취소 기능을 출고 전에만 가능하다는 제약 규칙이 있으므로 이 규칙을 적용하기 위해 changeShippedInfo()와 cancal()은 verifyNotYetShipped() 메서드를 먼저 실행한다.
</br>

<h2>엔티티와 밸류</h2>

<h3>엔티티</h3>

엔티티의 가장 큰 특징은 식별자를 가진다는 것이다.</br>
식별자는 엔티티 객체마다 고유해서 각 엔티티는 서로 다른 식별자를 갖는다.</br>

<h3>엔티티의 식별자 생성</h3>

식별자 생성 방법</br>
 - 특정 규칙에 따라 생성
 - UUID나 Nano ID와 같은 고유 식별자 생성기 사용
 - 값을 직접 입력
 - 일련번호 사용(시퀀스나 DB의 자동 증가 칼럼 사용)


<h3>밸류 타입</h3>

```
 public class ShippingInfo{

   private String receiverName;             // 받는 사람
   private String receiverPhoneNumber;      // 받는 사람
   private String shippingAddress1;         // 주소
   private String shippingAddress2;         // 주소
   private String shippingAddress3;         // 주소

   }

```

각각 변수들은 의미하는 것들이 같다.

```
 public class Receiver {
    private String name;
    private String phoneNumber;

    public Receiver(String name, String phoneNumber){

      this.name = name;
      this.phoneNumber = phoneNumber;
  }
}

```

Receiver자체를 받는 사람이라는 클래스 객체로 생성한다.</br>
주소도 마찬가지~ㅋ</br>


```
 public class ShippingInfo {

   private Receiver receiver;
   private Address address;

}

```

이처럼 ShippingInfo 클래스가 변경된다.</br></br>

밸류 객체의 데이터를 변경할 때는 기본 데이터를 변경하기보다는 변경한 데이터를 갖는 새로운 밸류 객체를 생성하는 방식을 선호한다.</br>
예를 들어 앞서 Money 클래스의 add() 메서드를 보면 Money를 새로 생성하고 있다.</br>

```
 public class Money{
   private int value;

   public Money add(Money money){
      return new Money(this.value + money.value);
   }

}

```

Money처럼 데이터 변경 기능을 제공하지 않는 타입을 불변이라고 표현한다.</br>
밸류 타입을 불변으로 구현하는 이유는 안전한 코드를 작성할 수 있다.</br>


```
 Money price = new Money(1000);
 OrderLine line = new OrderLine(product, price, 2);
 price.setValue(2000);

```

이 경우 값이 꼬이게 된다.</br></br>

이를 방지하기 위해서는</br>

```

  public class OrderLine {


     public OrderLine(Product product, Money price, int quantitiy){

        this.product = product;
        this.price = new Money(price.getValue())  // price 파라미터가 변경될 때 발생하는 문제를 방지하기 위해 데이터를 복사한 새로운 객체를 생성
            ...

       }

  }

```

위처럼 표현할 수 있다.</br>
Money가 불변이면 이런 코드를 작성할 필요가 없다. Money의 데이터를 바꿀 수 없기 때문에 파라미터로 전달받은 price를 안전하게 사용할 수 있다.</br>


<h3>도메인 모델에 set메서드 넣지 않기</h3>

```
 public class Order{

    public void setShippingInfo(ShippingInfo newShipping){..}
    public void setOrderState(OrderState state){..}
}

```

앞서 사용했던 chanegeShippingInfo()가 배송지를 새로 변경한다는 의미를 가졌다면 setShippingInfo()는 배송지 값을 설정한다는 의미를 한다.</br></br>

set메서드는 필드값만 변경하고 끝나기 때문에 생태 변경과 관련된 도메인 지식이 코드에서 사라지게 된다.</br>
set메서드의 또 다른 문제는 도메인 객체를 생성할 때 온전하지 않은 상태가 될 수 있다.</br>

```
  Order order = new Order();
  order.setOrderLine(lines);
  order.setShippingInfo(shippingInfo);
  order.setState(OrderState.PREPARING);

```

위 코드는 주문자 설정을 누락하고 있다.</br>
주문자 정보를 담고 있는 필드인 orderer가 null인 상황에서 order.setState()메서드를 호출해서 상품 준비 중 상태로 바꿨다.</br></br>

도메인 객체가 불완전한 상태로 사용되는 것을 막으려면 생성 지점에 필요한 것을 전달해 주어야 한다.</br>

```

 public class Order {

    public Order( Orderer orderer, List<OrderLine> orderLines, ShippingInfo shippingInfo, OrderState state){
       setOrderer(orderer);
       setOrderLines(orderLines);

   }
}

```

이 코드의 set 메서드는 앞서 언급한 set메서드와 차이점이 있는데 바로 접근 범위가 private이라는 점이다.</br>
이 코드에서 set 메서드를 클래스 내부에서 데이터를 변경할 목적으로 사용된다.</br>
private이기 때문에 외부에서 데이터를 변경할 목적으로 set 메서드를 사용할 수 없다.</br></br></br>


※Dto의 set/get 메서드</br></br>

DTO가 도메인 로직을 담는 경우는 없으므로 get/set 메서드를 제공해도 큰 문제는 없다.</br>

