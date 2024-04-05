<h1>스프링 데이터 JPA를 이용한 조회 기능</h1>

<h2>시작에 앞서</h2>

CQRS는 명령 모델과 조회모델을 분리하는 패턴이다.</br>
명령 모델은 상태를 변경하는 기능을 구현할 때 사용하고, 조회 모델은 데이터를 조회하는 기능을 구현할 때 사용한다.</br></br>

즉 이 장에서 살펴볼 구현 방법은 조회 모델을 구현할 때를 본다.</br>

<h2>검색을 위한 스펙</h2>
검색 조건을 다양하게 조합해야 할 때 사용할 수 있는 것이 스펙이다.</br>
스펙은 애그리거트가 특정 조건을 충족하는지를 검사할 때 사용하는 인터페이스다.</br></br>

```
public interface Specification<T> {
	public boolean isSatisfiedBy(T agg);
}

```

- 스펙을 리포지터리에 사용하면 agg는 애그리거트 루트가 되고
- 스펙을 DAO에 사용하면 agg는 검색 결과로 리턴할 데이터 객체가 된다.

```
public class OrdererSpec implements Specification<Order> {

  private String ordererId;

  public boolean isSatisfiedBy(Order agg) {
    return agg.getOrdererId().getMemberId().getId().equals(ordererId);
  }

}

```
리포지터리나 DAO는 검색 대상을 걸러내는 용도로 스펙을 사용한다.</br>


<h2>스프링 데이터 JPA를 이용한 스펙 구현</h2>

```
package org.springframework.data.jpa.domain;

public interface Specification<T> extends Serializable {
	@Nullable
	Predicate toPredicate(Root<T> root, 
    				CriteriaQuery<?> query, 
    				CriteriaBuilder cb);
}

public class OrdererIdSpec implements Specification<OrderSummary> {
    private String ordererId;

    public OrdererIdSpec(String ordererId) {
        this.ordererId = ordererId;
    }

    @Override
    public Predicate toPredicate(Root<OrderSummary> root,
					    CriteriaQuery<?> query,
    					CriteriaBuilder cb) {
        return cb.equal(root.get(OrderSummary_.ordererId), ordererId);
    }
}

```

1. OrderIdSpec 클래스는 Specification<OrderSummary>타입을 구현하므로 OrderSummary에 대한 검색 조건을 표현한다.</br>
2. toPredicate()메서드를 구현한 코드에서 ordererId 프로퍼티 값이 생성자로 전달받은 ordererId와 동일한지 비교하는 Predicate을 생성
</br>
※스펙 구현 클래스를 개별적으로 만들지 않고 별도 클래스에 스펙 생성 기능을 모아도 된다.</br></br>
