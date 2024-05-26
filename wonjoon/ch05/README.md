## CQRS

CQRS란 명령 모델과 조회 모델을 분리하는 패턴이다.

엔티티, 애그리거트, 레포지토리 등 앞에서 살펴봤던 모델은 주문 취소, 배송지 변경과 같이 상태를 변경할 때 주로 사용되는 것으로, 도메인 모델은 명령 모델로 주로 사용된다.

반면에 이 장에서 설명할 정렬, 페이징, 검색 조건 지정과 같은 기능은 단순 조회 기능에 사용되는 것으로 이번 장의 방법은 조회 모델을 구현할 때 주로 사용된다.

## 스펙

검색 조건을 다양하게 조합해야 할 때 Specification이라는 스펙을 사용할 수 있다.

```java
public interface Specification<T> {
	public boolean isSatisfiedBy(T agg);
}
```

이러한 스펙을 레포지토리에 사용하면 agg는 애그리거트가 되고, DAO에 사용하면 agg는 검색 결과로 리턴할 수 있다.

```java
public class OrderSpec implements Specification<Order> {
	private String orderId;
	
	public OrderSpec(String orderId) {
		this.orderId = orderId;
	}

	public boolean isSatisfiedBy(Order agg) {
		return agg.getOrderId().getMemberId().getId().equals(OrderId);
	}
}
```

이런식으로 원하는 형식으로 만들고

```java
public List<Order> findAll(Specification<Order> spec) {
	return findAll().stream()
		.filter(order -> spec.isStatisfiedBy(Order))
		.toList();
}
```

이런 식으로 사용할 수 있다.

하지만 실제 스펙은 이렇게 구현하지 않는다. 모든 애그리거트 객체를 메모리에 보관하기도 어렵고 설사 메모리에 다 보관할 수 있다 하더라도, 조회 성능에 심각하게 문제가 밠애하기 때문이다.

실제 스펙은 사용 기술에 맞춰 구현하게 되는데, 이 장에서는 스프링 데이터 JPA를 사용할 것이다.