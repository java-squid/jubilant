# Repository vs DAO

## 요약

#### Repository 와 DAO 는 다르다

- Repository 를 DAO 와 같은 방식 혹은 다른 방식으로 구현할 수 있다.
- 다만 회사 레거시가 전부 DAO 라서, DAO 를 Repository 로 리네이밍하는 경우가 많을 뿐이다.
- 즉, DAO 는 전부 Repository 이다. (DAO는 모두 DataSource 에 대한 메커니즘을 제공한다.)
- 하지만, Repository 가 전부 DAO 는 아니다. (DAO 와 다른 메커니즘을 가진 Repository 가 존재한다.)

#### Repository 와 Repository Interface 는 다르다

- DB 와 DBMS 가 다른 것처럼 다르다.
- 프레임워크가 Repository Interface 패턴으로 구현되어 있는 경우가 많다.
- 그렇다고, 개발자가 구현한 Repository Interface 패턴이 프레임워크에 의존적일 필요는 없다.

## 예시 코드

본격적으로 설명에 들어가기에 앞서,

"이름은 빈 칸일 수 없다." 라는 비즈니스 로직을 가진

User 엔티티를 하나 정의해두자.

```kt
@Entity(tableName = "user")
class User(
    @PrimaryKey(autoGenerate = true)
    val id: Long?,
    val name: String
) {
    init {
        check(name.isNotBlank()) {
            "이름은 빈 칸일 수 없습니다."
        }
    }
}
```

check 는 조건을 만족하지 않으면 IllegalStateException 을 발생시키는 코틀린 문법이다.

## DAO 란?

- Data Access Object
- Data 에 Access 할 수 있는 로직을 가지고 있을 뿐이지, Data 그 자체를 갖고 있지는 않다.
- 대신, DAO 는 dataSource 에 대한 메커니즘을 갖고 있다.
  - 가령 DB 에 Access 하는 DAO 의 경우, "쿼리"와 "Mapper" 2가지를 통해 메커니즘을 구현한다.
- Data 를 갖고 있으리라 기대되는 무언가를 **"DataSource"** 라고 부른다.
  - DataSource 의 예시로 DataBase 나, 데이터를 갖고 있는 Repository 가 있다.
  - Data 는 영속적일 수도, 영속적이지 않을 수도 있다.
  - 데이터가 영속적일 경우, 저장소를 Storage 라고 부른다.
  - 데이터가 휘발성일 경우, In Memory 저장소라는 표현을 사용한다.

### DB의 데이터에 Access 하는 DAO 구현 예시

```java
public class UserDao {
    private final JdbcTemplate jdbcTemplate;

    public UserDao(DataSource dataSource) {
        jdbcTemplate = new JdbcTemplate(dataSource);
    }

    public void save(User user) {
        String sql = "INSERT INTO `user` (`name`) values (?) "
                + "ON DUPLICATE KEY UPDATE `name` = ?";
        jdbcTemplate.update(sql, user.getName(), user.getName());
    }

    public long count() {
        String sql = "SELECT COUNT(`id`) AS count FROM `user`";
        RowMapper<Long> rowMapper = (rs, rowNum) -> rs.getLong("count");
        return jdbcTemplate.query(sql, rowMapper).get(0);
    }
}
```

## Repository 란?

Repository 는 한국말로 저장소로 번역되고는 한다.

가령 GitHub Repository 는 깃헙 저장소로 불린다.

더 엄밀한 정의를 살펴보기 위해, springframework 의 공식 문서를 살펴보자.

### org.springframework.stereotype.Repository

Indicates that an annotated class is a "Repository", originally defined by Domain-Driven Design (Evans, 2003) as **"a mechanism for encapsulating storage, retrieval, and search behavior which emulates a collection of objects".**

즉 Repository 의 핵심은 데이터 그 자체가 아니라, 데이터와 관련된 캡슐화를 가능케 하는 메카니즘 그 자체이다. (mechanism for encapsulating)

Repository 는 본인 스스로가 데이터를 가질수도, 데이터를 가지고 있지 않을 수도 있다.

반면 DAO 는 저장소가 아닌 "Access Object" 이므로, 통상적으로 데이터를 가지고 있다고 생각하지 않는다.

### 데이터를 직접 가진 Repository

```kt
class UserRepository {
    private val users: MutableMap<Long, User> = mutableMapOf()

    fun save(user: User) {
        val id = user.id ?: (count() + 1)
        users[id] = user
    }

    fun saveAll(users: Iterable<User>) = users.forEach(::save)

    fun count(): Long = users.size.toLong()

    fun findById(id: Long): User = users.getValue(id)

    fun findAll(): List<User> = users.values.toList()
}
```

getValue 의 경우 key 에 해당하는 value 가 map에 없으면,  
NoSuchElementException 을 발생하는 코틀린 함수이다.

데이터를 In Memory 형태로 스스로 가진 UserRepository 를 구현해보았다.  
유저 데이터를 private 을 통해 캡슐화하였고,  
조작 가능한 매커니즘을 함수를 통해 제공하므로 저장소(Repository)라고 부를 수 있다.

하지만 개발을 하다보면 데이터의 Source 가 동일하지 않는 경우가 자주 생긴다.

가령 프로세스의 메모리 위에 올라간 Data 와,  
로컬 머신의 Storage 에 저장되는 Data 와,  
리모트 DB 서버에서 관리되는 Data 와 같이 3가지 다른 Source 를 가질 수 있다.

각각은 따로따로 관리되어야 하며 동기화를 해주어야할 필요성 뿐만 아니라,  
각각의 Data Source 에 접근(Access) 하기 위한 매커니즘도 모두 다르다.

이러한 차이에 대응하기 위하여, interface 를 통한 추상화가 필요하다.  
이를 **Repository Interface** 디자인 패턴이라고 한다.

개발을 하다보면 Repository Interface 를 축약해서 Repository 라고 하는 경우가 많이 보이는데,  
(마치 DBMS 를 그냥 DB 라고 부르는 것과 비슷하다.)

"Repository" 와 "Repository Interface 패턴" 은 다르다는 인식이 필요하다.

### Repository Interface

```kt
interface  UserRepository {
    fun save(user: User)
    fun saveAll(users: Iterable<User>)
    fun count(): Long
    fun findById(id: Long): User
    fun findAll(): List<User>
}
```

```kt
class InMemoryUserRepository : UserRepository {
    private val users: MutableMap<Long, User> = mutableMapOf()

    override fun save(user: User) {
        val id = user.id ?: (count() + 1)
        users[id] = user
    }

    override fun saveAll(users: Iterable<User>) = users.forEach(::save)

    override fun count(): Long = users.size.toLong()

    override fun findById(id: Long): User = users.getValue(id)

    override fun findAll(): List<User> = users.values.toList()
}
```

UserRepository 인터페이스를 먼저 정의한 후에,  
InMemoryUserRepository 라는 실제 구현체를 만들었다.

그리고 JPA 를 활용하면 위의 코드를 거의 자동으로 생성을 해준다.

```java
public interface RemoteUserRepository extends UserRepository, JpaRepository<User, Long> {}
```

한 줄만 작성하면 RemoteUserRepository 인스턴스를 의존성 주입을 받았을 때,  
JPA 가 자동으로 구현해주는 save, count, findAll, findById 를 마음껏 사용할 수 있다.

하지만 한가지 위화감이 든다.  
RemoteUserRepository 는 `org.springframework.data.jpa.repository.JpaRepository` 에 의존적이기 때문이다.  
즉 RemoteUserRepository 를 캡슐화하고, 메카니즘을 구현하는 주체는 개발자가 아닌 프레임워크이다.

그에 따라 프레임워크에 의존적인 인터페이스를 "DataSource" 라는 다른 명칭을 쓰기로 하였다.  
그렇게 RemoteDataSource 와 LocalDataSource 를 표현해 보자면 다음과 같다.

DataSource 의 인터페이스와 Repository 의 인터페이스는 충분히 다를 수 있다고 생각했기 때문에,  
DataSource 인터페이스는 Repository 인터페이스를 extends 하지 않는다.

```java
public interface RemoteUserDataSource extends JpaRepository<User, Long> {}
```

```kt
@Dao
interface LocalUserDataSource {
    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun saveAll(vararg users: User)

    @Query("SELECT COUNT(id) FROM user")
    suspend fun count(): Int

    @Query("SELECT * FROM user WHERE id = :id")
    suspend fun findById(id: Int): User

    @Query("SELECT * FROM user")
    fun findAll(): List<User>
}
```

보다시피, 안드로이드 Room 의 Dao 를 활용해서 SQLite DB 에 접근할 때,  
vararg 와 관련된 기능을 제공하므로, saveAll 만 있고 save 가 없다.

즉 프레임워크에 의존적인 인터페이스는,  
개발자가 인터페이스를 직접 정의하는게 아니라 프레임워크의 가이드라인을 따르게 되는 경우가 많다.

### 비즈니스가 정말로 요구한 메커니즘

프레임워크에 의존적인 DataSource 인터페이스와,  
비즈니스가 요구하는 Repository 인터페이스를 구분하기 시작하면  
핵심 비즈니스 로직을 바닐라 언어로 구현하는 자유를 얻을 수 있다.

가령 다음과 같은 2개의 비즈니스 로직이 요구되었다고 생각해보자.

1. Remote 서버에 있는 데이터를, Local SQLite DB 로 다운받는다.
2. Local SQLite DB 에 있는 DB 를 조회한다.

그에 따라 구현한 최종 Repository 는 다음과 같다.

```kt
interface  UserRepository {
    fun pullDataFromServer()
    fun getUsers(): List<User>
}
```

```kt
class UserRepositoryImpl(
    private val remoteDataSource: RemoteUserDataSource,
    private val localDataSource: LocalUserDataSource
) : UserRepository {
    override fun pullDataFromServer() {
        val users = remoteDataSource.findAll()
        localDataSource.saveAll(*users.toTypedArray())
    }

    override fun getUsers(): List<User> = localDataSource.findAll()
}
```

## 결론

데이터베이스의 **엔티티** 는 relation(테이블)을 인스턴스화한 자료 구조이다.  
비즈니스 로직을 가진 **도메인 모델** 과는 구분되어야 한다.

엔티티는 필연적으로 프레임워크에 의존적일 수 밖에 없지만,  
도메인 모델은 바닐라 언어로 구현되어야 한다.

## Reference

- [Repository vs DAO](http://aeternum.egloos.com/1160846)
- [Classes vs Data Structure](https://blog.cleancoder.com/uncle-bob/2019/06/16/ObjectsAndDataStructures.html)
- [iOS: Repository Pattern in Swift](https://haningya.tistory.com/232)
- [Android 아키텍처 구성요소](https://developer.android.com/topic/libraries/architecture?hl=ko)
