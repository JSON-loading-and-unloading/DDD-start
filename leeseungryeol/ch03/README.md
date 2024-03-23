<h1>애그리거트</h1>

 - 복잡한 도메인을 이해하고 관리하기 쉬운 단위로 만들려면 상위 수준에서 모델을 조망할 수 있는 방법이다.
 - 애그리거트는 관련된 객체를 하나의 군으로 묶어 준다.
 - 수많은 객체를 애그리거트로 묶어서 바라보면 상위 수준에서 도메인 모델 간의 관계를 파악할 수 있다.


![애그리거트](https://github.com/JSON-loading-and-unloading/DDD-start/assets/106163272/2438d648-ea45-4e98-b1c4-a5e39db7006b)

- 애그리거트는 경계를 갖는다.
- 한 애그리거트에 속한 객체는 다른 애그리거트에 속하지 않는다.
- 애그리거트는 독립된 객체 군이며 각 애그리거트는 자기 자신을 관리할 뿐 다른 애그리거트를 관리하지 않는다.


A가 B를 갖는다. 로 설계할 수 있는 요구사항이 있다면 A와 B를 한 애그리거트로 묶어서 생각하기 쉽다.</br>

※상품 상세 페이지에 들어가면 상품 상세 정보와 함께 리뷰 내용을 보여줘야 한다는 요구사항이 있을 때 Product 엔티티와 Review엔티티가 한 애그리거트에 속한다고 생각할 수 있다.</br>
하지만 Product와 Review는 함께 생성되지 않고, 함께 변경되지도 않는다.</br>
 => Review의 변경이 Product에 영향을 주지 않고 반대로 Product의 변경이 Reviwe에 영향을 주지 않기 때문에 이 둘은 한 애그리거트에 속하기 보다는 서로 다른 애그리거트에 속한다.</br>

 <h2>애그리거트 루트</h2>

 애그리거트는 여러 객체로 구성되기 때문에 한 객체만 상태가 정상이면 안된다.</br>
 도메인 규칙을 지키려면 애그리거트에 속한 모든 객체가 정상 상태를 가져야한다.</br></br>

 애그리거트에 속한 모든 객체가 일관된 상태를 유지하려면 애그리거트 전체를 관리할 주체가 필요한데, 이 책임을 지는 것이 바로 애그리거트 루트 엔티티이다.</br>

 ![루트](https://github.com/JSON-loading-and-unloading/DDD-start/assets/106163272/b5ff5fb2-f1eb-48b6-9476-77dfaf310f34)


 주문 애그리거트에서 루트 역할을 하는 엔티티는 Order이다.</br>
 OrderLine, ShippingInfo, Orderer등 주문 애그리거트에 속한 모델은 Order에 직접 또는 간접적으로 속한다.</br></br>

 <h2>도메인 규칙과 일관성</h2>

 애그리거트 루트의 핵심 역할은 애그리거트의 일관성이 깨지지 않도록 하는 것이다.</br>
이를 위해 애그리거트 루트는 애그리거트가 제공해야 할 도메인 기능을 구현한다.</br>
예를 들면 주문 애그리거트는 배송지 변경, 상품 변경과 같은 기능을 제공하고, 애그리거트 루트인 Order가 이 기능을 구현한 메서드를 제공한다.</br></br></br>

```
public class Order {
	// 애그리거트 루트는 도메인 규칙을 구현한 기능을 제공한다.
    public void changeShippingInfo(ShippingInfo newShippingInfo) {
    	verifyNotYetShipped();
        setShippingInfo(newShippingInfo);
    }
    
    private void verifyNotYetShipped() {
    	if(state != OrderState.PAYMENT_WAITING && state != OrderState.PREPARING)
        	throw new IllegalStateException("already shipped");
    }
}

```

애그리거트 외부에서 애그리거트에 속한 객체를 직접 변경하면 안된다.</br>

```
ShippingInfo si = order.getShippingInfo();
si.setAddress(newAddress);

```

주문 상태에 상관없이 배송지 주소를 변경하는데, 이는 업무 규칙을 무시하고 직접 db 테이블의 데이터를 수정하는 것과 같은 결과를 만든다.</br></br>

일관성을 지키기 위해 상태 확인 로직을 응용 서비스에 구현할 수도 있다.</br>
하지만 이렇게 되면 동일한 검사 로직을 여러 응용 서비스에서 중복으로 구현할 가능성이 높아져 유지 보수에 도움이 되지 않는다.</br></br>

불필요한 중복을 피하고 애그리거트 루트를 통해서만 도메인 로직을 구현하게 만들려면 도메인 모델에 대해 다음의 두 가지를 습관적으로 적용해야한다.</br>

 - 단순히 필드를 변경하는 set 메서드를 공개 범위로 만들지 않는다.
 - 밸류 타입은 불변으로 구현한다.

```
public void setName(String name) {
	this.name = name;
}

```
공개 set 메서드는 도메인의 의미나 의도를 표현하지 못하고 도메인 로직을 도메인 객체가 아닌 응용 영역이나 표현 영역으로 분산시킨다.</br>
도메인 모델의 엔티티나 밸류에 공개 set 메서드만 넣지 않아도 일관성이 깨질 가능성이 줄어든다.</br>
공개 set 메서드를 사용하지 않으면 의미가 드러나는 메서드를 사용해서 구현할 가능성이 높아진다.</br></br>

공개 set 메서드를 만들지 않는 것의 연장으로 밸류는 불변 타입으로 구현한다.</br></br>

밸류 객체가 불변이면 밸류 객체의 값을 변경하는 방법은 새로운 밸류 객체를 할당하는 것뿐이다.</br>
즉, 다음과 같이 애그리거트가 제공하는 메서드에 새로운 밸류 객체를 전달해서 값을 변경하는 방법밖에 없다.</br>
=> 밸류 타입의 내부 상태를 변경하려면 애그리거트 루트를 통해서만 가능하다.</br>


<h2>애그리거트 루트의 기능 구현</h2>

애그리거트 루트는 애그리거트 내부의 다른 객체를 조합해서 기능을 완성한다.</br>

```
public class Member{
	private Password password;

	public void changePassword(String currentPassword, String newPassword){
		if (!password.match(currentPassword)){
			throw new PasswordNotMatchException();
		}
		this.password = new Password(newPassword);
	}
}

```
Member 애그리거트 루트는 암호를 변경하기 위해 Password객체에 암호가 일치하는지를 확인할 것이다.</br>


```
public class OrderLines{
	private List<OrderLine> lines;

	public int getTotalAmounts(){...}
	public void changeOrderLines(List<OrderLine> newLines){
		this.lines = newLines;
	}
}

```

애그리거트 루트가 구성요서의 상태만 참조하는 것이 아니다. 기능 실행을 위임하기도 한다.</br>


<h2>트랜잭션 범위</h2>

잠금 대상이 많아지면 그 만큼 동시에 처리할 수 있는 트랜잭션 개수가 줄어든다는 것을 의미하고 이것은 전체적인 성능을 떨어뜨린다.</br>
한 트랜잭션에서 한 애그리거트만 수정한다는 것은 애그리거트에서 다른 애그리거트를 변경하지 않는다는 것을 의미한다.</br>
한 애그리거트에서 다른 애그리거트를 수정하면 결과적으로 두 개의 애그리거트를 수정하게 되므로,</br>
애그리거트 내부에서 다른 애그리거트의 상태를 변경하는 기능을 실행하면 안 된다.</br>

=> 부득이하게 한 트랜잭션으로 두 개 이상의 애그리거트를 수정해야 한다면 애그리거트에서 다른 애그리거트를 직접 수정하지 말고 응용 서비스에서 두 애그리거트를</br>
수정하도록 구현한다.</br></br>

```
public class ChangeOrderService{
	//두 개 이상의 애그리거트를 변경해야 하면, 응용 서비스에서 각 애그리거트의 상태를 변경한다.
	@Transactional
	public void changeShippingInfo(OrderId id, ShippingInfo newShippingInfo, boolean useNewSHippingAddrAsMemberAddr){
		Order order = orderRepository.findbyId(id);
		if (order == null) throw new OrderNotFoundException();
		order.shipTo(newShippingInfo); //1
		if (useNewShippingAddrAsMemberAddr) {
			order.getOrderer().getCustomer().changeAddress(newShippingInfo.getAddress());
		} //2
	}
}

```

한 트랜잭션에서 한 개의 애그리거트를 변경하는 것을 권장하지만, 다음 경우에는 한 트랜잭션에서 두 개 이상의 애그리거트를 변경하는 것을 고려할 수 있다.

</br>
