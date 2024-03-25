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

=> 부득이하게 한 트랜잭션으로 두 개 이상의 애그리거트를 수정해야 한다면 애그리거트에서 다른 애그리거트를 직접 수정하지 말고 응용 서비스에서 두 애그리거트를 수정하도록 구현한다.</br></br>

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
<h2>레포지터리와 애그리거트</h2>

- 애그리거트는 개념상 완전한 한 개의 도메인 모델을 표현하므로 객체의 영속성을 처리하는 레포지터리라는 애그리거트 단위로 존재한다.</br>
  ( Order와 OrderLine을 물리적으로 각각 별도의 DB 테이블에 저장한다고 해서 Order와 OrderLine을 위한 레포지터리를 각각 만들지 않는다. )</br>


리포지터리는 보통 다음의 두 메서드를 기본으로 제공한다.</br>
- save : 애그리거트 저장
- findById : ID로 애그리거트를 구함

애그리거트는 개념적으로 하나이므로 리포지터리는 애그리거트 전체를 저장소에 영속화해야 한다.</br>
ex) Order 애그리거트와 관련되 테이블이 세 개라면 Order 애그리거트를 저장할 때 애그리거트 루트와 매핑되는 테이블뿐만 아니라 애그리거트에 속한 모든 구성요소에 매핑된 테이블에 테이터를 저장해야한다.</br></br>


```
Order order = orderRepository.findById(orderId);

```
동일하게 애그리거트를 구하는 리포지터리 메서드는 완전한 애그리거트를 제공해야한다.</br>
즉, 다음 코드를 실행하면 order 애그리거트는 OrderLine, Orderer 등 모든 구성요소를 포함하고 있어야 한다.</br></br>


<h2>ID를 이용한 애그리거트 참조</h2>

애그리거트도 다른 애그리거트를 참조한다.</br>

![2322](https://github.com/JSON-loading-and-unloading/DDD-start/assets/106163272/6609e0e4-e578-495a-a876-513f288247ae)

 jpa는 @ManyToOne 등 애너테이션을 이용해서 연관된 객체를 로딩하는 기능을 제공하고 있으므로 필드를 이용해 다른 애그리거트를 쉽게 참조할 수 있다.</br>
 하지만 필드를 이용한 애그리거트 참조는 문제를 야기할 수 있다.</br>

 1. 편한 탐색 오용
 2. 성능에 대한 고민
 3. 확장 어려움

위 세 가지 문제를 완화할 때 사용할 수 있는 것이 ID를 이용해서 다른 애그리거트를 참조하는 것이다.</br>
![668](https://github.com/JSON-loading-and-unloading/DDD-start/assets/106163272/c2515b22-4c5b-4f56-ab06-22db6b2600ef)

ID 참조를 사용하면 모든 객체가 참조로 연결되지 않고 한 애그리거트에 속한 객체들만 참조로 연결된다.</br>
이는 애그리거트의 경계를 명확히 하고 애그리거트 간 물리적인 연결을 제거하기 때문에 모델의 복잡도를 낮춰준다.
</br></br>
<h3>ID를 이용한 참조와 조회 성능</h3>

한 DBMS에 데이터가 있다면 조인을 이용해서 한 번에 모든 데이터를 가져올 수 있음에도 불구하고 주문마다 상품 정보를 읽어오는 쿼리를 실행하게 된다.</br></br>


N+1문제는 주문 개수가 10개면 주문을 읽어오기 위한 1번의 쿼리와 주문별로 각 상품을 읽어오기 위한 10번의 쿼리를 실행한다. </br>
'조회 대상이 N개일 때 N개를 읽어오는 한 번의 쿼리와 연관된 데이터를  읽어오는 쿼리를 N번 실행한다' 해서 이를 N+1 조회 문제라고 부른다. </br>
ID를 이용한 애그리거트 참조는 지연로딩과 같은 효과를 만드는데 지연로딩과 관련된 대표적인 문제가 N+1조회 문제이다. </br></br></br>

N+1 조회 문제는 더 많은 쿼리를 실행하기 때문에 전체 조회 속도가 느려지는 원인이 된다.</br>
이를 해결하기 위한 방법은 조인을 사용하는 것이다. 하지만 이럴 경우 객체 참조 방식으로 다시 되돌리는 것이다.</br>

해결방법</br>

```
	@Repository
	public class JpaOrderViewDao implements OrderViewDao{
		@PersistenceContext
		private EntityManager em;
	
		@Override
		public List<OrderView> selectByOrderer(String ordererId){
			String selectQuery = 
					"select new com.myshop.order.application.dto.OrderView(o,m,p)"+
					"from Order o join o.orderLines ol, Member m, Product p" +
					"where o.orderer.memberId.id = :ordererId " +
					"and o.orderer.memeberId = m.id"+
					"and index(ol) = 0"+
					"and ol.productId = p.id" +
					"order by o.number.number desc";
			TypedQuery<OrderView> query = 
						em.createQuery(selectQuery, OrderView.calss);
			query.setParameter("ordererId", ordererId);
			return query.getResultList();
		}
	}

```

다음과 같이 JPA를 사용할 경우 JPQL을 사용해서 한 번의 쿼리로 데이터를 조회한다.</br>

<h2>애그리거트 간 집합 연관</h2>

보통 목록 관련 요구사항은 한 번에 전체 상품을 보여주기보다는 페이징을 이용해 제품을 나눠서 보여준다. </br> </br>

카테고리 입장을 1-N연관을 이용해 구현하면 다음과 같은 방식으로 코드를 작성해야 한다. </br>

```

	public class Category {
	  private Set<Product> products;
	  
	  
	  public List<Product> getProduct(int page, int size) {
	    List<Product> sortedProducts = sortById(products);
	    return sortedProducts.subList((page - 1) * size, page * size);
	  }
	}

```


```
 public class Product {
	private CategoryId categoryId;
	}

```
이 코드를 실제 DBMS와 연동해서 구현하면 Category에 속한 모든 Product를 조회하게 된다. </br>
Product 개수가 수만 개 정도로 많다면 이 코드를 실행할 때마다 실행 속도가 급격히 느려져 성능에 심각한 문제를 일으킬 것이다. </br> </br>

응용 서비스는 ProductRepository를 이용해서 categoryId가 지정한 카테고리 식별자인 Product 목록을 구한다. </br>

```
	public class ProductListService{
		public Page<Product> getProductOfCategory(Long categoryId, int page, int size){
			Category category = categoryRepository.findById(categoryId);
			
			List<Product> products = 
				productRepository.findByCategoryId(category.getId(),page,size);
			int totalCount = productRepository.countsByCategoryId(category.getId());
			return new Page(page,size,totalCount,products);
		}
	...
	}

```

M-N 연관은 개념적으로 양쪽 애그리거트에 컬렉션으로 연관을 만든다. </br>

 - 보통 특정 카테고리에 속한 상품 목록을 보여줄 때 목록 화면에서 각 상품이 속한 모든 카테고리를 상품 정보에 표시하지는 않는다.
 - 제품이 속한 모든 카테고리가 필요한 화면은 상품상세 화면이다.
 - 이러한 요구사항을 고려할 때 카테고리에서 상품으로의 집합 연관은 필요하지 않는다.
 - 즉 개념적으로는 상품과 카테고리 양방향 M-N 연관이 존재하지만 실제 구현에서는 상품에서 카테고리로의 단방향 M-N 연관만 적용하면 되는 것이다.

JPA ID 참조를 이용하여 M-N 단방향 연관을 구현할 수 있다. </br>

```
	public class ProductListService{
		public Page<Product> getProductOfCategory(Long categoryId, int page, int size){
			Category category = categoryRepository.findById(categoryId);
			
			List<Product> products = 
				productRepository.findByCategoryId(category.getId(),page,size);
			int totalCount = productRepository.countsByCategoryId(category.getId());
			return new Page(page,size,totalCount,products);
		}
	...
	}

```

위 매핑 사용 시 JPQL의 member of 연산자를 이용해서 특정 Category에 속한 Product 목록을 구하는 기능을 구현할 수 있다. </br>

```

@Repository
public class JpaProductRepository implements ProductRepository{
	@PersistenceContext
	private EntityManager entityManager;

	@Override
	public List<Product> findByCategoryId(CategoryId categoryId, int page, int size){
		TypedQuery<Product> query = entityManager.createQuery(
				"select p from Product p **where :catId member of p.categoryIds** order by p.id.id desc",
				Product.class);
		query.setParameter("catId",categoryId);
		query.setFirstResult((page-1)*size);
		query.setMaxResults(size);
		return query.getResultList();
	}
...
}

```

이 코드에서  :catId member of p.categoryIds는 categoryIds 컬렉션에 catId로 지정한 값이 존재하는지를 검사하기 위한 검색 조건이다. </br>

<h2>애그리거트를 팩토리로 사용하기</h2>

```
public class RegisterProductService {
	public ProductId registerNewProduct(NewProductRequest req) {
		Store account = accountRepository.findStoreById(req.getStoreId());
		checkNull(account);
		if (!account.isBlocked()) {
			throw new StoreBlockedException();
		}
		ProductId id = productRepository.nextId();
		Product product = new Product(id, account.getId(), ...);
		productRepository.save(product);
		return id;
	}
}

```

위 코드는 도메인 로직 처리가 응용 서비스에 노출 되었다. </br>
-> Store가 Product를 생성할 수 있는지를 판단하고 Product를 생성하는 것은 논리적으로 하나의 도메인 기능인데 이 도메인 기능을 응용 서비스에서 구현하고 있다. </br> </br>

이 도메인 기능을 넣기 위한 별도의 도메인 서비스나 팩토리 클래스를 만들 수도 있지만 이 기능을 Store 애그리거트에 구현할 수도 있다. </br>

```
	public class Store {
		public Product createProduct(ProductId id, ... ) {
			if (!account.isBlocked()) {
				throw new StoreBlockedException();
			}
			return new Product(id, account.getId(), ...);
		}
	}

```

Store애그리거트의 createdProduct()는 Product 애그리거트를 생성하는 팩토리 역할을 한다. </br>

```
public class Store extends Member {
	public Product createProduct(ProductId id, ... ) {
		if (!account.isBlocked()) {
			throw new StoreBlockedException();
		}
		return new Product(id, account.getId(), ...);
	}
}

```

앞선 코드와 차이점은 응용 서비스에서 더 이상 Store의 상태를 확인하지 않는다는 것이다. </br>
=> Product 생성 가능 여부를 확인하는 도메인 로직을 변경해도 도메인 영역의 Store만 변경하면 되고 응용 서비스는 영향을 받지 않는다. </br> </br>

애그리거트가 갖고 있는 데이터를 이용해서 다른 애그리거트를 생성해야 한다면 애그리거트에 팩토리 메서드를 구현하는 것을 고려해 보자 </br>


