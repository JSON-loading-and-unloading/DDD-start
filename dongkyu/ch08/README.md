# Chapter8 애그리거트 트랜잭션 관리

## 8.1 애그리거트와 트랜잭션

- 한 애그리거트를 두 사용자가 동시에 접근하게 되면 애그리거트 일관성이 깨질 수도 있다.
- 일관성을 지키기 위해 다음 두 가지 중 한 방법을 선택해야 한다.
    - 한 사용자가 상태 변경을 하는 동안 다른 사용자의 접근을 막는다.
    - 일관성이 깨진 후 한 사용자에게 다시 상태를 수정하도록 한다.

## 8.2 선점 잠금

- **선점 잠금**(Pessimistic Lock)은 먼저 애그리거트를 구한 스레드가 다른 스레드의 애그리거트 접근을 막는 방식이다.
    - 늦게 접근한 스레드는 먼저 온 스레드가 잠금을 해제할 때까지 블로킹된다.
- 한 스레드가 수정하는 동안 다른 스레드가 수정할 수 없으므로 데이터 충돌 문제를 해소할 수 있다.
- 선점 잠금은 보통 DBMS가 제공하는 행 단위 잠금을 사용해서 구현한다.
- 스프링 데이터 JPA는 @Lock 애너테이션을 사용해 잠금 모드를 지정한다.

    ```java
    @Lock(LockMode.Type.PESSIMISTIC_WRITE)
    @Query("select m from Member m where m.id = :id")
    Optional<Member> findByIdForUpdate(@Param("id") MemberId memberId);
    ```


### 8.2.1 선점 잠금과 교착 상태

- 선점 잠금을 사용할 때는 잠금 순서에 따른 교착 상태(deadlock)가 발생하지 않도록 주의해야 한다.
- 선점 잠금에 의한 교착 상태는 사용자 수가 많을 때 발생할 가능성이 높다.
- 교착 상태를 해결하려면 잠금을 구할 때 최대 대기 시간을 지정해야 한다.
- 스프링 데이터 JPA는 `@QueryHints` 애너테이션을 통해 쿼리 힌트를 지정할 수 있다.

    ```java
    @Lock(LockMode.Type.PESSIMISTIC_WRITE)
    @QueryHints({
    	@QueryHint(name = "javax.persistence.lock.timeout", value = "2000)
    })
    @Query("select m from Member m where m.id = :id")
    Optional<Member> findByIdForUpdate(@Param("id") MemberId memberId);
    ```

- DBMS마다 교착 상태인 커넥션을 처리하는 방식이 다르기에 DBMS에 따라 JPA가 어떤 식으로 대기 시간을 처리하는지 반드시 확인해야 한다.

## 8.3 비선점 잠금

- 선점 잠금만으로 모든 트랜랜잭션 충돌 문제가 해결되는 것은 아니다.
- **비선점 잠금**(Optimistic Lock)은 동시에 접근하는 것을 막는 대신 변경한 데이터를 DBMS에 반영하는 시점에 변경 가능 여부를 확인하는 방식이다.
- 비선점 잠금을 구현하려면 애그리거트에 버전으로 사용할 숫자 타입 프로퍼티를 추가해야 한다.
    - 애그리거트를 수정할 때마다 버전 프로프티 값이 1씩 증가하도록 구현한다.

    ```sql
    UPDATE aggtable SET version = version + 1, colx = ?, coly = ?
    WHERE aggid = ? and version = 현재_버전
    ```

    - 쿼리를 수정할 때 버전 값이 현재 애그리거트 버전과 동일한 경우에만 데이터를 수정한다.
    - 버전이 다르면 수정에 실패하게 된다.
- JPA에서 비선점 잠금을 이용하려면 `@Version`을 사용할 수 있다.

    ```java
    @Entity
    public class Order {
    	// ...
    	@Version
    	private Long version;
    	// ...
    }
    ```

    - JPA는 엔티티가 변경될 때 `@Version`에 명시한 필드를 이용해 비선점 잠금 쿼리를 실행한다.
- 응용 서비스는 버전에 대해 알 필요는 없다.
    - JPA가 버전을 통해 알아서 해준다.
- 비선점 잠금 쿼리 실행 후 수정된 행 개수가 0이면 트랜잭션 충돌이 일어난 것이므로 **트랜잭션 종료 시점에 예외가 발생한다.**
    - `OptimisticLockingFailureException`이 발생

### 8.3.1 강제 버전 증가

- 애그리거트에 루트 엔티티 외에 다른 엔티티가 존재하는데 기능 실행 도중 루트가 아닌 다른 엔티티 값만 변경된다면 루트 엔티티의 버전이 갱신되지 않는다.
- 위 현상이 도메인 일관성을 깨뜨린다면 강제로 버전 값을 증가시킬 수도 있다.

    ```java
    @Lock(LockModeType.OPTIMISTIC_FORCE_INCREMENT)
    @Query("select m from Member m where m.id = :id")
    Optional<Member> findByIdForUpdate(@Param("id") MemberId memberId);
    ```

    - `LockModeType.OPTIMISTIC_FORCE_INCREMENT`를 사용하면 엔티티가 변경되었는지에 상관 없이 버전 값 증가 처리를 한다.

## 8.4 오프라인 선점 잠금

- **오프라인 선점 잠금**(Offline Pessimistic Lock)은 한 사용자가 데이터를 수정하는 도중에 다른 사용자의 접근 자체를 사용자 인터페이스 수준에서 막는 방식이다.
    - 아틀라시안의 컨플루언스는 문서를 편집할 때 누군가 먼저 편집을 하는 중이면 다른 사용자가 문서를 수정하고 있다는 안내 문구를 보여준다.
- 단일 트랜잭션에서 동시 변경을 막는 선점 잠금과 달리 오프라인 선점 잠금은 여러 트랜잭션에 걸쳐 동시 변경을 막는다.
    - 첫 번째 트랜잭션에서 오프라인 잠금 선점하고 마지막 트랜잭션에서 잠금을 해제한다.
    - ex) 수정 폼을 요청할 때부터 오프라인 잠금을 획득한 뒤 수정 처리과 완료되면 잠금을 해제
    - 잠금 해제에 실패해 영원히 사용자가 기다리지 않도록 잠금 유효 시간을 가져야 한다.
    - 사용자가 수정하는 도중에는 잠금이 해제되지 않도록 일정 주기로 유효 시간을 증가시키는 방식이 필요하다.
  
