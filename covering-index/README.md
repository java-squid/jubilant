## 문제사항
- `many-to-many` 로 걸려있는 변수(List)에, 사용자가 내부 값들을 정렬 후 업데이트 했는데 (DB 적용되었음.) 이 변수를 사용하는 쿼리 조회 시 반영이 되지 않던 문제 있었음.
- 의심 가는 부분은, 인덱스 개선 작업.
    - 이 전 인덱스 삭제 -> 새로운 인덱스 (asc, desc) 형태로 새로 생성하는 작업을 진행하였는 데 여기서 파생되지 않았나 생각함.

## 원인
- 커버링 인덱스의 생성 때문에 발생
    - 기존 인덱스 사용 여부를 고려하지 않고, 인덱스(커버링)를 새로 생성하였고 , 이를 사용해서 발생.
-  정리하면 `many-to-many` 로 맵핑되는 다대다 테이블에서 A 필드만 잡고 있던 기존 인덱스를 삭제하고 A,B 를 모두 포함하는 커버링 인덱스 생성으로 인해, 조회 쿼리에서 커버링 인덱스 사용으로 발생한 것으로 판단.

## Index?
- MySQL index 에 대해
- 인덱스는 데이터임.
- 인덱스는 특정 컬럼에 빠르게 접근할 수 있도록 사용됨.
- 인덱스도 정렬할 수 있는 듯
    - MySQL 5.7의 경우 hint를 줘도 asc, 8버젼 부터는 힌트줬을 경우 그에 따라 정렬되는듯.
- 인덱스 갱신은 비싼 작업.
    - 즉 인덱스 갱신이 자주되는 건, DB입장에서 바람직하지 않을듯.

## Covering index?
- SELECT, WHERE, ORDER BY, GROUP BY 등에 사용되는 모든 컬럼이 인덱스의 구성요소인 경우.
- EXPLAIN으로 알아보면, Extra 필드에 `Using Index` 라 표시됨.
    - Extra 필드는 MySQL이 Query를 어떻게 해결할 것인지에 대한 추가적인 정보를 의미.
- `Using Index` ..

> The column information is retrieved from the table using only information in the index tree without having to do an additional seek to read the actual row. 

- 공식문서에 나온 문구
- 해석해보면, 실제 Database Row(데이터가 저장되어있는) 에 접근하는 것이 아닌, Index만 가지고 쿼리를 완성하는 것을 의미한는듯.

### Clustered Key and Non Clustered Key

![](https://t1.daumcdn.net/cfile/tistory/999315505E4933DF02)

- clustered key 는 테이블 당 1개만 있음, PK 혹은 PK 없을 경우 unique key가 이에 해당.
- non clustered key 는 일반적인 인덱스를 의미. 
    - 실제 데이터가 없으며, cluster key로 **접근할 수 있는 값**이 포함되어 있음

> 그러면..

- 커버링 인덱스는 non-cluster-key로만 쿼리를 해결한다는 것.
- 다시 말해, 실제 데이터에 대한 접근이 없다.
- 만약에 실제 데이터의 순서가 반영되어야하는 상황이면 (21, 17, 19.. 순에서 -> 19, 17, 21 저장되었으면..), non-cluster-key로만 접근하는 커버링 인덱스를 사용할 경우, **데이터 갱신이 제대로 안된 것처럼**럼 보일 수 있을듯 (이게 원인)

## 왜 사용하는가?
- 쿼리 속도 향상을 위해.
- WHERE 절 동등비교 + 커버링 인덱스 사용, 그렇지 않을 경우 ORDER BY, GROUP BY 에 최대한 커버링 인덱스를 타게 만드는게 좋을듯.

## 사용시 주의해야할점은?
- 현재 커버링 인덱스를 사용하고 있지 않은데, 예를 들면 모든 Row를 찾고 있는 경우 도입한다면, 주의가 필요 할듯.
- mysql 5.7 의 non-clustered-key를 사용하는 경우에, 정렬을 오름차순만 지원됨.
    - 조회 쿼리가 non-clustered-key를 사용하고 있다면, 해당 테이블 컬럼의 순서를 바꿔도 반영되지 않을 수 있음.

## 정리
- 커버링 인덱스는 조건절에 필요한 컬럼을 모두 '커버' 하는 인덱스를 의미하는듯.
- 커버링 인덱스를 적절하게 사용한다면, 쿼리 실행 속도를 줄일 수 있음.
- 커버링 인덱스를 설정할 때, 컬럼 순서도 중요함
    - 다만, 커버링 인덱스가 사용되기 위해서는 충족이 필요한 조건들이 있음. 이점 유의해야할듯.
- 그리고 인덱스가 무분별하게 늘어나는 건 좋지 않음.

## 추가
- https://jojoldu.tistory.com/243, https://jojoldu.tistory.com/476 포스팅을 통해 실행계획에 대해 한번 이해하면 좋을듯.
- Extra 항목에는 `Using temporary` 와 `Using filesort` 가 나왔는데, 해당 테이블에 인덱스가 있으면, 실행 쿼리가 적절하게 인덱스를 사용하고 있지 못하다라는 증거가 될 수 있음.
- SQL 튜닝시, https://d2.naver.com/helloworld/1155 포스팅도 도움이 될듯.


## 참고
- https://jojoldu.tistory.com/243
- https://jojoldu.tistory.com/476
- https://stackoverflow.com/questions/57606332/what-is-a-mysql-covering-index
- https://tech.kakao.com/2018/06/19/mysql-ascending-index-vs-descending-index/
- https://dev.mysql.com/doc/refman/5.7/en/mysql-indexes.html
- https://tecoble.techcourse.co.kr/post/2021-10-12-covering-index/
