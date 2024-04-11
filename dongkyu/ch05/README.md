# Chapter5 스프링 데이터 JPA를 이용한 조회 기능

## 5.1 시작에 앞서

- CQRS란 명령(Command) 모델과 조회(Query) 모델을 분리하는 패턴이다.
    - 명령 모델은 상태를 변경하는 기능
    - 조회 모델은 데이터를 조회하는 기능
    - 엔티티, 애그리거트, 리포지터리 등의 모델은 명령 모델에 주로 사용된다.
- 조회 모델을 구현할 때 JPA, 마이바티스, JdbcTemplate 등 다양한 DB 연동 기술을 사용할 수 있다.

## 5.3 스프링 데이터 JPA를 이용한 스펙 구현

- 검색 조건이 고정되어 있다면 단순하게 특정 조건으로 조회하는 기능을 만들면 된다.
- 하지만 다양한 검색 조건을 조합해야 한다면 `find` 메서드가 너무 많이 증가하게 될 것이다.
- 검색 조건을 다양하게 조합해야 할 때 스펙(specification)을 사용할 수 있다.

```java
public interface Specification<T> {
  public boolean isSatisfiedBy(T agg);
}
```

- `isSatisfiedBy()`의 `agg`는 검사 대상 객체다.
    - 레포지터리에 사용하면 애그리거트 루트가 되고 DAO에 사용히면 검색결과로 리턴할 데이터 객체가 된다.

## 5.3 스프링 데이터 JPA를 이용한 스펙 구현

- 스프링 데이터 JPA는 `Specification` 인터페이스를 제공한다.

```java
public interface Specification<T> extends Serializable {
  @Nullable
  Predicate toPredicate(Root<T> root, 
                       CriteriaQuery<?> query,
                       CriteriaBuilder cb);
}
```

- `toPredicate` 메서드는 JPA 크리테리아 API에서 조건을 표현하는 `Predicate`를 생성한다.
    - 아래 예제는 `orderId`가 동일한지 비교하는 `Predicate`를 생성한다.

```java
public class OrderIdSpec implements Specification<OrderSummary> {

    private String orderId;
    
    public OrderIdSpec(String orderId) {
      this.orderId = orderId
    }
    
    @Override
    public Predicate toPredicate(Root<OrderSummary> root,
                                CriteriaQuery<?> query,
                                CriteriaBuilder cb) {
        return cb.equal(root.get(OrderSummary.orderId), orderId);
    }                                
}
```

## 5.4 리포지터리/DAO 스펙 사용하기

- 스펙을 충족하는 엔티티를 검색하고 싶다면 findAll 메서드를 사용하면 된다.

```java
public interface OrderSummaryDao extends Repository<OrderSummary, String> {
  List<OrderSummary> findAll(Specification<OrderSummary> spec);
}
```

## 5.5 스펙 조합

- 스펙 인터페이스에는 스펙을 조합할 수 있는 두 메서드를 제공한다.

```java
public interface Specification<T> extends Serializable {

  default Specification<T> and(@Nullable Specification<T> other) { ... }
  default Specification<T> or(@Nullable Specification<T> other) { ... }

  @Nullable
  Predicate toPredicate(Root<T> root, 
                       CriteriaQuery<?> query,
                       CriteriaBuilder cb);
}
```

- `spec1.and(spec2)`와 같이 작성하면 두 스펙 모두 충족하는 표현을 생성한다.
- 조건을 반대로 적용하는 `not()` 메서드도 존재한다.
- `where()` 메서드를 사용하면 `null` 사용 시 아무 조건도 생성하지 않는 스펙을 생성한다.

## 5.6 정렬 지정하기

- 스프링 데이터 JPA를 쓴다면 메서드 이름으로 정렬 조건을 명시할 수 있다.
    - ex) `findBy…OrderByNumberDesc()`
- 정렬 조건이 복잡하다면 `Sort` 객체를 사용할 수 있다.

```java
Sort sort = Sort.by("number").ascending()
List<OrderSummary> results = orderSummaryDao.findByOrderId(String orderId, Sort sort)
```

- `Sort`도 `and`나 `or`로 조건을 연결할 수 있다.
    - `sort1.and(sort2)`

## 5.7 페이징 처리하기

- `Pageable` 인터페이스로 페이징 처리를 할 수 있다.
    - `PageRequest` 클래스로 `Pageable` 타입을 생성할 수 있다.
- `PageRequest`와 `Sort`를 함께 사용할 수도 있다.

```java
Sort sort = Sort.by("name").descending()
PageRequest req = PageRequest.of(1, 10, sort);
List<MemberData> user = memberDataDao.findByNameLike("...%", req)
```

## 5.8 스펙 조합을 위한 스펙 빌더 클래스

- `if`와 각 스펙을 조합해서 여러 스펙을 조합할 수도 있다.

```java
Specification<MemberData> spec = SpecBuilder.builder(MemberData.class)
    .ifTrue(searchRequest.isOnlyNotBlocked(), 
        () -> MemberDataSpecs.nonBlocked())
    .ifHasText(searchRequest.getName(), 
        name -> MemberDataSpecs.nameLike(searchRequest.getName()))
    .toSpec()
List<MemberData> result = memberDataDao.findAll(spec, PageRequest.of(0, 5));
```

- 빌더 클래스는 아래와 같다.
    - 필요한 메서드는 추가해서 사용하면 된다.

```java
public class SpecBuilder {
    public static <T> Builder<T> builder(Class<T> type) {
        return new Builder<T>();
    }

    public static class Builder<T> {
        private List<Specification<T>> specs = new ArrayList<>();

        public Builder<T> and(Specification<T> spec) {
            addSpec(spec);
            return this;
        }

        public Builder<T> ifHasText(String str,
                                    Function<String, Specification<T>> specSupplier) {
            if (StringUtils.hasText(str)) {
                addSpec(specSupplier.apply(str));
            }
            return this;
        }

        public Builder<T> ifTrue(Boolean cond,
                                 Supplier<Specification<T>> specSupplier) {
            if (cond != null && cond.booleanValue()) {
                addSpec(specSupplier.get());
            }
            return this;
        }

        public Specification<T> toSpec() {
            Specification<T> spec = Specification.where(null);
            for (Specification<T> s : specs) {
                spec = spec.and(s);
            }
            return spec;
        }
    }
}
```

## 5.9 동적 인스턴스 생성

- JPA는 쿼리 결과에서 임의 객체를 동적 생성하는 기능을 제공하고 있다.
    - full package name으로 클래스를 지정하고 `new` 키워드로 생성한다.

```java
@Query("""
        select new com.myshop.order.query.dto.OrderView(...)
        ...
    """)
List<OrderView> findOrderView(String orderId);
```

- 조회 전용 모델을 만드는 이유는 표현 영역을 통해 사용자에게 데이터를 보여주기 위함이다.
- 동적 인스턴스의 장점은 JPQL을 그대로 사용하면서 지연/즉시 로딩과 같은 고민 없이 데이터를 조회할 수 있는 점이다.

## 5.10 하이버네이트 @Subselect 사용

- `@Subselect`는 쿼리 결과를 `@Entity`로 매핑할 수 있는 유용한 기능이다.

```java
@Entity
@Immutable
@Subselect(
  """
  select o.order_number as number,
         o.version,
         o.orderer_id,
         o.order_name,
         o.total_amounts,
         o.receiver_name,
         o.state,
         o.order_date,
         p.pruduct_id,
         p.name as product_name
  from purchase_order o inner join order_line ol
    on o.order_number = ol.ourder_number
    cross join product p
  where ol.line_idx = 0
    and ol.product_id = p.product_id   
"""
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
  //...
  
  protected OrderSummary() {
  }
}
```

- `@Subselect` - 조회 쿼리를 값으로 갖는다.
    - select 쿼리의 결과를 매핑할 테이블처럼 사용한다.
- `@Immutable`
    - `@Subselect`로 매핑된 엔티티는 실제 테이블이 아니기 때문에 수정되면 변경 감지가 동작하여 에러가 발생할 것이다.
    - 때문에 `@Immutable` 애너테이션으로 변경을 무시하도록 한다.
- `@Synchronize`
    - 해당 엔티티와 관련된 테이블 목록을 명시
    - 하이버네이트는 엔티티를 로딩하기 전 지정한 테이블과 관련된 변경이 발생하면 플러시를 먼저 한다.

```java
Order orer = findOrder(orderNumber);
order.changeShippingInfo(newInfo);

// 위 order의 변경 사항이 flush 된 후의 상태를 가져올 수 있음
List<OrderSummary> summaries = orderSummarRepository.findByrdererId(userId);
```

- `@Subselect`는 일반 엔티티와 같기 때문에 `EntityManager`, `JPQL`, `Criteria`를 사용해서 평범하게 조회 가능하다.
    - `from` 절의 서브 쿼리로 사용되어 조회된다.
    - 서브 쿼리를 사용하고 싶지 않다면 네이티브 쿼리나 마이바티스 같은 별도 매퍼를 사용해야 한다.
