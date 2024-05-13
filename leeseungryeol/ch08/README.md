<h1>애그리거트 트랜잭션 관리</h1>

<h2>애그리거트와 트랜잭션</h2>

![download](https://github.com/JSON-loading-and-unloading/DDD-start/assets/106163272/d7ac3885-5ecd-45af-be65-3eec3373e46b)

각 스레드는 독립적으로 실행되고,  이 이미지 순서의 문제점은 운영자는 기존 배송지 정보를 이용해서 배송 상태로 변경했는데 그 사이 고객은 배송지 정보를 변경했다.</br>
이를 해결하기 위해 트랜잭션 처리 기법을 사용한다.</br>
선점, 비선점으로 나뉜다.</br>


<h2>선점 잠금</h2>

선점 잠금은 먼저 애그리거트를 구한 스레드가 애그리거트 사용이 끝날 때까지 다른 스레드가 해당 애그리거트를 수정하지 못하게 막는 방식이다.</br>
![images](https://github.com/JSON-loading-and-unloading/DDD-start/assets/106163272/344e696a-7826-4aa8-ad97-977aeaf7a55a)

위 이미지 처럼 운영자 스레드가 들어가 일을 할 때, 잠금을 걸어 고객 스레드는 일을 못하고 대기하고 운영자 스레드가 잠금을 해제해야지 고객 스레드가 일을 한다.</br>

```
  public interface MemberRepository extends Repository<Member, MemberId> {
    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("select m from Member m where m.id = :id")
    함수
```

jpa에서는 위와 같은 Lock 어노테이션을 사용하여 </br>
해당 엔티티와 매핑된 테이블을 이용해서 선점 잠금 방식을 적용할 수 있다.</br>

<h3>선점 잠금과 교착 상태</h3>
</br>
하나의 스레드가 일을 완료하지 못하고 잠금을 계속 진행한다면 다른 스레드는 일을 처리하지 못하고 교착 상태가 된다.</br>
상대적으로 사용자 수가 많을 때 발생할 가능성이 높고 사용자 수가 많아지면 교착 상태가 빠지는 스레드는 더 빠르게 증가한다.</br>

```
  public interface MemberRepository extends Repository<Member, MemberId> {
    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @QueryHints({ @QueryHint(name="javax.persistence.lock.timeout", value=2000)})
    @Query("select m from Member m where m.id = :id")
    함수
```

jpa 어노테이션을 활용하여 타임아웃을 걸어 오래걸릴 경우 알아서 빠져나올 수 있도록 한다.</br>


<h2>비선점 잠금</h2>
![image](https://github.com/JSON-loading-and-unloading/DDD-start/assets/106163272/b7572b03-17ad-4da9-ab77-0008b1abfe2b)

위 이미지에서는 운영자가 배송지 정보를 조회하고 배송 상태로 변경하는 사이에 고객이 배송지를 변경하는 것이 문제다.</br></br>


비선점 잠금은 동시에 접근하는 것을 막는 대신 변경한 데이터를 실제 DBMS에 반영하는 시점에 변경 가능 여부를 확인하는 방식이다.</br>

※비선점 잠금을 구현혀려면 애그리거트에 버전으로 사용할 숫자 타입 프로퍼티를 추가한다.</br>

```
  UPDATE aggtable SET version = version + 1, colx = ?, coly = ?
  WHERE aggid = ? and version = 현재 버젼
```

위와 같이 수정 쿼리를 날려 수정할 애그리거트와 매핑되는 테이블의 버전이 같다면 통과, 아니라면 에러를 발생시킨다.</br></br>


❗JPA는 버전을 이용한 비선점 잠금 기능을 지원한다.</br>

```
  @Entity
  @Table(name = "purchage_order")
  @Access(AccessType.FIELD)
  public class Order {
  	@EmbeddedId
  	private OrderNo number;
  
  	@Version
  	private long version;
  	
  	...
  }
```

<h3>강제버전증가</h3>

루트 엔티티가 아닌 다른 엔티티의 값만 변경될 시, JPA는 루트 엔티티의 버전 값을 증가시키지 않는다.</br>
이는 애그리거트 관점에서 문제가 되고, 애그리거트 구성요소 중 일부 값이 바뀌면 논리적으로 애그리거트는 바뀐 것이다.</br>
따라서 루트 애그리거트 버전도 바껴야한다.</br>

```
  @Repository
  public class JpaOrderRepository implements OrderRepository {
  	@PersistenceContext
  	private EntityMangager entityManager;
  
  	@Override
  	public Order findbyIdOptimisticLockMode(OrderNo id) {
  		return entityManager.find(Order.class, id
  				LockModeType.OPTIMISTTIC_FORCE_INCREMENT);
  	}
```

jpa는 EntityManager#find() 메서드로 엔티티를 구할 때 강제로 버전 값을 증가시키는 잠금 모드를 지원한다.</br>
LockModeType.OPTIMISTTIC_FORCE_INCREMENT를 사용하면 해당 엔티티의 상태가 변경되었는지에 상관없이 트랜잭션 종료 시점에 버전 값 증가 처리를 한다.</br>
