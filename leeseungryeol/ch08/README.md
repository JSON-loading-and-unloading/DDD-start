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
