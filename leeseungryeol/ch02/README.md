

<h1>아키텍처 개요</h1>

<h2>네 개의 영역</h2>

표현, 응용, 도메인, 인프라스트럭처는 아키텍처를 설계할 때 출현하는 네 가지 영역이다.</br>

![인프라](https://github.com/JSON-loading-and-unloading/DDD-start/assets/106163272/6bfe6ece-f8ca-46c1-850d-dbf42046018e)

표현 영역 : 웹 브라우저가 HTTP 요청 파라미터로 전송한 데이터를 응용 서비스가 요구하는 형식의 객체 타입으로 변환해서 전달하고, 응용 서비스가 리턴한 결과를 JSON 형식으로 변환해서 HTTP 응답으로 웹 브라우저에 전송한다.</br>
응용 영역 : 시스템이 사용자에게 제공해야 할 기능을 구현하는데 '주문 등록', '주문 취소', '상품 상세 조회'와 같은 기능 구현을 예를 들 수 있다. </br>
            ( 응용 영역은 기능을 구현하기 위해 도메인 영역의 도메인 모델을 사용한다, 응용 서비스는 로직을 직접 수행하기보다는 도메인 모델에 로직 수행을 위임한다.)</br>
![응용](https://github.com/JSON-loading-and-unloading/DDD-start/assets/106163272/22bb39ae-eada-44c7-a6f4-ab2907e36aa6)


도메인 영역 : 도메인 모델을 구현 ( Order, OrderLine, ShippingInfo )와 같은 모델이 이 영역에 해당</br>
인프라스트럭처 영역 : 구현 기술에 대한 것을 다룸.( RDBMS 연동을 처리하고, 메시징 큐에 메시지를 전송, 몽고DB나 레디스와 데이터 연동을 처리)</br></br>

<h2>계층 구조 아키텍처</h2>



![아키텍처](https://github.com/JSON-loading-and-unloading/DDD-start/assets/106163272/80ed242e-8fd9-4b57-9194-b767e23f0a52)


계층 구조는 그 특성상 상위 계층에서 하위계층으로의 의존만 존재하고 하위 계층은 상위 계층에 의존하지 않는다.</br></br>

응용 영역이 인프라스트럭처영역에 의존하는 경우가 있다.</br>
하지만 문제를 가지고 있다.</br>
1. 테스트하기 어렵다.(인프라스트럭처 파일을 모두 구성 후 테스트 가능)
2. 구현 방식을 변경하기 어렵다.(응용 영역이 인프라스트럭처 영역의 의존하고 있을 수 있다.)


<h2>DIP</h2>


![ddd](https://github.com/JSON-loading-and-unloading/DDD-start/assets/106163272/c9742fd8-6dbd-4f27-8d42-b39951cd17c8)

고수준 모듈이 저수준 모듈을 사용하면 앞서 계층 구조 아키텍처에서 언급했던 두 가지 문제, 구현 변경과 테스트가 어렵다는 문제가 발생한다.</br></br>

DIP는 이 문제를 해결하기 위해 저수준 모듈이 고수준 모듈에 의존하도록 바꾼다.</br>
=> 추상화한 인터페이스가 있다.</br>

```
public interface RuleDiscounter {
	public Money applyRules(Customer customer, List<OrderLine> orderLines);
}

```

```public class CalculateDiscountService {
	private CustomerRepository customerRepository;
	private RuleDiscounter ruleDiscounter;

	public Money calculateDiscount(OrderLine orderLines, String customerId) {
		Customer customer = customerRepository.findCusotmer(customerId);
		return ruleDiscounter.applyRules(customer, orderLines);
	}
}

```

CalculateDiscountService에는 Drools에 의존하는 코드가 없다.</br>
단지 RuleDiscounter가 룰을 적용한다는 사실만 안다.</br></br>

룰 적용을 구현한 클래스는 RuleDiscounter인터페이스를 상속받아 구현한다.</br>

```
public class DroolsRuleDiscounter implements RuleDiscounter{
	private KieContainer kContainer;

	@Override
	public void applyRules(Customer customer, List<OrderLine> orderLines) {
		...
	}
}

```

![33](https://github.com/JSON-loading-and-unloading/DDD-start/assets/106163272/97de1561-cab1-4ea0-8ecd-fc389b228a50)



위 구조를 보면 CalculateDiscountService는 더 이상 구현 기술인 Drools에 의존하지 않는다.</br>
' 룰을 이용한 할인 금액 계산'을 추상화한 RuleDiscounter 인터페이스에 의존할 뿐이다.</br>
'룰을 이용한 할인 금액 계산'은 고수준 모듈의 개념이므로 RuleDiscounter인터페이스는 고수준 모듈에 속한다.</br>
DroolsRuleDiscounter는 고수준의 하위 기능인 RuleDiscounter를 구현한 것이므로 저수준 모듈에 속한다.</br></br>

※고수준 모듈이 저수준 모듈을 사용하려면 고수준 모듈이 저수준 모듈에 의존해야 하는데, 반대로 저수준 모듈이 고수준 모듈에 의존한다고 해서 이를 DIP 의존 역전 원칙이라고 한다.</br>

```
RuleDisCounter ruleDiscounter = new DroolsRuleDiscounter();
CalculateDiscounterService disService = new CalculateDiscountService(ruleDiscounter);

```

구현 기술을 변경하더라도 Service를 수정할 필요가 없다.</br>

```
RuleDisCounter ruleDiscounter = new SimpleRuleDiscounter();
CalculateDiscounterService disService = new CalculateDiscountService(ruleDiscounter);

```

<h3>DIP 주의사항</h3>

DIP를 잘못 생각하면 단순히 인터페이스와 구현 클래스를 분리하는 정도로 받아들일 수 있다.</br>
DIP의 핵심은 고수준 모듈이 저수준 모듈에 의존하지 않도록 하기 위함인데 DIP를 적용한 결과 구조만 보고</br>
저수준 모듈에서 인터페이스를 추출하는 경우가 있다.</br>


![44](https://github.com/JSON-loading-and-unloading/DDD-start/assets/106163272/d3a57273-4836-429d-ae04-4533ad5865be)

RuleEngine 인터페이스는 고수준 모듈인 도메인 관점이 아니라 룰 엔진이라는 저수준 모듈관점에서 도출한 것이다.</br>


![55](https://github.com/JSON-loading-and-unloading/DDD-start/assets/106163272/b8047428-5c95-451f-8525-6e0158f09107)


<h3>DIP와 아키텍처</h3>

인프라스트척처 영역은 구현 기술을 다루는 저수준 모듈이고 응용영역과 도메인 영역은 고수준 모듈이다.</br>
인프라스트럭처 계층이 가장 하단에 위치하는 계층형 구조와 달리 아키텍처에 DIP를 적용하면 인프라스트럭처 영영이 응용 영역과 도메인 영역에 의존하는 구조가 된다.</br></br>


인프라스트럭처에 위치한 클래스가 도메인이나 응용 영역에 정의한 인터페이스를 상속받아 구현하는 구조가 되므로</br>
도메인과 응용 영역에 대한 영향을 주지 않거나 최소화하면서 구현 기술을 변경하는 것이 가능한다.</br></br>


![22](https://github.com/JSON-loading-and-unloading/DDD-start/assets/106163272/58d56761-71e0-41e8-ad32-9def6e9bf39a)

(JPA를 구현 기술로 사용하고 싶다면 JPA를 이용한 OrderRespository 구현 클래스를 인프라스트럭처 영역에 추가한다.)</br>


<h2>도메인 영역의 주요 구성요소</h2>


![구성](https://github.com/JSON-loading-and-unloading/DDD-start/assets/106163272/fbbce9c2-a701-44e9-9898-fdaeae48f83d)

<h3>엔티티와 밸류</h3>

도메인 모델의 엔티티와 DB 모델의 엔티티는 다르다.</br>

차이점 </br>
 1. 도메인 모델의 엔티티는 데이터와 함께 도메인 기능을 함께 제공한다.
    - 도메인 모델의 엔티티는 단순히 데이터를 담고 있는 데이터 구조라기보다는 데이터와 함께 기능을 제공하는 객체이다.
 2. 도메인 모델의 엔티티는 두 개 이상의 데이터가 개념적으로 하나인 경우 밸류 타입을 이용해서 표현할 수 있다.


<h3>애그리거트</h3>

관련 객체를 하나로 묶은 군집이다.</br>
주문 => 주문, 배송지 정보, 주문자, 주문 목록, 총 결제 금액의 하위 모델</br></br>

애그리거트를 사용하면 개별 객체가 아닌 관련 객체들을 묶어서 객체 군집 단위로 모델을 바라볼 수 있게 된다.</br>
![애그리](https://github.com/JSON-loading-and-unloading/DDD-start/assets/106163272/14c6979a-b0b7-4761-a550-a34ebc762228)


애그리거트를 사용하는 코드는 애그리거트 루트가 제공하는 기능을 실행하고 애그리거트 루트를 통해서 간접적으로 애그리거트 내의 다른 엔티티나 밸류 객체로 접근한다.</br>
=> 애그리거트 내부 구현을 숨겨서 애그리거트 단위로 구현을 캡슐화할 수 있도록 돕는다.</br>
(모든 접근은 에그리거트를 통해서 접근할 수 있다.)</br>


<h3>리포지터리</h3>

 - 엔티티나 밸류가 요구사항에서 도출되는 도메인 모델이라면 리포지터리는 구현을 위한 도메인 모델이다.
 - 리포지터리는 애그리거트 단위로 도메인 객체를 저장하고 조회하는 기능을 정의한다.

<img width="360" alt="flvh" src="https://github.com/JSON-loading-and-unloading/DDD-start/assets/106163272/1a01d5e1-809d-4b9d-a195-392491d5ed81">

응용 서비스는 의존 주입과 같은 방식을 사용해서 실제 리포지터리 구현 객체에 접근한다.</br>

응용 서비스와 리포지터리는 밀접한 연관이 있다.</br>
 - 응용 서비스는 필요한 도메인 객체를 구하거나 저장할 때 리포지터리를 사용한다.
 - 응용 서비스는 트랜잭션을 관리하는데, 트랜잭션 처리는 리포지터리 구현 기술의 영향을 받는다.
</br>
응용 서비스가 사용하는 기본 메서드</br>

 - 애그리거트를 저장하는 메서드
 - 애그리거트 루트 식별자로 애그리거트를 조회하는 메서드

<h2>요청 처리 흐름</h2>

 - 요청을 처음 받는 영역은 표현 영역이다.
 - 컨트롤러가 사용자의 요청을 받아 처리하게 된다.

표현 영역은 사용자가 전송한 데이터 형식치 올바른지 검사하고 문제가 없다면 데이터를 이용해서 응용 서비스에 기능 실행을 위임한다.
이때 표현 영역은 사용자가 전송한 데이터를 응용 서비스가 요구하는 형식으로 변환해서 전달한다.

응용 서비스는 도메인 모델을 이용해서 기능을 구현한다.
기능 구현에 필요한 도메인 객체를 리포지터리에서 가져와 실행하거나 신규 도메인 객체를 생성해서 리포지터리에 저장한다.

<h2>인프라스트럭처 개요</h2>

DIP에서 언급한 것처럼 도메인 영역과 응용 영역에서 인프라스트럭처의 기능을 직접 사용하는 것보다 이 두 영역에 정의한 인터페이스를 인프라스트럭처 영역에서 구현하는 것이 시스템을 더 유연하고 테스트하기 쉽게 만들어준다.

응용 서비스는 트랜잭션 처리를 위해 스프링이 제공하는 @Transactional을 사용해서 것이 편리하다.
영속성 처리를 위해 JPA를 사용할 경우 @Entitiy나 @Table과 같은 JPA 전용 애너테이션을 도메인 모델 클래스에 사용하는 것이 XML매핑 설정을 이용하는 것보다 편리하다.

=> 응용 영역과 도메인 영역이 인프라스트럭처에 대한 의존을 완전히 갖지 않도록 시도하는 것은 자칫 구현을 더 복잡하고 어렵게 만들 수 있다.

<h3>모듈 구성</h3>

패키지 구성 규칙에 정답은 없다.

1. 영역별로 별도 패키지로 구성한 모듈 구조
![1](https://github.com/JSON-loading-and-unloading/DDD-start/assets/106163272/fdbc183c-d67f-43be-9330-33fc38aef343)


2. 도메인이 크면 하위 도메인 별로 모듈을 나눔.
![2](https://github.com/JSON-loading-and-unloading/DDD-start/assets/106163272/1b05ae36-b5d6-46ac-9469-779cb79a934b)

하위 도메인을 하위 패키지로 구성한 모듈 구조
![3](https://github.com/JSON-loading-and-unloading/DDD-start/assets/106163272/59711b09-6d0c-4309-9034-bcdf3722fe89)
