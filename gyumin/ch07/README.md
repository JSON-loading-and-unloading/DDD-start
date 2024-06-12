# 7장 도메인 서비스

## 7.1 여러 애그리거트가 필요한 기능

* 한 애그리거트에 넣기 애매한 도메인 기능을 억지로 특정 애그리거트에 구현하면 안됩니다.
* 억지로 구현하면 애그리거트는 자신의 책임 범위를 넘어서는 기능을 구현하기 때문에 과한 의존성과 수정하기 어려운 코드를 만들기 떄문입니다.

## 7.2 도메인 서비스

* 도메인 서비스는 다음과 같은 상황에서 사용됩니다.
  * 계산 로직 : 여러 애그리거트가 필요한 계산 로직이나, 한 애그리거트에 넣기에는 다소 복잡한 계산 로직
  * 외부 시스템 연동이 필요한 도메인 로직 : 구현하기 위해 타 시스템을 사용해야 하는 도메인 로직
* 애그리거트 객체에 도메인 서비스를 전달하는 것은 응용 서비스 책임입니다.
* 애그리거트 메서드를 실행할 때 도메인 서비스를 인자로 전달하지 않고 반대로 도메인 서비스의 기능을 실행할 때 애그리거트를 전달하기도 한다.
* 도메인 서비스는 도메인 로직을 표현하므로 도메인 서비스의 위치는 다른 도메인 구성요소와 동일한 패키지에 위치한다.
* 도메인 서비스의 로직이 고정되어 있지 않은 경우 도메인 서비스 자체를 인터페이스를 구현하고 이를 인프라계층에서 구현할 수 있다.