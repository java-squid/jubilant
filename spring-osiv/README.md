# Open session in view (OSIV)?
- 말 그대로 해석해보면 세션을 view 레이어에도 열어둔다.
- 세션은 데이터베이스와 연결된 객체은 session을 의미하는 듯하고.
- 보통 이러한 세션은 트랜잭션과 생명주기를 같이 하는 걸로 알고 있는데, 이 부분을 넘어서 사용할 수 있도록 도와주는 기술을 말하는 듯.
- `LazyInitializationException` 와도 많이 연관이 있는 키워드 임.
    - 이 에러는 트랜잭션 범위를 넘어서, lazy field에 접근하려 하면 던져지는 에러 인듯.

# Spring / Hibernate..
- Spring에서는 `Hibernate.support` 패키지 아래에 이 기능을 도와주는 클래스가 있는듯.
- [OpenSessionInViewInterceptor](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/orm/hibernate5/support/OpenSessionInViewInterceptor.html)
    - 생성자에는 `sessionFactory` 가 들어감.
    - Spring web request interceptor that binds a Hibernate Session to the thread for the entire processing of the request.
    - 웹 request가 들어왔을 때, 매칭되는 스레드에 바인딩되는 session을 가지고 있는 interceptor class인듯.
- 어떻게 스프링에서 이용할 지 정리해보면,
    1. request 시작에, 스프링은 `OpenSessionInViewInterceptor` 생성. 이 요청에 매칭되는 interceptor 클래스를 만들어놓는듯.
    2. 어플리케이션에서, 세션이 필요해지는 특정 시점이 있다면 (트랜잭션 범위를 벗어난 Lazy loading), OpenSessionInViewInterceptor 에서 반환받은 세션을 통해 해결하는 듯
    3. request 시점 마지막에는, 사용되었던 세션을 모두 정리하는 과정이 있고.
- 그러니까 핵심은, 이름 처럼 view 단에서 (html...) 에서만 사용되는 것이 아닌 듯. 
- 트랜잭션 범위를 벗어나 접근하는 경우, 사용되게 되는듯.
![](https://3513843782-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-LxjHkZu4T9MzJ5fEMNe%2Fsync%2Fcb2a3996efdd4796447ebbcf545eec8838afd6f7.png?generation=1622890518451162&alt=media)


# On Developer Productivity
- osiv를 사용하면 좋은 점은 우선 편함 즉 `LazyInitializationException` 을 걱정하지 않아도 됨.
    - 또한 Session 의 생명주기를 관리해주기 때문에.
- osiv 가 켜져있는 상태에서 트랜잭션 범위 밖에서 lazy field를 조회한다면 아래와 같은 문제가 발생할 수 있음.
    1. exhausting the Connection Pool
        - 세션이 request 종료시까지 남아있을 수도 있기 때문에, 위와 같은 문제가 발생할 수 있음.
        - 더 심해지면, application 응답이 아예 안되는 문제도 있을 수 있음.
    2. unnecessary Queries
        - 불필요한 쿼리가 발생한다.
        - 즉 진짜 필요한 필드 이외에 다른 필드 (lazy) 접근만으로도 개발자가 생각하지 못한 쿼리가 실행될 수 있음.
        - 이러한 쿼리는 [auto-commit](https://docs.oracle.com/en/java/javase/11/docs/api/java.sql/java/sql/Connection.html#setAutoCommit(boolean)) 모드로 실행됨.

- 그리고 osiv가 꺼져있다면, 트랜잭션 범위 안에서, 필요한 필드 (lazy field) 를 initialize 해줘야함.
    - 그 과정에서 `Hibernate.initialize()` 를  이용하는듯.


# Alternatives
- osiv를 이용하는 거 말고, 대안은?

## Entity Graphs
- `@EntityGraph` 라는 어노테이션에, 미리 fetch하고 싶은 속성을 기재함으로써, 쿼리 레벨에서 필요한 필드에 대한 정보를 모두 가져오도록 하는 방법

## Hibernate.initialize()
- 위에서도 언급했지만, 조회 후에, 해당 엔티티의 Lazy field (추후에 필요한 필드) 에 hibernate 에서 제공하는 메서드를 이용. Lazy 한 쿼리를 발생 시켜서 필요한 필드만 조회하도록 하는 방법.
- 이 방법은 추가적인 쿼리(최소한 한 개 이상)가 발생된다는 단점이 있음.

## Fetch Joins
- 하나의 쿼리에, 필요한 정보를 모두 가져오게 함.
- Entity graph와 거의 동일한 목적으로 보이지만, 아마 방법론에서 차이가 있을듯. 
    - 두 개의 방법에 어떤 차이가 있는지 알아보면 좋을듯.
    - [Spring Data JPA and Named Entity Graphs](https://www.baeldung.com/spring-data-jpa-named-entity-graphs),[JPA Join Types](https://www.baeldung.com/jpa-join-types) 


# Conclusion

# Reference
- https://www.baeldung.com/spring-open-session-in-view
- https://www.baeldung.com/jpa-hibernate-persistence-context
- https://stackoverflow.com/questions/30549489/what-is-this-spring-jpa-open-in-view-true-property-in-spring-boot
- https://kingbbode.tistory.com/27