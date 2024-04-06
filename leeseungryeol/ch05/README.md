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

- 스펙을 리포지터리에 사용하면 agg는 애그리거트 루트가 되고</br>
- 스펙을 DAO에 사용하면 agg는 검색 결과로 리턴할 데이터 객체가 된다.</br>

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


<h2>리포지터리/DAO에서 스펙 사용하기</h2>

스펙을 충족하는 엔티티를 검색하고 싶다면 findAll()메서드를 사용한다.</br>

```
public interface OrderSummaryDao extends Repository<OrderSummary, String> {
    List<OrderSummary> findAll(Specification<OrderSummary> spec);
}

// 스펙 객체 생성하고
Specification<OrderSummary> spec = new OrdererIdSpec("user1");
// findAll() 메서드를 이용해서 검색
List<OrderSummary> results = orderSummaryDao.findAll(spec);

```

<h2>스펙 조합</h2>

스프링 데이터 JPA가 제공하는 스펙 인터페이스는 스펙을 조합할 수 있는 두 메서드를 제공한다.</br>

```
public interface Specification<T> extends Serializable {

	default Specification<T> and(@Nullable Specification<T> other) {...}
	default Specification<T> or(@Nullable Specification<T> other) {...}

	@Nullable
	Predicate toPredicate(Root<T> root, 
    				CriteriaQuery<?> query, 
    				CriteriaBuilder cb);
	}
}

```

- and() 메서드는 두 스펙을 모두 충족하는 조건을 표현하는 스펙을 생성하고
- or() 메서드는 두 스펙 중 하나 이상 충족하는 조건을 표현하는 스펙을 생성한다.

```
Specification<OrderSummary> spec = OrderSummarySpecs.ordererId("user1")
                .and(OrderSummarySpecs.orderDateBetween(from, to));

```
불필요한 변수 사용을 줄일 수 있다.


```
Specification<OrderSummary> spec = Specification.not(OrderSummarySpecs.ordererId("user1"));

Specification<OrderSummary> spec = Specification.where(createNullableSpec()).and(createOtherSpec());
```

- not()은 정적 메서드로 조건을 반대로 적용할 때 사용한다.
- where() 스펙 인터페이스의 정적 메서드로 null을 전달하면 아무 조건도 생성하지 않는 스펙 객체를 리턴하고 null이 아니면 인자로 받은 스펙 객체를 그대로 리턴한다.


<h2>정렬 지정하기</h2>


1. 메서드 이름에 OrderBy를 사용해서 정렬 기준 지정
2. Sort를 인자로 전달


```
public interface OrderSummaryDao extends Repository<OrderSummary, String> {
    List<OrderSummary> findByOrdererIdOrderByNumberDesc(String ordererId);
```

- ordererId프로퍼티 값을 기준으로 검색 조건 지정
- number 프로퍼티 값 역순으로 정렬

```
public interface OrderSummaryDao extends Repository<OrderSummary, String> {
    List<OrderSummary> findByOrdererId(String ordererId, Sort sort);
}
Sort sort1 = Sort.by("number").ascending();
Sort sort2 = Sort.by("orderDate").ascending(); // 오름차순
Sort sort = Sort1.and(sort2);       // 연속 두 개 사용
List<OrderSummary> results = orderSummaryDao.findByOrdererId("user1",sort);

```

<h2>페이징 처리하기</h2>

JPA는 페이징 처리를 위해 Pageable 타입을 이용한다.</br>
Sort 타입과 마찬가지로 find메서드에 Pageable 타입 파라미터를 사용하면 페이징을 자동으로 처리해 준다.</br>

```
public interface MemberDataDao extends Repository<MemberData, String> {
    
  List<MemberData> findByNameLike(String name, Pageable pageable);

}
PageRequest pageReq = PageRequest.of(1, 10);
List<MemberData> uesr = memberDataDao.findByNameLike("사용자%", pageReq);

```

- PageRequest.of() 메서드의 첫 번째 인자는 페이지 번호를, 두 번째 인자는 한 페이지의 개수를 의미한다.</br>


```
  Sort sort = Sort.by("name").descending();
  PageRequest pageReq = PageRequest.of(1, 10, sort);
  List<MemberData> uesr = memberDataDao.findByNameLike("사용자%", pageReq);

```

PageRequest와 Sort를 사용하면 정렬 순서를 지정할 수 있다.</br>

```
Page<MemberData> page = memberDataDao.findByBlocked(false, pageReq);
List<Todo> content = page.getContent(); // 조회 결과 목록
long totalElements = page.getTotalElements(); // 조건에 해당하는 전체 개수
int totalPages = page.getTotalPages(); // 전체 페이지 번호
int number = page.getNumber(); // 현재 페이지 번호
int numberOfElements = page.getNumberOfElements(); // 조회 결과 개수
int size = page.getSize(); // 페이지 크기

```

- Pageable을 사용하는 메서드의 리턴 타입이 Page일 경우 스프링 데이터 JPA는 목록 조회 쿼리와 함께 COUNT 쿼리도 실행해서 조건에 해당하는 데이터 개수를 구한다.
- Page는 전체 개수, 페이지 개수 등 페이징 처리에 필요한 데이터도 함께 제공한다.



- 프로퍼티를 비교하는 findBy프로퍼티 형식의 메서드는 Pageable 타입을 사용하더라도 리턴 타입이 List면 COUNT 쿼리를 실행하지 않는다.
- 스펙을 사용하는 findAll 메서드에 Pageable타입을 사용하면 리턴 타입이 Page가 아니어도 COUNT 쿼리를 실행한다.

처음부터 N개의 데이터가 필요하다면 Pageable을 사용하지 않고 findFirstN 메서드를 사용할 수도 있다.</br>

```
List<MemberData> findFirst3By~!()

```

First 대신 Top을 사용해도 된다 -> First나 Top 뒤에 숫자가 없으면 한 개 결과만 리턴한다.</br>

```
MemberData findFirst~()

```
