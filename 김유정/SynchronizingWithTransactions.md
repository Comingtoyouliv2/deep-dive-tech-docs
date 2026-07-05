# 트랜잭션으로 리소스 동기화하기
다양한 트랜잭션 매니저를 만드는 방법과 트랜잭션에 동기화가 필요한 리소스들
(예를 들어 JDBC DataSource 를 위한 DataSourceTransactionManager, Hibernate SessionFactory 를 위한 HibernateTransactionManager)
이 어떻게 연결되는지 이제는 명확하게 해야합니다.
이 섹션에서는 애플리케이션 코드가(JDBC, Hibernate 또는 JPA 와 같은 persistence API를 직접적 또는 간접적으로 사용하면서)
이러한 리소스들을 만들고, 재사용하고 적절하게 정리하는 것을 보장하는 방법에 대해 설명합니다.
또한, 이 섹션은 트랜잭션 동기화가 관련된 `TransactionManager`를 통해 선택적으로 발동되는 방법에 대해서도 설명합니다.

## High-level Synchronization Approach
템플릿 기반의 persistence integration API 를 사용하거나 native 리소스 팩토리를 관리하기 위한 트랜잭션-감지 팩토리 빈 또는 프록시와 함께 native ORM API를
사용하는게 선호되는 접근이다. 이 트랜잭션-감지 솔루션은 내부적으로 자원 생성과 재사용, 정리, 리소스의 선택적 트랜잭션 동기화를 내부적으로 다룬다.
그러므로 유저는 이 문제를 다룰 필요가 없고, boilerplate 가 아닌 영속성 로직에 집중할 수 있다.
일반적으로 native ORM API 또는 JdbcTemplate을 통해 JDBC에 접근하는 템플릿을 사용한다.
이 솔루션들은 이후 섹션에서 자세하게 다룬다.

## Low-level Synchronization Approach
`DataSourceUtils`(for JDBC), `EntityManagerFactoryUtils`(for JPA), `SessionFactoryUtils`(for Hibernate) 같은 클래스 등.
native persistence API 의 리소스 타입을 직접적으로 다루고 싶을 때, 이러한 클래스들을 사용하면
스프링 프레임워크에서 관리하는 적절한 인스턴스를 얻을 수 있고, 트랜잭션이 동기화되며(선택적으로) 프로세스에서 발생하는 예외가 일관된 API로 적절하게 매핑되도록 할 수 있다.

예시로, JDBC의 경우, DataSource 에서 `getConnection()` 메서드를 호출하는 전통적인 JDBC 대신
`org.springframework.jdbc.datasource.DataSourceUrils` 클래스를 다음과 같이 사용할 수 있다.
```java
Connection conn = DataSourceUtils.getConnection(datasource);
```
이미 동기화되어있는 커넥션을 가진 트랜잭션이 존재한다면, 그 인스턴스를 반환할 것이다. 
그렇지 않다면, 메서드는 새로운 커넥션을 생성할 것이고, 이는 기존 트랜잭션에 동기화하고(선택적으로) 이후 해당 트랜잭션 내에서 재사용이 가능하게 된다.
앞서 언급한 것처럼, `SQLException`은 스프링의 unchecked `DataAccessException` 타입의 계층 중 하나인 `CannotGetJdbcConnectionException` 에서 래핑된다.
(-> 개발자가 일일히 catch 해주지 않아도 됨). 이러한 접근은 `SQLException`보다 풍부한 예외 정보를 제공했고, 다른 데이터베이스 심지어 다른 영속성 기술들로 전환되는 걸 보장했다.(기술이 바뀌여도 예외 코드는 변하지 않기 때문)
```java
//JDBC 직접 쓸 때 (Spring 없이)
try {
    // ...
} catch (SQLException e) {
    if (e.getErrorCode() == 1062) { // MySQL의 duplicate key 에러코드
        // 중복 처리
    }
}
```
물론, 한번 Spring JDBC supprt, JPA support 또는 Hibernate support 를 사용하기 시작하면, 일반적으로는 `DataSourceUtils` 나 다른 헬퍼 클래스를 사용하지 않는 걸 선호할 것이다.
관련 API 를 직접 다루는 것보다는 Spring 추상화를 통해 작업하는 것이 더 편리하기 때문이다. 
예를 들어, JDBC를 단순하게 사용하기 위해 `JdbcTemplate` 또는 `jdbc.object` 를 사용한다면, 특별히 코드를 작성하지 않더라도 커넥션 획득히 알아서 처리된다.



## 새롭게 알게 된 표현
* address: 다루다. 대응하다(handle 은 처리하고 해결하는 결과까지 포함. treat 은 취급하다 에 가까움)
* consequent: ~의 결과로 일어나는
* subsequent: 그 다음의, 하추의