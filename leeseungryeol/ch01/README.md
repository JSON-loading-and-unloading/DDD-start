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
    
    // ...
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
 ...
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
