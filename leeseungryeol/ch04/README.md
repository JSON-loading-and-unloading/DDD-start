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

<h3>필드 접근 방식 사용</h3>

엔티티에 프로퍼티를 위한 공개 get/set 메서드를 추가하면 도메인의 의도가 사라지고 객체가 아닌 데이터 기반으로 엔티티를 구현할 가능성이 높아진다.
-> 상태 변경을 위한 setState() 메서드보다 calcel()메서드가 도메인을 잘 표현한다.
객체가 제공할 기능 중심으로 엔티티를 구현하게끔 유도하려면 JPA 매핑 처리를 프로퍼티 방식이 아닌 필드 방식으로 선택해서 불필요한 get/set 메서드를 구현하지 말아야 한다.

<h3>AttributeConverter를 이용한 밸류 매핑 처리</h3>

두 개 이상의 프로퍼티를 가진 밸류 타입을 한 개 칼럼에 매핑하려면 @Embeddable 애네테이션으로 처리할 수 있다. 이럴 때 사용할 수 있는 것이 AttributeConverter이다.

```
@Converter(autoApply = true)
public class MoneyConverter implements AttributeConverter<Money, Integer> {
  @Override
  public Integer convertToDatabaseColumn(Money money) {
    if (money == null) {
      return null;
    } else {
      return money.getValue();
    }
  }
  
  @Override
  public Money convertToEntityAttribute(Integer value) {
    if (value == null) return null;
    else return new Money(value);
  }
}

```

@Converter 애너테이션의 autoApply 속성값을 보면 true가 있다.
true일 경우 모델에 출현하는 모든 Money타입의 프로퍼티에 대해 MoneyConverter를 자동으로 적용한다.
false일 경우(디폴트) 프로퍼티 값을 변환할 때 사용할 컨버터를 직접 지정애햐한다.

```
@Column()
@Convert(converter = MoneyConverter.class)
private Money ~

```

<h3>밸류 컬렉션: 별도 테이블 매핑</h3>

Order 엔티티는 한 개 이상의 OrderLine을 가질 수 있다.
밸류 컬렉션을 별도 테이블로 매핑할 떄는 @ElementCollection과 @CollectionTable을 함께 사용한다.

```
@Entity
@Table(name = "purchase_order")
public class Order {

  // ...
  @ElementCollection
  @CollectionTable(name = "order_line", joinColumns = @JoinColumn(name = "order_number"))
  @OrderColumn(name = "line_idx")
  private list<OrderLine> orderLines;

}

@Embeddable
public class OrderLine {

  // ...
  @Embedded
  private ProductId productId;

}

```

OrderLine의 매핑을 함께 표시했는데 OrderLine에는 List의 인덱스 값을 저장하기 위한 프로퍼티가 존재하지 않는다.
-> List타입 자체가 인덱스를 가지고 있기 때문이다.
-> Jpa는 @OrderColumn애너테이션을 이용해서 지정한 컬럼에 리스트 인덱스 값을 저장한다.
- @CollectionTable은 밸류를 저장할 테이블을 지정한다.
- name 속성은 테이블 이름을 지정하고 joinColumn속성은 외부키로 사용할 컬럼을 지정한다.


<h3>밸류 컬렉션: 한 개 칼럼 매핑</h3>

밸류 컬렉션을 별도 테이블이 아닌 한 개 칼럼에 저장해야 할 때가 있다.
DB에는 한 개 칼럼에 콤마로 구분해서 저장해야 할 때가 있다.

```
ublic class EmailSet {

  private Set<Email> emails = new HashSet<>();

  private EmailSet() {
  }

  private EmailSet(Set<Email> emails) {
    this.emails.addAll(emails);
  }

  public Set<Email> getEmails() {
    return Collections.unmodifiableSet(emails);
  }

}

@Converter
public class EmailSetConverter implements AttributeConveter<EmailSet, String> {

  @Override
  public String convertToDatabaseColumn(EmailSet attribute) {
    if (attribute == null) {
      return null;
    }
    return attribute.getEmails().stream()
        .map(Email::toString)
        .collect(Collectors.joining(","));
  }

  @Override
  public EmailSet convertToEntityAttribute(String dbData) {
    if (dbData == null) {
      return null;
    }
    String[] emails = dbData.split(",");
    Set<Email> emailSet = Arrays.stream(emails)
        .map(value -> new Email(value))
        .collect(toSet());
    return new EmailSet(emailSet);
  }

}
@Column(name = "emails")
@Convert(converter = EmailSetConverter.class)
private EmailSet emailSet;

```

<h3>밸류를 이용한 ID 매핑</h3>

밸류 타입을 식별자로 매핑하면 @Id 대신 @EmbeddedId 애너테이션을 사용한다.

```
@Embeddable
public class OrderNo implements Serializable {

  @Column(name = "order_number")
  private String number;

  public boolean is2ndGeneration() {
    return number.startsWith("N");
  }
  
  // ...
}

```

Jpa에서 식별자 타입은 Serializable 타입이어야 하므로 식별자로 사용할 밸류 Serializable인터페이스를 상속받아야 한다.

<h3>별로 테이블에 저장하는 밸류 매핑</h3>

 - 애그리거트에서 루트 엔티티를 뺀 나머지 구성요소는 대부분 밸류이다.
 - 루트 엔티팅 외에 또 다른 엔티티가 있다면 진짜 엔티티인지 의심해보자
 - 단지 별도 테이블에 데이터를 저장한다고 해서 엔티티인 것은 아니다.
 - 주문 애그리거트도 OrderLine을 별도 테이블에 저장하지만 OrderLine 자체는 엔티티가 아니라 밸류이다.

->
- 밸류가 아니라 엔티티가 확실하다면 해당 엔티티가 다른 애그리거트는 아닌지 확인해야 한다.
- 특히 자신만의 독자적인 라이프 사이클을 갖는다면 구분되는 애그리거트일 가능성이 높다.


애그리거트에 속한 객체가 밸류인지 엔티티인지 구분하는 방법은 고유 식별자를 갖는지를 확인하는 것이다.
하지만 식별자를 찾을 때 매핑되는 테이블의 식별자를 애그리거트 구성요소의 식별자와 동일한 것으로 착각하면 안된다.
-> 별도 테이블로 저장하고 테이블에 pk가 있다고 해서 테이블과 매핑되는 애그리거트 구성요소가 항상 고유 식별자를 갖는 것은 아니기 때문이다.
![image](https://github.com/JSON-loading-and-unloading/DDD-start/assets/106163272/b9af49a5-1121-47c4-9ea1-79c92db6911c)

밸류를 매핑 한 테이블을 지정하기 위해 @SecondaryTable과 @AttributeOverride를 사용한다.

```
@Entity
@Table(name = "article")
@SecondaryTable(
    name = "article_content",
    pkJoinColumns = @PrimaryKeyJoinColumn(name = "id")
)
public class Article {

  @Id
  private Long id;

  private String title;
  
  @AttributeOverrides({
      @AttributeOverride(name = "content", column = @Column(table = "article_content", name = "content")),
      @AttributeOverride(name = "contentType", column = @Column(table = "article_content", name = "content_type"))
  })
  private ArticleContent content;
  
}

```

- @SecondaryTable의 name 속성은 밸류를 저장할 테이블을 지정한다.
- pkJoinColumns 속성은 밸류 테이블에서 엔티티 테이블로 조인할 때 사용할 컬럼을 지정한다.
- content 필드에 @AttributeOverride를 적용했는데 이 애너테이션을 사용해서 해당 밸류 데이터가 저장된 테이블 이름을 지정한다.


<h3>밸류 컬렉션을 @Entitiy로 매핑하기</h3>


