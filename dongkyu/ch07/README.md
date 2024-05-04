# Chapter7 도메인 서비스

## 7.1 여러 애그리거트가 필요한 기능

- **어떤 기능을 한 애그리거트에서 해결하기 애매한 경우 별도의 도메인 서비스를 만들어 기능을 구현할 수 있다.**
    - 예를 들어 결제 금액을 계산할 땐 ‘상품’, ‘주문’, ‘할인 쿠폰’, ‘회원’의 등급 등의 여러 애그리거트의 정보가 필요하다.
- 한 애그리거트에 넣기 애매한 도메인 기능을 억지로 특정 애그리거트에 구현하는 경우
    - 애그리거트는 자신의 책임 범위를 넘어서는 기능을 구현하게 된다.
    - 외부에 대한 의존이 높아진다.
    - 코드가 복잡해져 수정이 어려워진다.
    - 애그리거트 범위를 넘어서는 개념이 숨어들어 도메인 개념이 명시적으로 드러나지 않게 된다.

## 7.2 도메인 서비스

- **도메인 서비스는 도메인 영역에 위치한 도메인 로직을 표현할 때 사용한다.**
    - 여러 애그리거트가 필요하고 복잡한 계산 로직
    - 외부 시스템 연동이 필요한 도메인 로직

### 7.2.1 계산 로직과 도메인 서비스

- 할인 금액 규칙처럼 한 애그리거트에 넣기 힘든 도메인 개념은 도메인 서비스를 이용해 구현하면 된다.
    - 응용 서비스가 응용 로직을 다룬다면 도메인 서비스는 도메인 로직을 다룬다.
- 도메인 서비스는 상태 없이 로직만 구현한다.
    - 로직을 구현하는 데 필요한 상태는 다른 방법으로 전달 받는다.
- 계산 로직 도메인 서비스

    ```java
    public class DiscountCalculationService {
    
    	public Money calculateDiscountAmounts(
    		List<OrderLine> orderLines,
    		List<Coupon> coupons,
    		MemberGrade grade) {
    			// ...
    	}
    }
    ```

- ‘할인 계산 서비스’를 사용하는 주체는 애그리거트일 수도, 응용 서비스일 수도 있다.
    - 애그리거트가 사용 주체가 되면 애그리거트 객체에 도메인 서비스를 전달하는 것은 응용 서비스의 책임이다.

    ```java
    public class OrderService {
    	private DiscountCalculationService discountCalculationService;
    
    	@Transactional
    	public OrderNo placeOrder(OrderRequest orderReq) {
    		// ...
    		Order order = new Order(orderReq.getInfos(), discountCalculationService);
    		// ...
    	}
    }
    ```

- 애그리거트에 도메인 서비스를 전달할 수도 있지만 도메인 서비스에 애그리거트를 전달하도록 구현할 수도 있다.
- 도메인 서비스는 응용 로직을 수행하진 않기에 트랜잭션 처리 등은 응용 서비스에서 처리해야 한다.

### 7.2.2 외부 시스템 연동과 도메인 서비스

- 외부 시스템이나 타 도메인과의 연동 기능도 도메인 서비스가 될 수 있다.
- 도메인 로직을 인터페이스로 작성하고 구현은 인프라 계층에서 수행한다.
- 설문 조사를 생성할 때 사용자가 생성 권한을 가진 역할인지 확인하기 위해 역할 관리 시스템과 연동하는 예제

    ```java
    public interface SurveyPermissionChecker {
    	boolean hasUserCreationPermission(String userId);
    }
    ```

    ```java
    public class CreateSurveyService {
    	private SurveyPermissionChecker permissionChecker;
    
    	public Long createSurvey(CreateSurveyRequest req) {
    		if (!permissionChecker.hasUserCreationPermission(req.getRequestorId()) {
    			throw new NoPermissionException();
    		}
    	}
    }
    ```


### 7.2.3 도메인 서비스의 패키지 위치

- 도메인 서비스는 도메인 구성요소와 동일한 패키지에 위치한다.
    - 예를 들어 ‘결제 금액 계산 도메인 서비스’는 주문 애그리거트와 같은 패키지에 위치한다.

### 7.2.4 도메인 서비스의 인터페이스와 클래스

- 도메인 서비스 자체를 인터페이스로 구현하고 이를 구현한 클래스를 둘 수도 있다.
    - 도메인 로직을 외부 시스템이나 별도 엔진으로 구현하는 경우
- 이를 통해 도메인 영역이 특정 구현에 종속되는 것을 방지할 수 있고 테스트가 쉬워진다.
