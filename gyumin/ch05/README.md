# 5장 스프링 데이터 JPA를 이용한 조회 기능

## 5.1 시작에 앞서

* CQRS는 커맨드와 쿼리를 분리하는 패턴입니다.
* 커맨드는 상태를 변경하는 기능을 구현할 때 사용합니다.
* 쿼리는 데이터를 조회할 때 사용합니다.

## 5.2 검색을 위한 스팩

* 다양한 검색 조건을 사용하기 위해 스펙(specification)을 사용할 수 있습니다.
* 스펙은 애그리거트가 특정 조건을 충족하는지를 검사할 때 사용하는 인터페이스 입니다.

## 5.3 스프링 데이터 JPA를 이용한 스펙 구현

* JPA에서도 검색 조건을 표현하기 위해 Specification 인터페이스를 제공합니다.
* 스펙의 toPredicate 메소드는 JPA 크리테리아 API에서 조건을 표현하는 Predicate를 생성합니다.

```Java
public class OrdererIdSpec implements Specification<OrderSummary>{
    private String ordererId;
    
    @Override
    public Predicate toPredicate(Root<OrderSummary> root,CriteriaQuery<?> query, CriteriaBuidler cb){
        return cb.equals(root.get(OrderSummary_.ordererId), ordererId);
    }
}
```

* 스펙 구현을 람다를 통해 구현할 수 있으며, 이를 통해 하나의 클래스 내부에 스펙 생성 기능들을 모아둘 수 있습니다.

## 5.4 리포지토리/DAO에서 스펙 사용하기

```Java
public interface OrderSummaryDao extends Repository<OrderSummary, String> {
    List<OrderSummary> findAll(Specification<OrderSummary> spec);
}
```

## 5.5 스펙 조합

* JPA에서 스펙들을 조합할 수 있는 and, or 메소드를 제공합니다.
* JPA에서 스펙에 대한 not을 표현해주는 not() 메소드를 제공합니다.
* JPA에서 스펙을 조합할 때, 스펙 자체가 null이 될 수 있으며, 이러한 상황으로 인해 NPE가 발생하는걸 줄이고자 where 메소드를 제공합니다.
* where에 null을 전달하면 아무 조건도 생성하지 않습니다.

## 5.6 정렬 지정하기

* 스프링 데이터 JPA에서는 두 가지 방법을 통해 정렬을 설정할 수 있습니다.
  * 메소드 이름에 OrderBy를 사용
  * Sort를 인자로 전달
* 메소드 이름을 통해 정렬을 설정하면 메소드 이름이 너무나도 길어질 수 있습니다. 이를 Sort 타입을 전달하는 방식을 통해 해결할 수 있습니다.

```Java
List<OrderSummary> findByOrdererIdOrderByOrderDateDescNumberAsc(String ordererId);
List<OrderSummary> findByOrdererId(String ordererId, Sort sort);
```

## 5.7 페이징 처리하기

* 데이터 목록을 조회할 때, 일부분만 조회하기 위해 페이징을 사용합니다.
* 스프링 데이터 JPA에서 Pageable을 통해 페이징 처리를 자동으로 할 수 있습니다.
* Pageable을 사용할 때, 메소드의 반환 타입이 page인 경우 목록 조회와 함께, COUNT 쿼리도 실행해서 조건에 해당하는 데이터 개수도 구합니다.

## 5.8 스펙 조합을 위한 스펙 빌더 클래스

* 스펙을 통해 동적 쿼리를 생성할 때, 조건문을 통해 스펙을 구성하다보면 실수하기 좋고 복잡한 구조를 만들기 쉽습니다.
* 스펙 빌더를 통해 앞선 문제를 해결할 수 있습니다.

```Java
Specification<MemberData> spec = SpecBuilder.builder(MemberData.class)
    .ifTrue(searchRequest.isOnlyNotBlocked(),
            () -> MemberDataSpecs.nonBlocked())
    .ifHasText(searchRequest.getName,
            name -> MeberDataSpecs.nameLike(searchRequest.getName())
    .toSpec();
```

* 스펙 빌더는 and(), ifHasText(), ifTrue() 메소드를 제공하며, 필요한 메소드를 추가로 구현하여 사용할 수 있습니다.

## 5.9 동적 인스턴스 생성

* JPA는 쿼리 결과에서 임의의 객체를 동적으로 생성할 수 있는 기능을 제공합니다. 즉, projection을 통해 뷰 모델을 만들며, 이를 통해 특정 컬럼들을 조회할 수 있습니다.

## 5.10 하이버네이트 @SubSelect 사용

* 하이버네이트 JPA 확장 기능으로 @SubSelect를 제공합니다.
* @SubSelect는 쿼리 결과를 @Entity로 매핑할 수 있는 기능을 제공합니다.

```Java
@Entity
@Immutable
@Subselect(
        """
                select o.order_number as number,
                o.version,
                o.orderer_id,
                o.orderer_name,
                o.total_amounts,
                o.receiver_name,
                o.state,
                o.order_date,
                p.product_id,
                p.name as product_name
                from purchase_order o inner join order_line ol
                    on o.order_number = ol.order_number
                    cross join product p
                where
                ol.line_idx = 0
                and ol.product_id = p.product_id"""
)
@Synchronize({"purchase_order", "order_line", "product"})
public class OrderSummary {
    @Id
    private String number;
    private long version;
    @Column(name = "orderer_id")
    private String ordererId;
    @Column(name = "orderer_name")
    private String ordererName;
    @Column(name = "total_amounts")
    private int totalAmounts;
    @Column(name = "receiver_name")
    private String receiverName;
    private String state;
    @Column(name = "order_date")
    private LocalDateTime orderDate;
    @Column(name = "product_id")
    private String productId;
    @Column(name = "product_name")
    private String productName;

    protected OrderSummary() {
    }

// getter 생략
}
```

* @Immutable, @Subselect, @Synchronize는 하이버네이트 전용 애노테이션입니다.
* @SubSelect는 조회 쿼리를 값으로 가집니다.
* @Immutable은 데이터를 변경하지 못하게 해주는, 즉 레코드에 대해 불변성을 제공해주는 애노테이션입니다.
* @Synchronize는 명시한 테이블의 변경이 있다면, 먼저 데이터베이스에 반영을 하고 값을 조회하는 기능을 제공합니다.
* @SubSelect를 사용해도 일반 @Entity와 같기에 JPA에서 제공하는 다양한 기능을 사용할 수 있습니다.