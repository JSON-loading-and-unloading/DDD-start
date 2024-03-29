<h1>리포지터리와 모델 구현</h1>

<h2>JPA를 이용한 리포지터리 구현</h2>

<h3>모듈 위치</h3>

![iiii](https://github.com/JSON-loading-and-unloading/DDD-start/assets/106163272/ac7d88f0-675d-4321-b950-4becd9a484c5)


리포지터리 인터페이스는 애그리거트와 같이 도메인 영역에 속하고, 리포지터리를 구현한 클래스는 인프라스트럭처 영역에 속한다.</br>
-> 가능하면 리포지터리 구현 클래스를 인프라스트럭처 영역에 위치시켜서 인프라스트럭처에 대한 의존을 낮춰야 한다.</br>

<h3>리포지터리 기본 기능 구현</h3>

리포지터리가 제공하는 기본 기능</br>
- ID로 애그리거트 조회하기</br>
- 애그리거트 저장하기</br>


```
  public interface OrderRepository{
        Order findById(OrderNo no);
        void save(Order order);
}

```

 - 인터페이스는 애그리거트 루트를 기준으로 작성한다.
 - findBy프로퍼티이름(프로퍼티 값) 형식 사용
</br>
해당 값의 애그리거트가 존재하면 Order를 리턴하고 존재하지 않는다면 null을 리턴 </br>
null을 사용하고 싶지 않다면 Optional을 사용해도 된다</br>.

<h2>스프링 데이터 jpa를 이용한 리포지터리 구현</h2>

- 스프링 데이터 JPA는 지정한 규칙에 맞게 리포지터리 인터페이스를 정의하면 리포지터리를 구현한 객체를 알아서 만들어 스프링 빈으로 등록해준다.</br>

스프링 데이터 JPA는 다음 규칙에 따라 작성한 인터페이스를 찾아서 인터페이스를 구현</br>
 - org.springframework.data.repository.Repository<T, ID> 인터페이스 상속</br>
 - T는 엔티티 타입을 지정하고 ID는 식별자 타입을 지정</br>

```

public interface OrderRepository extends Repository<Order, OrderNo> {
    Order findById(OrderNo id);
    void save(Order order);
}

```

다음은 예시이다.
</br>

<h2>매핑 구현</h2>

애그리거트 JPA 매핑 기본 규칙</br>

 - 애그리거트 루트는 엔티티이므로 @Entitiy로 매핑 설정한다.</br>

한 테이블에 엔티티와 밸류 데이터가 같이 있다면</br>
 - 밸류 @Enbeddable로 매핑 설정한다.</br>
 - 밸류 타입 프로퍼티는 @Embedded로 매핑 설정한다.</br>


루트 애그리거트 Order은 @Entitiy로 매핑</br>

```
@Entity
@Tagble(name = "purchase_order")
public class Order {
	...

	@Embedded
	Orderer orderer;
}

```

또한, Order에 속하는 Orderer는 밸류이므로 @Embeddable로 매핑한다.</br>

```
@Embeddable
public class Orderer {
	// MemberId에 정의된 칼럼 이름을 변경하기 위해
	// @AttributeOverride 애노테이션 사용
	@Embedded
	@AttributeOverrides(
		@AttributeOverride(name = "id", column = @Column(name = "orderer_id"))
	)
	private MemberId memberId;

	@Column(name = "orderer_name")
	private String name;

	...
}

```
Orderer에 MemeberId가 속해 있으므로 @Embedded를 이용해서 밸류 타입 프로퍼티를 설정한다.</br>
Orderer의 memberId는 Member 애그리거트를 ID로 참조한다.</br>

```
@Embaddable
public class MemberId implements Serializable {
	@Column(name="member_id")
	private String id;

	...
}

```

<h3>기본 생성자</h3>

```
@Embeddable
public class Receiver {
	@Column(name = "receiver_name")
	private String name;

	...

	protected Receiver() {} // JPA를 적용하기 위해 기본 생성자 추가

	public Receiver(String name, String phone) {
		this.name = name;
		this.phone = phone;
	}
}

```

엔티티와 밸류의 생성자는 객체를 생성할 때 필요한 것을 전달받는다.</br>

- Receiver가 불변 타입이면 생성 시점에 필요한 값을 모두 전달받으므로 값을 변경하는 set 메서드를 제공하지 않는다.</br>
- 이는 Receiver 클래스에 기본 생성자를 추가할 필요가 없다는 것을 의미한다.</br>

하지만 JPA에서 @Entity와 @Embeddable로 클래스를 매핑하려면 기본 생성자를 제공해야 한다.</br>
DB에서 데이터를 읽어와 매핑된 객체를 생성할 때 기본 생성자를 사용해서 객체를 생성하기 때문이다.</br>
-> 기본 생성자는 JPA 프로바이더가 객체를 생성할 때만 사용한다.</br>
-> 기본 생성자를 다른 코드에서 사용하면 값이 없는 온전하지 못한 객체를 만들게 된다.</br>
-> protected로 선언한다.</br></br>


