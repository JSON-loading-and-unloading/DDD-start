# Chapter4 리포지터리와 모델 구현

## 4.1 JPA를 이용한 리포지터리 구현

RDBMS 사용 시 객체 기반의 도메인 모델과 관계형 데이터 모델 간 매핑을 처리하는 기술로 ORM만한 것이 없다.

### 4.1.1 모듈 위치

- 리포지터리 인터페이스는 애그리거트와 같이 도메인 영역에 속한다.
- 리포지터리를 구현한 클래스는 인프라스트럭쳐 영역에 속한다.
- 이는 인프라스트럭처에 대한 의존을 낮추기 위함이다.

## 4.3 매핑 구현

### 4.3.1 엔티티와 밸류 기본 매핑 구현

- 애그리거트 루트는 `@Entity`로 매핑 설정한다.

    ```java
    @Entity
    @Table(name = "purchase_order")
    public class Order { ... }
    ```

- 밸류는 `@Embeddable`로 매핑 설정한다.

    ```java
    @Embeddable
    public class Orderer {
    
    	@Embedded
    	@AttributeOverridesss(
    		@AttributeOverride(name = "id", @Column(name = "orderer_id"))
    	)
    	private MemberId memberId
    ```

    - 위 예제에선 `MemberId`에 정의된 칼럼 이름을 변경하기 위해 `@AttributeOvveride` 애너테이션을 사용했다.

    ```java
    @Embeddable
    public class MemberId implements Serializable {
    	@Column(name = "member_id")
    	private String id;
    }
    ```

- 밸류 타입 프로퍼티는 `@Embedded`로 매핑 설정한다.

    ```java
    @Embedded
    private Orderer orderer;
    ```


### 4.3.4 AttributeConverter를 이용한 밸류 매핑 처리

- 밸류 타입의 여러 프로퍼티를 한 개 칼럼에 매핑하거나 다른 형식으로 DB 칼럼에 적용해야할 때가 있다.

    ```java
    public class Length {
    	private int value;
    	private String unit;
    	// ...
    }
    // -> DB에 'WIDTH VARCHAR(20)'으로 저장하려면?
    ```

- `AttributeConverter`를 사용하면 밸류 타입과 칼럼 데이터 간 변환을 처리할 수 있다.

    ```java
    public interface AttributeConverter<X, Y> {
    	Y convertToDatabaseColumn(X attribute);
    	X convertToEntityAttribute(Y dbData);
    }
    ```

    - X는 밸류 타입, Y는 DB 타입이다.
    - 이 인터페이스를 구현한 뒤 밸류 타입 프로퍼티에 적용하면 된다.

    ```java
    @Embedded
    @Converter(converter = LengthConverter.class)
    private Length length
    ```

    - `@Converter`의 `autoApply` 속성은 기본적으로 `false`인데 이는 컨버터를 직접 지정해야 한다는 것이다.

### 4.3.7 밸류를 이용한 ID 매핑

- **식별자라는 의미를 부각시키기 위해 식별자 자체를 밸류 타입으로 만들 수도 있다.**

    ```java
    @Entity
    @Table(name = "purchase_order")
    public class Order {
    	@EmbeddedId
    	private OrderNo number;
    }
    ```

- 식별자 타입은 `Serializable` 타입이어야 한다.

    ```java
    @Embeddable
    public class OrderNo implements Serializable {
    	@Column(name="order_number")
    	private String number;
    }
    ```

- 밸류 타입으로 식별자를 구현하면 식별자에 기능을 추가할 수 있는 장점이 있다.
- `equals()`와 `hashcode()`를 알맞게 구현해야 한다.

### 4.3.8 별도 테이블에 저장하는 밸류 매핑

- 애그리거트에서 루트 엔티티를 뺀 나머지 구성 요소는 대부분 밸류이다.
    - 단지 별도 테이블에 저장한다고 해서 엔티티인 것은 아니다.
    - 고유 식별자를 가지지 않는다면 밸류 타입이다.
- 밸류가 아니라 엔티티가 확실하다면 다른 애그리거트는 아닌지 확인해야 한다.
    - ex) ‘상품’과 ‘상품 리뷰’는 한 애그리거트처럼 보이지만 서로 생명 주기도 다르고 상품이 리뷰에 영향을 주지도 않기에 다른 애그리거트다.
- ‘게시글’ 엔티티에서 ‘게시글 내용’을 별도의 테이블로 뽑는다고 해서 ‘게시글 내용’이 엔티티가 되는 것은 아니다.
- 밸류 타입을 별도의 테이블로 사용하려면 `@SecondaryTable`과 `@AttributeOvveride`를 사용한다.

    ```java
    @Entity
    @SecondaryTable(
    	name = "article_content",
    	pkJoinColumns = @PrimaryKeyJoinColumn(name = "id")
    )
    public class Article {
    
    	// ...
    	@AttributeOverrides({
    		@AttributeOverride(
    			name = "content",
    			column = @Column(table=="article_content", name="content")),
    		@AttributeOvveride(
    			name = "contentType",
    			column = @Column(table="article_content", name="content_type"))
    	})
    	@Embedded
    	private ArticleContent content;
    	// ...
    }
    ```

    - `@SecondaryTable`의 `name` 속성은 밸류를 저장할 테이블을 지정한다.
    - `pkJoinColumns` 속성은 밸류 테이블에서 엔티티 테이블로 조인할 때 사용할 칼럼을 지정한다.
    - 이 방식을 사용하면 지연 로딩을 사용할 수 없다.

### 4.3.9 밸류 컬렉션은 @Entity로 매핑하기

- 개념적으로 밸류지만 구현 기술의 한계나 팀 표준에 의해 `@Entity`를 사용해야 할 때도 있다.
    - 밸류에 상속 구조를 적용해야 하는 경우 `@Embeddable` 타입 클래스는 상속 매핑을 지원하지 않기에 엔티티로 만들 수밖에 없다.

### 4.3.10 ID 참조와 조인 테이블을 이용한 단방향 M-N 매핑

- 애그리거트 간 집합 연관은 성능 상 이유로 피해야 하지만 I**D참조를 이용한 단방향 집합 연관을 적용**할 수는 있다.

```java
@Entity
public class Product {
	@EmbeddedId
	private ProductId id;

	@ElementCollection
	@CollectionTable(
		name = "Product_category",
		joinColumns = @JoinColumn(name="product_id")
	)
	private Set<CategoryId> categoryIds;
	// ...
}
```

- `@ElementCollection`을 이용하기에 `Product`를 삭제할 때 매핑에 사용한 조인 테이블의 데이터도 함께 삭제된다.
- 직접 참조 방식이었다면 영속성 전파나 로딩 전략을 고민해야 하는데 ID참조이기에 이런 고민을 없앨 수 있다.

## 4.4 애그리거트 로딩 전략

- **애그리거트 루트를 로딩하면 루트에 속한 모든 객체가 완전한 상태여야 한다.**
    - 상태 변경 기능을 실행할 때 애그리거트 상태가 완전해야 업무 규칙에 맞게 변경할 수 있다.
    - 표현 영역에서 애그리거트의 상태 정보를 보여줄 때 필요하다.
- 즉시 로딩으로 설정하면 조인 등의 이유로 성능 문제가 많이 발생하기에 지연 로딩으로 설정하는 것이 좋다.
    - 조회를 위한 기능을 실행할 때는 별도의 조회 전용 기능과 모델을 구현하는 방식을 사용하면 된다.
    - 상태 변경은 조회 기능보다 호출 빈도가 낮기 때문에 지연 로딩으로 인한 추가 쿼리는 보통 문제되지 않는다.

## 4.5 애그리거트의 영속성 전파

- 애그리거트가 완전한 상태여야 한다는 것은 **애그리거트 루트를 저장하고 삭제할 때도 하나로 처리해야 함**을 의미한다.
- `cascade` 속성과 `orphanRemoval` 속성을 사용하여 구현할 수 있다.

    ```java
    @OneToMany(cascade = {CascadeType.PERSIST, CascadeType.REMOVE},
    	orphanRemoval = true)
    @JoinColumn(name = "product_id")
    private List<Image> images = new ArrayList<>();
    ```


## 4.6 식별자 생성 기능

- 식별자는 크게 세 방식 중 하나로 생성한다.
    - 사용자가 직접 생성
    - 도메인 로직으로 생성
    - DB를 이용한 일련번호 사용
- **식별자 생성 규칙이 있다면 엔티티를 생성할 때 도메인 영역에 위치한 별도 서비스가 식별자를 생성해야 한다.**
    - 이를 **도메인 서비스**라고 한다.

```java
public class CreateProductService {

	private ProductIdService idService;
	private ProductRepository productRepositoroy;

	@Transactional
	public ProductId createProduct(ProductCreationReq req) {
		ProductId id = idService.nextId();
		Product product = new Product(id, req.getDetail());
		productRepository.save(product);
		return id;
	}
}
```

- DB 자동 증가 칼럼을 식별자로 사용하려면 `@GeneratedValue`를 사용하면 된다.

## 4.7 도메인 구현과 DIP

- JPA를 사용하는 리포지터리는 DIP 원칙을 어기고 있다.
    - 도메인 영역의 엔티티엔 JPA에 특화된 애너테이션들을 사용하고 있다.
    - 도메인 패키지의 리포지터리 인터페이스가 `JpaRepository`를 상속하고 있다.
    - 즉 도메인이 인프라에 의존하는 것이다.
    - 도메인을 순수하게 유지하려면 JPA 관련 코드를 걷어내고 JPA를 연동하기 위한 클래스를 추가하고 별도의 작업을 해야 한다.
- 하지만 도메인 영역에 JPA 관련 코드가 들어가는 것은 합리적이다.
    - DIP를 적용하는 주된 이유는 저수준 구현이 변경되더라도 고수준이 영향을 받지 않도록 하기 위함이다.
    - 리포지터리와 도메인 모델의 구현 기술은 거의 바뀌지 않는다.
    - 개발 편의성과 실용성 또한 중요하다.
      
