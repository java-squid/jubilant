# Pessimistic Locking vs. Optimistic Locking
- 낙관적인 락, 비관적인 락.
- 들어가기 전에,
  - 아마도 락에 대한 처리를 어떻게 할지에 대한 개념일듯.
  - 락에 대해 낙관적으로 생각하느냐, 락에 대해 비관적으로 생각하느냐..정도..?

# About
- 두가지 모두 Lock을 어떻게 다룰 것인가에 대한 모델
  - 데이터베이스 기본 기능은 아닌듯.
- 낙관적인 락
  - 변경 사항이 데이터베이스에 커밋될 때만 레코드가 잠길 것임. 
  - 즉 변경사항이 있어도, 아직 커밋이 안되었으면 잠겨있지 않을듯.
- 비관적인 락
  - 레코드가 변경이 되고 있다면, 잠겨있을 거임. 즉 Lock이 걸려 있을 것.
  - `낙관적인 락` 보다는 좀 더 엄격한 느낌. 만약에 어떤 레코드가 수정 중이라면 (아직 커밋이 안된 상태) 락이 걸려있을 거라 기대하는 모델인듯.


## Optimistic locking

![](https://vladmihalcea.com/wp-content/uploads/2021/03/LostUpdateOptimisticLocking-1536x821.png)
- physical, logcial 적으로 구현이 가능한듯.
- 그렇지만 여러모로 Logical을 사용하는 게 좀 더 효율적으로 볼 수 있을듯
  - 즉 Database 자체 Lock 보다는, 락 처럼 구현한 (Logical) 걸 사용하는게 낫다는 의미인듯.
- 그래서 위 그림으로 보면, `version` 이라는 컬럼을 통해, Logcial Lock을 구현
  - commit 순간 version 을 증가시킴으로써, 다른 Update 문에 `version = 1` 조건이 되어있는 구문은 무시됨을 알 수 있음.
  - 이럴 경우 `PreapredStatement` 는 0을 리턴하고 `data access framework` (예를 들면 JPA..?) 에서는 `OptimisticLockException` 을 던짐.
  - 그럴 경우 Alice 가 진행하던 Transaction 은 **롤백**됨.
  - 적용하려고 했던 update문에 대한 정보가 없어지니까.. 이 부분에 대한 처리가 application 에서 필요할 지도..


## Pessimistic lock
![](https://vladmihalcea.com/wp-content/uploads/2021/03/LostUpdatePessimisticLocking-1536x814.png)
- 하나의 레코드에 동시에 업데이트가 되는 것을 막기 위해 사용됨.
- 어떤 유저가 어떤 레코드에 업데이트를 실행하면, 해당 레코드에는 락이 걸리고, 이 락이 풀어지기 전(이 전 유저의 커밋이 완료될 때 까지) 에는 다른 유저가 해당 레코드에 업데이트를 실행할 수 없음.
- 락이 걸려있을 테니, 충돌을 방지할 수 있음.
  - 즉 트랜잭션 간 충돌을 예방할 수 있음.
  - 그렇지만 성능적으로는 별로 안 좋지 않을까? 
  - 락을 걸고, 풀고, 혹은 다른 트랜잭션에서 해당 레코드에 락이 걸려있는 지 확인하는 작업이 필요할 테니..
- 위 첨부된 그림을 보면, Bob의 UPDATE 구문의 트랜잭션 락은, Alice 의 커밋이 완료되기 전까지는 획득할 수 없음.
  - 그런데 만약에 Alice가 commit이 어떤 이유에서 그런지 끝나지 않는다면..?
  - 일정 시간이 지나면 Dead Lock 같은게 발생하지 않을까?

# In JPA


# Sample

> 코드를 통해, 지금까지 정리한 내용 되짚어보기.


# 추가적인 키워드
- Shared Lock, Exclusive Lock 
- MVCC
- Dead Lock

# 참고
- https://www.ibm.com/docs/en/rational-clearquest/7.1.0?topic=clearquest-optimistic-pessimistic-record-locking
- https://vladmihalcea.com/optimistic-vs-pessimistic-locking/
- https://www.baeldung.com/jpa-optimistic-locking
- https://okky.kr/article/1023929
- https://velog.io/@wmpark90/JPA-%EB%82%99%EA%B4%80%EC%A0%81-%EB%9D%BD%EA%B3%BC-%EB%B9%84%EA%B4%80%EC%A0%81-%EB%9D%BD