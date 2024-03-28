# Chapter3 애그리거트

## 3.1 애그리거트

- **애그리거트는 복잡한 도메인을 이해하고 쉽게 관리하기 위해 상위 수준에서 모델을 조망할 수 있는 방법이다**
    - 도메인 관계를 파악하기 어렵다는 것은 코드를 변경하고 확장하는 것이 어려워진다는 것
    - 애그리거트 단위로 일관성을 관리할 수 있다.
- 애그리거트는 **경계**를 갖는다.
    - 한 애그리거트에 속한 객체는 다른 애그리거트에 속하지 않는다.
    - 애그리거트는 자기 자신을 관리할 뿐 다른 애그리거트를 관리하지 않는다.
    - ex) 주문 애그리거트는 배송지나 상품 개수를 변경할 수 있지만 회원 비밀번호나 상품 가격을 변경하지는 않는다.
- 경계 설정의 기본은 도메인 규칙과 요구 사항이다.
    - 함께 생성되는 요소는 한 애그리거트일 가능성이 높다.
    - ex)상품 개수, 배송지 정보, 주문자 정보는 ‘주문’과 함께 생성되므로 한 애그리거트다.
- ‘A가 B를 갖는다’라고 해서 무조건 한 애그리거트인 것은 아니다.
    - ex) ‘상품’과 ‘상품 리뷰’는 함께 생성되지도 않고, 함께 변경되지도 않는다.

## 3.2 애그리거트 루트

- 애그리거트는 여러 객체로 구성되기에 모든 객체가 정상 상태를 가져야 한다.
    - ex) `Order`의 `totalAmounts` 값은 `OrderLine`의 `quantity` 값이 변경되면 함께 바뀌어야 한다.
- **애그리거트 루트는 애그리거트의 모든 객체의 일관된 상태를 유지시키는 관리 주체이다.**
    - 애그리거트에 속한 객체는 애그리거트 루트 엔티티에 직접 또는 간접적으로 속하게 된다.

### 3.2.1 도메인 규칙과 일관성

- 애그리거트 루트가 제공하는 메서드는 도메인 규칙에 따라 애그리거트 내 모든 객체의 일관성이 깨지지 않도록 구현해야 한다.
- 애그리거트 외부에서 애그리거트 객체를 직접 변경하면 업무 규칙을 무시하는 것과 같다.
    - `set` 메서드를 `public`으로 구현하지 않아야 한다.
    - 밸류 타입은 불변으로 구현한다.
- 업무 규칙 일관성을 검사하는 로직을 응용 계층에서 구현하는 것은 유지 보수에 도움이 되지 않는다.
    - 주요 도메인 로직이 중복으로 구현될 수도 있다.
- 애그리거트 루트가 도메인 규칙을 올바르게만 구현하면 애그리거트 전체 일관성을 올바르게 유지할 수 있다.

### 3.2.2 애그리거트 루트의 기능 구현

- 애그리거트 루트는 애그리거트 내부의 다른 객체를 조합해서 기능을 완성한다.

    ```java
    // Order는 총 주문 금액을 구하기 위해 OrderLine 목록을 사용한다.
    public class Order {
    	private Money totalAmounts;
    	private List<OrderLine> orderLines;
    
    	private void calculateTotalAmounts() {
    		int sum = orderLines.stream()
    			.mapToInt(ol -> ol.getPrice() * ol.getQuantity())
    			.sum();
    		this.totalAmounts = new Money(sum);
    	}
    ```

- 애그리거트 루트가 구성 요소의 상태를 참조하는 것 뿐만 아니라 기능 실행을 위임하기도 한다.

    ```java
    // OrderLine 목록을 가지는 일급 컬렉션
    public class OrderLines {
    	private List<OrderLine> lines;
    
    	public Money getTotalAmounts() { ... }
    	public void changeOrderLines(List<OrderLine> newLines) {
    		this.lines = newLines;
    	}
    }
    
    public class Order {
    	private OrderLines orderLines;
    	private Money totalAmounts;
    
    	// 일급 컬렉션 OrderLines에 기능 실행을 위임
    	public void changeOrderLines(List<OrderLine> newLines) {
    		orderLines.changeOrderLines(newLines);
    		this.totalAmounts = orderLines.getTotalAmounts();
    	}
    }
    ```

    - `Order`에서 `getOrderLines()` 메서드를 통해 `OrderLines`를 반환한 뒤, `Order` 객체 외부에서에서 `OrderLines.changeOrderLines()`를 호출하면 업무 규칙 일관성이 깨지게 된다.
    - 때문에 애그리거트 외부에서 `OrderLines`를 변경할 수 없도록 불변으로 구현해야 한다. (불변 컬렉션 사용)

### 3.2.3 트랜잭션 범위

- **한 트랜잭션에서는 한 개의 애그리거트만 수정해야 한다.**
    - 트랜잭션에 복수의 테이블을 수정하면 잠금을 거는 테이블도 많아지고 이는 전체적인 성능 저하를 유발한다.
    - 한 애그리거트에서 다른 애그리거트를 변경하지 않아야 한다는 것
- 애그리거트는 최대한 서로 독립적이어야 한다.
    - 다른 애그리거트의 상태까지 관리하게 되면 애그리거트 간 결합도가 높아지고 유지보수 비용이 증가한다.
- 한 트랜잭션에서 두 개 이상의 애그리거트를 수정해야 한다면 응용 서비스에서 두 애그리거트를 수정하도록 구현한다.
    - 도메인 객체 내부에서 다른 애그리거트 도메인을 변경하지 말아야 한다.

    ```java
    // 응용 서비스에서 각 애그리거트를 변경하는 코드
    public ChangeOrderService{
    
    	@Transactional
    	public void changeShippingInfo(OrderId id, ShippingInfo newShippingInfo, boolean useNewShippingAddrAsMemberAddr) {
    		Order order = findOrderById(id);
    		order.shipTo(newShippingInfo);
    		if useNewShippingAsMemberAddr) {
    			Member member = findMember(order.getOrderer());
    			member.changeAddress(newShippingInfo.getAddress());
    		}
    	}
    }
    ```

- 도메인 이벤트를 사용하면 한 트랜잭션에서 동기나 비동기로 다른 애그리거트 상태를 변경하도록 구현할 수 있다.
- 다음의 경우엔 예외적으로 두 개 이상의 애그리거트 변경을 고려할 수 있다.
    - **팀 표준**: 팀이나 조직의 표준에 따라 사용자 유스케이스와 관련된 응용 서비스 기능을 한 트랜잭션으로 실행해야 하는 경우가 있다.
    - **기술 제약**: 기술적으로 이벤트 방식을 도입할 수 없는 경우
    - **UI 구현의 편리**: 운영자의 편리함을 위해 주문 목록 화면에서 여러 주문의 상태를 한 번에 변경하고 싶은 경우

## 3.3 리포지터리와 애그리거트

- **객체의 영속성을 처리하는 리포지터리는 애그리거트 단위로 존재한다.**
    - 애그리거트는 개념상 완전한 하나의 도메인 모델을 표현하기 때문
    - `Order`와 `OrderLine`을 별도의 DB 테이블에 저장한다고 해서 각각의 리포지터리를 만들지는 않는다.
- 리포지터리는 다음의 두 메서드를 기본으로 제공한다.
    - `save()`
    - `findById()`
- 애그리거트 루트를 저장할 때 애그리거트에 속한 모든 구성요소에 매핑된 테이블에 데이터를 저장해야 한다.
- 애그리거트를 조회할 때에는 완전한 애그리거트를 제공해야 한다.
    - ex) `Order` 에그리거트를 조회하면 내부에 `OrderLine`, `Orderer` 등 모든 요소를 포함해야 한다.
    - 온전한 애그리거트가 아니면 기능 실행 도중 NPE와 같은 문제가 발생한다.
- RDBMS를 이용하면 트랜잭션을 이용해 애그리거트의 일관성 있는 변경을 저장소에 반영시킬 수 있다.

## 3.4 ID를 이용한 애그리거트 참조

- **애그리거트가 다른 애그리거트를 참조할 수 있다.**
    - 애그리거트 관리 주체는 애그리거트 루트이므로 다른 애그리거트 루트를 참조한다는 것과 같다.
- 애그리거트 간 참조는 필드를 통해 쉽게 구현할 수 있다.

    ```java
    public class Order {
    	private Member orderer;
    	// ...
    }
    ```

    - 필드로 직접 참조 하는 것은 구현의 편리함을 제공한다.
    - JPA와 같은 ORM 기술 덕에 직접 참조를 쉽게 구현할 수 있다.
- 필드를 이용한 애그리거트 참조는 다음 문제를 야기할 수 있다.
    - **편한 탐색 오용**
        - 다른 애그리거트 객체에 접근 가능하기에 다른 애그리거트의 상태를 쉽게 변경해버릴 수도 있다.
        - 이는 애그리거트 간 의존 결합도를 높여 애그리거트 변경을 어렵게 만든다.
    - **성능에 대한 고민**
        - JPA를 사용한다면 조회를 어떤 목적으로 하느냐에 따라 지연로딩/즉시로딩을 고민해야 한다.
        - 다양한 경우의 수를 고려하여 연관 매핑과 JPQL/Querydsl 쿼리 로딩 전략을 결정해야 한다.
    - **확장 어려움**
        - 트래픽이 증가하면 부하 분산을 위해 하위 도메인별로 서로 다른 저장소를 사용하기도 한다.
        - 이는 더 이상 애그리거트 루트를 참조하기 위해 JPA와 같은 단일 기술을 사용할 수 없음을 의미한다.
- **ID를 이용해서 다른 애그리거트를 참조**하면 위 세 가지 문제를 완화시킬 수 있다.

    ```java
    public class Order {
    	private MemberId memberId;
    }
    ```

    - 한 애그리거트에 속한 객체들만 참조로 연결하고 다른 애그리거트는 ID로 참조한다.
    - 애그리거트 간 경계를 명확히하고 물리적 연결을 제거하기에 모델의 복잡도를 낮춰준다.
    - 애그리거트 간 의존을 제거하고 응집도를 높여주기도 한다.
    - 참조를 지연 로딩으로 할지 즉시 로딩으로 할지 고민하지 않아도 되기에 구현 복잡도도 낮아진다.
    - 참조하는 애그리거트 객체가 필요하면 응용 서비스에서 ID를 통해 로딩하면 된다. (지연 로딩과 동일한 결과)
    - 다른 애그리거트를 수정하는 문제를 근원적으로 방지할 수 있다.
    - 애그리거트별로 다른 구현 기술을 사용하는 것도 가능해 진다.

### 3.4.1 ID를 이용한 참조와 조회 성능

- 다른 애그리거트를 ID로 참조하면 여러 애그리거트를 조회할 때 성능 문제가 발생할 수도 있다.
    - ex) 주문 10개를 가져오고, 주문 당 상품 정보를 10번 추가로 더 읽어야 한다. → N+1 문제
    - 직접 참조를 했다면 조인을 통해 한 번에 가져올 수 있지만 ID 참조는 불가능하다.
- **조회 전용 쿼리**를 사용하면 ID 참조 방식을 사용하면서 N + 1 문제를 해결할 수 있다.
    - 복잡한 조회를 위한 별도의 리포지터리를 만들고 JPQL이나 Querydsl을 이용해 직접 조인 쿼리를 짜서 DTO로 필요한 결과를 반환하면 된다.
    - 그 외에 조회에 특화된 기술을 사용해서 구현할 수도 있다.
- 애그리거트마다 서로 다른 저장소를 사용하면 한 번에 조회할 수 없기에 캐시나 조회 전용 저장소를 따로 구성한다.
    - 코드가 복잡해지지만 시스템 처리량을 높일 수 있다.
    - 한 대 DB로 대응할 수 없는 트래픽이 발생하는 경우 필수로 선택해야 하는 기법이다.

## 3.5 애그리거트 간 집합 연관

- 1-N(일대다) 연관에서는 1이 N을 갖는 것이 아닌 N이 1을 참조하는 식으로 구현해야 한다.

    ```java
    public class Product {
    	// ...
    	private CategoryId categoryId;
    	// ...
    ```

    - 1이 N을 가지도록 하면 DBMS와 연동할 때 N이 수십만개가 되면 모두 조회해야 하기에 성능에 심각한 문제를 일으킬 수 있다.

## 3.6 애그리거트를 팩토리로 사용하기

**애그리거트가 갖고 있는 데이터를 이용해서 다른 애그리거트를 생성해야 한다면 애그리거트에 팩토리 메서드를 구현하는 것을 고려해 보자.**

- ‘신고 당해 차단된 상점은 물건을 등록하지 못한다’ 기능을 팩토리 없이 구현

    ```java
    public class RegisteredProductService {
    
    	public ProductId registerNewProduct(NewProductRequest req) {
    		Store store = findStoreById(req.getStoreId());
    		if (store.isBlocked()) {
    			throw new StoreBlockedException();
    		}
    		Product product = new Product(store.getId(), req.getName());
    		return new ProductId(productRepository.save(product));
    	}
    }
    ```

    - `Product` 생성 코드와 `Product`를 생성 가능한지 판단하는 코드가 분리되어 있다.
    - 주요 도메인 로직이 응용 서비스에 노출되어 있다.
- `Store` 애그리거트에 `Product` 생성 기능을 위임

    ```java
    public class Store {
    	
    	public Product createProduct(ProductId newProductId, String name) {
    		if (isBlocked()) {
    			throw new StoreBlockedException();
    		}
    		return new Product(this.id, name);
    }
    
    public class RegisteredProductService {
    
    	public ProductId registerNewProduct(NewProductRequest req) {
    		Store store = findStoreById(req.getStoreId());
    		Product product = store.createProduct(req.getName());
    		return new ProductId(productRepository.save(product));
    	}
    }
    ```

    - `Store.createProduct()`는 상품 애그리거트를 생성하는 팩토리 역할을 한다.
    - 팩토리이면서 주요 도메인 로직을 구현하고 있어 응용 서비스에 로직이 노출되지 않는다.
    - `Product` 생성 가능 여부 로직을 변경해도 도메인 영역의 `Store`만 변경하면 되고 응용 서비스는 영향 받지 않는다.
    - 도메인 응집도도 높아졌다.
- `Store` 애그리거트가 `Product`를 생성할 때 많은 정보를 알아야 한다면 다른 팩토리에 위임하는 방법도 있다.

    ```java
    public class Store {
    	
    	public Product createProduct(ProductInfo info) {
    		if (isBlocked()) {
    			throw new StoreBlockedException();
    		}
    		return ProductFactory.create(this.id, info);
    	}
    }
    ```

    - 다른 팩토리를 쓰더라도 도메인 로직은 한곳에 계속 위치한다.
