# Hibernate Architecture
> 들어가면서..
- 현재 Spring 5, Hibernate 5, HikariCP 사용 중.
- 목표는? database-routing (main <-> repl)
    - 검색해보면?
    - https://www.baeldung.com/spring-abstract-routing-data-source 와 같은 방법을 찾아볼 수도 있음.
- 그렇지만 현재 환경에서 안될 수도?
    - 왜? 
    - Dao Layer 에서 SessionFactory를 직접 주입 받아서 하고 있기 때문에
    - 위 포스팅은 `Datasource` 레벨에서의 스위칭 인듯
- 그러면.. 만약에 A- TransactinoManager가 진행 중인 트랜잭션 중에 B- SessionFactory로 switch 한다면, 어떻게 될까?
    - 에러? 성공?..


## Architecture 
![](https://access.redhat.com/webassets/avalon/d/Red_Hat_JBoss_Web_Server-3-Hibernate_Core_Reference_Guide-en-US/images/c3ee197b49364a876bc5867d5c2c6db7/1091.png)


### SessionFactory
- Session 을 생성하는 공장, (그만큼 이 객체를 생성하는 데 많은 자원이 소모되는 듯.)
- **Thread-safe**
    - 여러 스레드에서 접근해도, 다른 세션을 건네줄듯.
- 하나의 데이터베이스와 매칭되는 듯.

### Session
- *하나의 스레드에 대응*되고, 짦은 생명 주기를 가졌으며, persistant layer와 application layer와의 소통을 의미하는 객체
- JDBC connection을 감싸고 있고 있음.
- **Not thread-safe**
- Transaction

### Transaction
> A single-threaded, short-lived object used by the application to specify atomic units of work. 

- atomic한 작업을 지정하는 데 사용되는 single threaded, short-lived한 객체
- JDBC, JTA or CORBA transaction 로 부터 추상화되어있음.
- 하나의 Session은 여러개에 Transaction에 걸쳐있을 수 있음.
 
### Reference
- https://www.decodejava.com/hibernate-architecture.html
- https://www.geeksforgeeks.org/hibernate-architecture/
- https://access.redhat.com/documentation/en-us/red_hat_jboss_web_server/3/html/hibernate_core_reference_guide/chap-hibernate_architecture


## HibernateTransactionManager
- 하나의 하이버네티트 세션 팩토리를 위한 PlatformTransactionManager 구현체
- `AbstractPlatformTransactionManager` 추상 클래스도 상속했음.

- 특정 factory에서 thread에  하이버네이트 세션을 바인딩 하는건, 잠재적으로 하나의 세션에 하나의 스레드라는 걸 허용하는거나 다름없는듯.
- `SessionFactory.getCurrentSession()` 는 요구됨. 트랜잭션 작업을 도와주는 하이버네이트 접근 코드에, 그리고 SpingSessionContext에 SesstionFactory가 정의되어야 있어아하고, 
- 커스텀 isolation level을 지원하고, timoeout option도 설정할 수 있도록 해주는 듯.
- 이 트랜잭션 매니저는 (HibernateTransactionManage) 는 트랜잭션 데이터 접근에, 하나의 하이버네이트 세션 팩토리를 이용하는 어플리케이션에 적합함. 

> 정리하자면,
- 즉 SessionFactory를 가지고 Session을 핸들링하면서, 원자적인 작업 단위인 Transaction을 관리하기 위한 클래스.
- 가능한 하나의 세션을 하나의 스레드에서 사용하게 하려 하는듯 (느낌상)


### LifeCycle

1. `doGetTransaction()`
- 현재 트랜잭션 상태에 대한 트랜잭션 개체를 반환
- 반환된 개체에는 트랜잭션 관리자에서 현재 getTransaction 호출 전에 이미 시작된 트랜잭션에 대한 정보가 포함되어 있음.
    - 즉 이 전에 시작한 트랜잭션 객체 가 있으면 그걸 반환할듯.
    - 반환되는 객체는 `SessionHolder`

2. `doBegin()`
> This method gets called when the transaction manager has decided to actually start a new transaction. Either there wasn't any transaction before, or the previous transaction has been suspended.

- 이전의 트랜잭션 객체가 없었을 경우에만 호출됨. 즉 진짜 새로운 트랜잭션일 때만
- 혹은 이전 트랜잭션이 중단되었을 경우에 호출되기도함.

3. `doSuspend()`
- 현재 트랜잭션을 중단.

4. `doResume()`
- 현재 트랜잭션을 재시작.


### Reference
- https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/orm/hibernate5/HibernateTransactionManager.html
- https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/transaction/support/AbstractPlatformTransactionManager.html

## SessionFactory
- `Session getCurrentSession()`
- Obtains the current session. The definition of what exactly "current" means controlled by the org.hibernate.context.spi.CurrentSessionContext impl configured for use.
    - current session은 CurrentSessionContext에서 관리되는 세션을 의미.
    - 해당 인터페이스 구현체 중에, SpringSessionContext 가 있음.
    - 아마도 스프링에서 세션은 이 구현체를 통해 관리되는 듯.

## SpringSessionContext
- `Session getCurrentSession()`
- Retrieve the Spring-managed Session for the current thread, if any.  
    - 현재 스레드에서 스프링이 관리하고 있는 세션이 있다면, 반환하는 메서드.
    - 스프링이 어떻게 관리하고 있는지는.. `TransactionSynchronizationManager.getResource()`에서 판단되는 듯.


## TransactionSynchronizationManager

> Central delegate that manages resources and transaction synchronizations per thread. To be used by resource management code but not by typical application code.

- 현재 진행되는 트랜잭션을 관리하는 매니저 인듯
- 스레드 별로 있는 듯.

### Reference
- https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/transaction/support/TransactionSynchronizationManager.html