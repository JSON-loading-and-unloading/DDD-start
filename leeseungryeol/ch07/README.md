<h1>도메인 서비스</h1>
<h2>여러 애그리거트 필요한 기능</h2>

도메인 영역의 코드를 작성하다 보면, 한 애그리거트 기능을 구현할 수 없을 때가 있다. </br>
책에 있는 결제 금액 계산에 대해서, </br>
쿠폰을 있는데 두 개이상의 쿠폰을 적용할 수 있다면 단일 할인 쿠폰 애그리거트로는 총 결제 금액을 계산할 수 없다. </br> </br>

이것을 무시하고 한 애그리거트에 넣기 애매한 도메인 기능을 억지로 특정 애그리트에 구현하면 안된다. </br>
억지로 구현하게 되면 애그리거트는 자신의 책임 범위를 넘어서는 기능을 구현하기 때문에 코드가 길어지고 외부에 대한 의존이 높아지게 되며 코드를 복잡하게 만들어 수정을 어렵게 만드는 요인이 된다. </br>
-> 이를 해결하기 위한 것이 바로 도메인 기능을 별로 서비스로 구현하는 것이다. </br>

<h2>도메인 서비스</h2>
도메인 서비스는 도메인 영역에 위치한 도메인 로직을 표현할 때 사용한다. </br>
 - 계산 로직 : 여러 애그리거트가 필요한 계산 로직이, 한 애그리거트에 넣기에는 다소 복잡한 계산 로직 </br>
 - 외부 시스템 연동이 필요한 도메인 로직 : 구현하기 위해 타 시스템을 사용해야 하는 도메인 로직 </br> </br>

 <h3>계산 로직과 도메인 서비스</h3>

 - 응용 영역의 서비스가 응용 로직을 다룬다면 도메인 서비스는 도메인 로직을 다룬다. </br>

```
public class DiscountCalculationService {

  public Money calculateDiscountAmounts(
      List<OrderLIne> orderLines,
      List<Coupon> coupons,
      MemberGrade grade) {
    Money couponDiscount = coupons.stream()
        .map(coupon -> calculateDiscount(coupon))
        .reduce(Money(0), (v1, v2) -> v1.add(v2));

    Money membershipDiscount = calculateDiscount(orderer.getMember().getGrade());

    return couponDiscount.add(membershipDiscount);
  }
}

```

위처럼 할인 계산 서비스를 사용하는 주체는 애그리거트가 될 수도 있고 응용 서비스가 될 수도 있다. </br>
다음과 같이 애그리거트의 결제 금액 계산 기능에 전달하면 사용 주체는 애그리거트가 된다. </br>

```
public class Order {
 
    public void calculateAmounts(DiscountCalculationService disCalSvc, MemberGrade grade) {
        Money totalAmounts = getTotalAmounts();
        Money discountAmounts = disCalSvc.calculateDiscountAmounts(this.orderLines, this.coupons, grade);
        this.paymentAmounts = totalAmounts.minus(discountAmounts);
    }
}

```
애그리거트 객체에 도메인 서비스를 전달하는 것은 응용 서비스 책임이다. </br> </br>

📌Note 도메인 서비스 객체를 애그리거트에 주입하지 않기 </br>
-> 애그리거트의 메서드를 실행할 때 도메인 서비스 객체를 파라미터로 전달한다는 것은 애그리거트가 도메인 서비스에  의존한다는 것을 의미한다. </br>
-> 의존 주입 기능을 사용해서 도메인 서비스를 애그리거트에 주입하는것은 좋은 방법이 아니다. </br> </br>

```
public class Order{

  @Autowired
  private DiscountCalculationService discountCalculationSevice;

}

```
도메인 객체는 필드로 구성된 데이터와 메서드를 이용해서 개념적으로 하나인 모델을 표현한다. </br>
그런데 discountCalculationService필드는 데이터 자체와는 관련이 없다. Order에서 모든 기능을 사용하는 것도 아니다. 일부 기능을 위해 굳이 도메인 서비스 객체를 애그리거트에 의존 주입할 이유는 없다.

 </br>
반대로 도메인 서비스의 기능을 실행할 때 애그리거트를 전달하기도 한다. </br>

```
public class TransferService {

  public void transfer(Account fromAcc, Account toAcc, Money amounts) {
    fromAcc.withdraw(amounts);
    toAcc.credit(amounts);
  }

}

```

응용 서비스는 두 Account 애그리거트를 구한 뒤에 해당 도메인 영역의  TransferService를 이용해서 계좌 이체 도메인 기능을 실행할 것이다. </br>

도메인 서비스는 도메인 로직을 수행하지 응용 로직을 수행하진 않는다. 트랜잭션 처리와 같은 로직은 응용 로직이므로 도메인 서비스가 아닌 응용 서비스에서 처리한다. </br>

📌특정 기능이 응용 서비스인지 도메인 서비스인지 감을 잡기 어려울 때는 해당 로직이 애그리거트의 상태를 변경하거나 애그리거트의 상태 값을 계산하는지 검사해 보면 된다. </br>

<h3>외부 시스템 연동과 도메인 서비스</h3>

외부 시스템이나 타 도메인과의 연동 기능도 도메인 서비스가 될 수 있다. </br>

시스템 간 연동은 HTTP API 호출로 이루어질 수 있지만 설문 조사 도메인 입장에서는 사용자가 설문 조사 생성 권한을 가졌는지 확인하는 도메인 로직으로 볼 수 있다. </br>
여기서 중요한 것은 도메인 로직 관점에서 인터페이스를 작성한다. </br>

```
public interface SurveyPermissionChecker {
  boolean hasUserCreationPermission(String userId);
}

```

응용 서비스는 이 도메인 서비스를 이용해서 생성 권한을 검사한다. </br>

```
public class CreateSurveyService {

  prvate SurveyPermissionCheckr permissionChecker;
}

```

surveyPermissionChecker 인터페이스를 구현한 클래스는 인프라스트럭처 영역에 위치해 연동을 포함한 권한 검사 기능을 구현한다. </br>


<h3>도메인 서비스의 패키지 위치</h3>

![img1 daumcdn](https://github.com/JSON-loading-and-unloading/DDD-start/assets/106163272/f2217780-add2-43c1-97b7-a9b575174167)

도메인 서비스는 도메인 로직을 표현하므로 도메인 서비스의 위치는 다른 도메인 구성요소와 동일한 패키지에 위치한다. </br>

<h3>도메인 서비스의 인터페이스와 클래스</h3>

도메인 서비스의 로직이 고정되어 있지 않는 경우 도메인 서비스 자체를 인터페이스로 구현하고 이를 구현한 클래스를 둘 수도 있다. </br>
<img width="258" alt="99547954-7b792880-29fb-11eb-9cec-e8974038d28b" src="https://github.com/JSON-loading-and-unloading/DDD-start/assets/106163272/94e1c7a3-227b-4884-8d18-ab3ca0597620">

위와 같이 도메인 서비스의 구현이 특정 구현 기술에 의존하거나 외부 시스템의 api를 실행한다면 도메인 영역의 도메인 서비스는 인터페이스로 추상화해야 한다. </br>
이를 통해 도메인 영역이 특정 구현에 종속되는 것을 방지할 수도 있고 도메인 영역에 대한 테스트가 쉬워진다. </br>
