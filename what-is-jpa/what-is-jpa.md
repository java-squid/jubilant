
# JPA
- Java persistence api, orm
- 즉 어떤 기능을 동작하는게 아니라 인터페이스.
- 디비에 쿼리를 날릴수있는 구현체들을 정의한 인터페이스
- 구현하는건 하이버네이트 등이 있음
- 어떤 data access object라도  jdbc를 이용하는건데 게발자는 보일러플레이트 코드를 작성하지 않아도 됨
- 예를들어 jpa 구현체인 하이버네이트를 통해서 jdbc 쿼리를 sql종속적이 아닌 자바 컨셉인 객체지향적으로 객체에 맞춰서 작성할수 있음 (jpql)
- 그럼 sql작성은?
- 하이버네이트가 자동으로 만들어준다. 이에 따른 장단점은?
- 일단 개발자가 객체지향적으로 코드를 짤수있도록 돕는다.

## JPQL
- JPA 엔티티에 대한 쿼리작성 문법 - SQL이랑 비슷
- 예를 들어 select * from member 이런거도 select * from Member 같이 from 절에 객체가 들어간다
- 엄청 신기하다 [select new memberDTO 이런것도 있다](https://doing7.tistory.com/133#:~:text=%EC%A0%84%EC%B2%B4%20%ED%81%B4%EB%9E%98%EC%8A%A4%EB%AA%85%20%EC%9E%85%EB%A0%A5-,select%20new%20%ED%8C%A8%ED%82%A4%EC%A7%80%EB%AA%85.memberDTO(m.username%2C%20m.age)%20from%20Member%20as%20m,-%F0%9F%8C%B1%C2%A0%EC%97%94%ED%8B%B0%ED%8B%B0%20%ED%95%84%EB%93%9C%EC%9D%98%20%EB%91%90%EA%B0%80%EC%A7%80)

## JPA 영속성 컨텍스트란?
- 디비와 어플리케이션 사이에 객체를 관리하는 논리적 공간 (컨텍스트)
- 영속성컨텍스트가 왜있는걸까? 이런 공간을 만든 이유가 있을텐데?
### 이유1: 1차캐시
  - 똑같은거 select할때 영속성 컨텍스트에 결과를 캐싱해놓으면 다음에 찾을때 디비와 커넥할필요가 없다
  - 1차 캐시와 2차 캐시의 차이?
      - 1차캐시는 스레드하나에서만 쓴다. 해당 스레드가 종료되면 1차캐시도 사라진다. 스레드 100개면 1차캐시도 100개생김. 트랜잭션 범위에서만 사용가능한것
      - 트랜잭션을 어플리케이션에서 차원에서 제공을 하는 것?
      - 2차캐시는 글로벌하게 사용가능

#### 1차 캐시와 2차 캐시의 구현 방법을 다르게 처리한 이유
- 1차 캐시 트랜잭션의 시작과 종료의 term일때만 유효
- 2차 캐시는 글로벌

스레드마다 영속성 컨텍스트 새로 생성된다, 즉 1차 캐시가 새로 생성된다.   
때문에, 서로 다른 영속성 컨텍스트에서 똑같은 member를 select할 때,   
항상 네트워크를 통해 디비를 조회하는 것은 비효율적.   
이를 해결하려면 글로벌적인 캐시가 있으면 해결됨.
그것이 2차 캐시!

참고: https://willseungh0.tistory.com/77
### 이유2: 동일성보장
  - 한번 select한 객체가 캐싱되어있기 때문에 똑같은 객체라 해당 객체에 대한 동일성 보장이 가능. 만약에 똑같은 row라도 따로 select 두번했으면 다른 객체가 만들어질꺼니까
### 이유3: 트랜잭션 지원 쓰기지연
  - 영속성 컨텍스트 같은게 없으면 insert할때 디비와 커넥이 많아짐 -> 노효율성
  - 영속성 컨텍스트에 insert할걸 쌓아두고 commit한다 -> 한번의 커넥으로 해결
### 이유4: 더티 체킹 (변경감지)
  - flush 했을때 캐싱되어있는 객체의 원본 스냅샷과 비교하여 변경있으면 update쿼리 생성하고 flush함.
  - 근데 이게 왜 dirty라는 이름?
  - > DynamicUpdate.  
    > 더티체킹은 객체 스냅샷과 비교해 변경감지하여 업데이트.  
    > 이때 기본젓으로 업데이트는 객체의 모든 필드를 업데이트.  
    > DynamicUpdate을 쓰면 내가 바꿔준 필드만 업데이트 가능.  

### 영속성 컨텍스트를 거치지 않는 액션

#### 벌크연산
- 영속성 컨텍스트와 2차캐시를 무시하고 바로 디비로 직접 쿼리
- 영속성 컨텍스트의 내용과 달라질수있음
  - em.refresh() 로 동기화
  - 영속성 컨텍스트를 사용하기전 벌크연산을 수행하면 안달라진
- 원래 jpa로 업데이트하려면 특정 member를 select해야한다
  - 벌크연산하려면 List<Member> 를 받는 쿼리를 호출할거다
  - 그다음 업데이트하기위해 List<Member>를 foreach로 객체를 다 변경해줬다
  - 더티체킹으로 List.length만큼 업데이트발생
  - 때문에 따로 벌크연산 메소드로 삽입한다
출처 https://dev-gorany.tistory.com/m/327

#### 수동액션
-  Native SQL..?
   - 어떤 이유로 JPQL로 객체질의를 할수없을때
   - 어떤 이유란 특정 데이터베이스 종속적인 쿼리 문법
 - JDBC 를 직접 사용해는 것을 말하는 것으로 이해

## N + 1 문제
- select 하나 요청을 의도했는데 where xxx_id = 테이블내의 레코드 갯수만큼 다수의 select를 하는 문제를 말함
- 예를 들어 post와 post_comment가 있다고 치자.
    - post 1개와 해당 포스트에 post_comment 5개가 있다.
    - 이때 post_comment 와 관계된 post를 불러오고 싶다
    - `select * from post_comment`
    - `select * from post where id = post_comment.id` -> post_comment의 갯수(N)만큼 select 함
    - 보통 이를 select 하려면 join을 사용할것이다
    - `select * from post_comment join post on post_comment.id = post.id` -> 1
    - 즉 원래 1개로 해결가능한 select인데 추가적으로 N개의 select이 발생해서 N + 1 문제라고 함
- manytoone에서 이문제가 발생
    - onetoone 에서도 발생한다는데 생각안해봄
- 하이버네이트를 사용할때 아무것도 명시하지않으면 fetchType.EAGER 로 설정되는데 n+1로 불러온다.
- 왜그러냐 하면 post_comment 리스트가 호출되기전에 post의 association가 먼저 호출되어져야하기 때문이라고 한다.
    - 이거 이해못함
    - 원문: [Notice the additional SELECT statements that are executed **because the post association has to be fetched prior to returning the List of PostComment entities.**](https://stackoverflow.com/questions/97197/what-is-the-n1-selects-problem-in-orm-object-relational-mapping/39696775#:~:text=WHERE%20p.id%20%3D%204-,Notice%20the%20additional%20SELECT%20statements%20that%20are%20executed%20because%20the%20post%20association%20has%20to%20be%20fetched%20prior%20to%20returning%20the%20List%20of%20PostComment%20entities.,-Unlike%20the%20default)
    - onetoone에서 발생하는 이유 by jypthemiracle: https://github.com/java-squid/2022-jubilant/pull/20/files#r846593587
- post association이 필요없다면 lazy로 변경해줘야한다 
- post association이 필여하다면 join fetch를 사용해야한다
- 근데 lazy로 명시해줘도 n +1 문제가 발생한다
    - 문제 원인과 해결법 제대로 파악못함
    
벌크연산 주의점
- 영속성 컨텍스트를 거치지않고 디비에 직접 쿼리하기때문에 데이터 무결성이 깨짐
- 예를들어 길동이 member 하나 select 했다
  - 영속성 컨텍스트에 길동이 member 존재, 길동이 age=45
  - jpql 벌크연산으로 set member.age = 1
  - 길동이 age는 그대로임
  - why?
  - 벌크연산 업데이트시 해당 member가 이미 영속성 컨텍스트에 있으면 디비에서 조회한 신규 데이터를 버리기 때문

해결책
- 1. select하기전에 벌크연산 먼저 수행한다
  - 영속성 컨텍스트에 기존데이터 없으니까 벌크연산 수행후에 영속성컨텍스트에 member list가 있을거니까
- 2. 벌크 연산 수행후 영속성 컨텍스트 초기화
  - 이후에 select 하게되면 비어있으므로 1차캐시가 없으니까 디비를 조회하게 됨
- 3. em.refresh(길동이) 길동이만 리프레쉬
- 4. 제일 좋은 것은 벌크연산과 그 외의 것을 분리하는 것입니다. CQRS를 생각해보면 그렇습니다. 만약 섞이는 경우가 있다면 명시적인 flush()가 필요합니다. 출처: dan (CQRS란?)

출처 https://velog.io/@cksdnr066/TIL-JPA-%EB%B2%8C%ED%81%AC-%EC%97%B0%EC%82%B0-%EC%82%AC%EC%9A%A9%EC%8B%9C-%EC%A3%BC%EC%9D%98%EC%A0%90

## Spring Data JPA
- JPA를 더 쉽게 사용하게 해줌
    - 원래 뭐가 불편했었나?
        - 예상: EntityManager 사용이 불편?
        - 이를 `Repository` 인터페이스가 해결?
        - 불편해서 나왔다기 보단 Spring 패밀리에서 뭔가 더쉽게 상호작용할수있도록 개발을 돕는다?
- 공식 레퍼런스보면서 느낀점은 _"아, CRUD 뿐만아니라 다양한 쿼리들을 직관적인 메소드 이름으로 간단하게 호출할 수 있구나. 그게 중심 기능이구나."_
- JPA가 기능적으로 영속성 컨텍스트 등을 통해 객체 관계 매핑 을 쉽게하여 쿼리를 자동으로 만들어준다면,
- Spring Data JPA는 거기에서 더 나아가 아예 [트랜잭션/세션 관리](https://onecellboy.tistory.com/349#:~:text=hibernate%20%EB%A5%BC%20%ED%86%B5%ED%95%9C%20%EC%BF%BC%EB%A6%AC%20%EC%8B%A4%ED%96%89)를 자동(?) 혹은 어노테이션기반(?) 으로 Repository interface를 자동으로 구현한 메소드만 호출하면 되도록 더 쉽게 만들었다?
- 생각해보니 [HQL 같이 JPA 구현체들의 쿼리를 쓴다는 것](https://null486.tistory.com/17)도 이것도 번거로울 수도 있겠구나 하는 생각이 들긴 함.
- Spring Data JPA 에선 걍 findAll() 호출하면 끝이니까.


## ORM vs SQL Mapper

- 근데 아직까진 잘모르곘다. 난 마이바티스 써보니까 좋던데?
- JPA에서 동적쿼리를 어떻게 사용하는진 모르겠지만 마이바티스에서는 
    직관적임
    - 그냥 쿼리 작성하고 안에 if-else 문 쓰면 되니까
      - [그렇게 쿼리에 `로직`이 쌓이게 된다.](https://github.com/java-squid/2022-jubilant/pull/20/files#r845704905)
    - Spring Data JPA에서 사용하려면 JPQL을 사용하던가 QueryDSL을 사용해야한다
    - > **QueryDSL?**    
    [자바코드로 쿼리 작성하게 해줌](https://doing7.tistory.com/123#:~:text=%F0%9F%8C%B1-,JPQL%20vs%20Querydsl%20%EB%B9%84%EA%B5%90,-%EC%BF%BC%EB%A6%AC%EB%A5%BC%20%EC%9E%90%EB%B0%94%EC%BD%94%EB%93%9C%EB%A1%9C%20%EC%9E%91%EC%84%B1%ED%95%9C) - 동적쿼리와 복잡한 쿼리 작성을 도와줌   
    php 프레임워크인 코드이그나이터에서 지원하는 쿼리작성이랑 비슷한 느낌   
    $this->db->select('name, age');   
    $this->db->from('member');   
    $result = $this->db->get();   
    이건 또 따지고보면 php 문법이라기 보다 코드이그나이터에서 따로만든 거니까 비교대상이 아니려나?   
    QueryDSL은 일단 간단한 예제로만 봤을땐 엄청 직관적이다.   
- Spring Data JPA를 많이 사용하는 이유가 뭘까? 분명히 마이바티스가 어떤 한계가 있어서일텐데..!
    - 한계라기보다 변경에 잘 대응할 수 있다: https://github.com/java-squid/2022-jubilant/pull/20/files#r845705511

    

---
> 참고   
> [1. N+1 문제 - 스택오버플로우 답변 - post와 post_comment 예제로 쉽게 설명](https://stackoverflow.com/a/39696775/14058876)   
> 2. JPA란? - 여러 한글로 작성된 블로그들 구글링   
> [3. Spring Data JPA - Reference - 어떤 기능들이 있는지 훑어보긴 괜찮은듯](https://docs.spring.io/spring-data/jpa/docs/current/reference/html/#repositories.create-instances)   
> 4. 나머지 참고는 바로 글에서 Link 해둠   
