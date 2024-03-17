

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
