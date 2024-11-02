---
title: "Spring JPA의 @GeneratedValue, 그리고 JUnit 테스트"
date: 2024-11-02 23:10:00 +0900
categories: ["JUnit", "Spring Boot"]
tags: [Spring Boot, Java, JUnit, Spring JPA]
---

## 0. 테스트 대상

먼저, 간단한 두 Entity를 보자.

```java
@Entity
public class Department {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private long id;

    @Length(max=255)
    private String name;
}

@Entity
public class Student {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private long id;

    private String name;

    @ManyToOne(fetch = FetchType.LAZY)
    private Department department;
}
```
두 엔티티 모두 ID를 자동 생성하여 인조키로 갖고, Student는 Department에 대한 N:1 참조를 갖는다.

두 엔티티에 대한 JpaRepository도 정의하였다.

```java
public interface StudentRepository extends JpaRepository<Student, Long> {}

public interface DepartmentRepository extends JpaRepository<Department, Long> {}
```

## 1. @GeneratedValue

두 Entity 모두 `@GeneratedValue`에 의해 PK가 Auto increment로 자동으로 생성된다. 만약 생성자에서 id를 임의로 지정해준다면 어떻게 될끼?

```java
@TestMethodOrder(MethodOrderer.OrderAnnotation.class)
@DataJpaTest
@Slf4j
class IdentityTest {

    @Autowired
    DepartmentRepository departmentRepository;

    @Test
    @Order(1)
    void _1Test() {
        Department department1 = new Department(1L, "컴공");
        Department department2 = new Department(3L, "화공");

        departmentRepository.saveAll(List.of(department1, department2));

        log.info("result: {}", departmentRepository.findAll().toString());
    }

    @Test
    @Order(2)
    void _2Test() {
        Department department1 = new Department(132323L, "컴공");
        Department department2 = new Department(3L, "화공");

        departmentRepository.save(department1);
        departmentRepository.save(department2);

        log.info("result: {}", departmentRepository.findAll().toString());
    }
}
```
테스트 환경은 Hibernate 및 H2 DB이다.
결과는 다음과 같다.

```log
result: [Department(id=1, name=컴공), Department(id=2, name=화공)]
result: [Department(id=3, name=화공)]
```

여기서 알 수 있는 사항들은 다음과 같다.

### 1) 데이터 레코드를 추가할 때, 사용자가 설정한 ID 값을 무시하는 @GeneratedValue

첫 번째 테스트에서 ID 값을 임의로 1, 3으로 지정하였지만, 실제로 나오는 ID 값은 1과 2였다.

### 2) 초기화되지 않는 Auto increment 값.

첫 번째 테스트에서 트랜잭션이 끝나 데이터들이 rollback 된 후, 두 번째 테스트에서 id가 3으로 시작하는 것을 볼 수 있다. [여기](https://flowerfulfort.github.io/junit/spring%20boot/JUnit5-%ED%85%8C%EC%8A%A4%ED%8A%B8-%EC%8B%A4%ED%96%89-%EB%B0%A9%EC%8B%9D-%EB%B0%8F-%EC%A3%BC%EC%9D%98%EC%A0%90/)에서 봤던 JUnit 테스트 안에서 다른 클래스의 static 필드가 유지되는 것과 비슷한 맥락으로, 매 테스트마다 Auto increment가 초기화되는 것이 아니다. 하나의 테스트 클래스 내에서 Auto increment 값은 초기화되지 않고 계속 유지된다.

만약 테스트 순서를 정해놓지 않았다면, Auto increment로 인해 부여되는 id값도 뒤죽박죽이 될 것이다.

### 3) save()는 insert 뿐만 아니라 update도 호출한다.

두 번째 테스트의 결과값은 id값이 3인 데이터 하나 뿐이다. 1번과 2번까지만 알고있다면 예상되는 값은 `[Department(id=3, name=컴공), Department(id=4, name=화공)]`이었을 것이다.

그러나 실제로는 둘의 요소가 하나씩 조합된 id가 3인 **화공**이라는 데이터다. 한번 실제 JPQL이 어떻게 생성되었는지 hibernate 옵션을 조절해 로그를 살펴보자.

```sql
[Hibernate] 
    /* insert for
        com.flowerfulfort.testground.entity.Department */insert 
    into
        department (name, id) 
    values
        (?, default)
[Hibernate] 
    /* update
        for com.flowerfulfort.testground.entity.Department */update department 
    set
        name=? 
    where
        id=?
```

처음엔 insert 문이지만 두번쨰는 update 문이다. 왜 그런지 알기 위해 JpaRepository.save() 메소드를 열어보자.

```java
@Transactional
public <S extends T> S save(S entity) {
    Assert.notNull(entity, "Entity must not be null");
    if (this.entityInformation.isNew(entity)) {
        this.entityManager.persist(entity);
        return entity;
    } else {
        return this.entityManager.merge(entity);
    }
}
```

만약 영속성 컨텍스트에서 해당 Entity 가 ID 값을 가지고 있다면(`entityInformation.isNew(entity)`는 id가 null인지 체크한다.), entityManager.merge(entity) 를 수행하는 것을 볼 수 있다.

이로 인해 save()로 전달된 entity를 detached로 인식하고, 영속성 컨텍스트 내에서 id값과 비교해 update문을 수행한다.

또, 임의로 비어있는 id값을 부여한*─비록 @GeneratedValue에 의해 무시되었지만─* entity도 merge()가 실행된다. 따라서 merge()는 insert와 update 구문 모두 생성할 수 있음을 알 수 있다.

<h3>결론</h4>

JpaRepository를 테스트하기 위해 테스트 데이터를 생성해야하고, 해당 테스트들이 인조키 ID에 의존하는 테스트라면 높은 확률로 테스트가 진행되지 않을 것이다.

테스트를 키에 의존하지 않는 방법을 사용하거나 테스트 쿼리를 작성하여 @Sql 어노테이션으로 붙여 데이터를 직접 주입해보자.

## 2. @Sql 어노테이션

테스트에 사용할 SQL 쿼리를 보자.

```sql
insert into department (name)
values ('컴퓨터공학과'), ('인공지능학과');

insert into student (name, department_id)
values ('홍길동', 1), ('김아무개', 2);
```
department를 두개 넣고, student도 2개 만들어 각각 department에 연관을 해주었다.

두 개의 department는 id를 자동으로 생성하게 하고, 두 student는 auto increment로 생성된 id가 1, 2가 될 것이라 생각하고 하드코딩으로 외래키를 작성한다.

이제 테스트 코드를 보자. @Sql 어노테이션을 이용하여 위의 쿼리를 테스트 전에 실행한다.
```java
@Slf4j
@TestMethodOrder(MethodOrderer.OrderAnnotation.class)
@DataJpaTest
class SqlTest {
    @Autowired
    DepartmentRepository departmentRepository;

    @Autowired
    StudentRepository studentRepository;

    @Test
    @Order(1)
    @Sql("classpath:test.sql")
    void _1Test() {
        log.info("Test 1: # of students: {}", studentRepository.count());
    }

    @Test
    @Order(2)
    @Sql("classpath:test.sql")
    void _2Test() {
        log.info("Test 2: # of students: {}", studentRepository.count());
    }
}
```
얼핏 보면 모두 값이 2로 동일한 것 아닌가? 싶다. 그러나 두번째 테스트는 실패하고 만다.
```log
INFO 34980 --- [testground] [main] SqlTest  : Test 1: # of students: 2
WARN 34980 --- [testground] [main] o.s.test.context.TestContextManager      : Caught exception while invoking 'beforeTestMethod' callback on TestExecutionListener [org.springframework.test.context.jdbc.SqlScriptsTestExecutionListener] for test method [void com.flowerfulfort.testground.test.SqlTest._2Test()] and test instance [com.flowerfulfort.testground.test.SqlTest@6d7a343f]

org.springframework.jdbc.datasource.init.ScriptStatementFailedException: Failed to execute SQL script statement #2 of class path resource [test.sql]: insert into student (name, department_id) values ('홍길동', 1), ('김아무개', 2)

...

Caused by: org.h2.jdbc.JdbcSQLIntegrityConstraintViolationException: Referential integrity constraint violation: "FKKH3M8C2TQ2TGRGMA1IYN7TVMX: PUBLIC.STUDENT FOREIGN KEY(DEPARTMENT_ID) REFERENCES PUBLIC.DEPARTMENT(ID) (CAST(1 AS BIGINT))"; SQL statement:
insert into student (name, department_id) values ('홍길동', 1), ('김아무개', 2)
...
```
첫 번째 테스트에서 로그를 출력하여 2라는 결과를 얻었지만, 두번째 테스트에서는 예외가 발생하며 실패해버린다.

로그를 보면 외래키 참조 무결성에 어긋난다고 말한다. 왜 이렇게 되는 것인지 다른 테스트를 진행해보자.

일단 문제가 되는 SQL 구문을 삭제한다.
```sql
insert into department (name)
values ('컴퓨터공학과'), ('인공지능학과');

-- insert into student (name, department_id)
-- values ('홍길동', 1), ('김아무개', 2);
```
그리고 테스트를 수정하여 department 전체를 select하여 출력해보자.
```java
    @Test
    @Order(1)
    @Sql("classpath:test.sql")
    void _1Test() {
//        log.info("Test 1: # of students: {}", studentRepository.count());
        log.info("Test 1: departments: {}", departmentRepository.findAll().toString());
    }

    @Test
    @Order(2)
    @Sql("classpath:test.sql")
    void _2Test() {
//        log.info("Test 2: # of students: {}", studentRepository.count());
        log.info("Test 2: departments: {}", departmentRepository.findAll().toString());
    }
```
결과는 다음과 같다.

```log
INFO 29964 --- [testground] [main] SqlTest  : Test 1: departments: [Department(id=1, name=컴퓨터공학과), Department(id=2, name=인공지능학과)]
INFO 29964 --- [testground] [main] SqlTest  : Test 2: departments: [Department(id=3, name=컴퓨터공학과), Department(id=4, name=인공지능학과)]
```

1번의 save() 테스트와 마찬가지로, 각 테스트마다 @Sql의 쿼리를 가져와 실행하고 끝나기전에 rollback 하는건 맞지만, Auto increment 값은 계속 증가한다.

그렇다면 원래의 SQL 쿼리에서 왜 참조 무결성 오류가 난 것인지 이해할 수 있다.
```sql
insert into student (name, department_id)
values ('홍길동', 1), ('김아무개', 2);
```
이미 1번, 2번 department는 rollback되어 사라졌고, 3번, 4번 department로 새로 추가되어 1, 2번을 찾을 수 없기 때문이다...

다음부턴 AutoIncrement라고 테스트 쿼리에서도 id 안쓰고 그러지 말고 꼬박꼬박 써주자.

## 3. 적절한 테스트 데이터 작성과 레포지터리 테스트

다음과 같이 id를 직접 정의하는 식으로 테스트 쿼리를 수정하였다.

```sql
insert into department (id, name)
values (1, '컴퓨터공학과'), (2, '인공지능학과');

insert into student (id, name, department_id)
values (1, '홍길동', 1), (2, '김아무개', 2);
```

@Sql과 AssertJ를 사용하여 테스트코드를 다음과 같이 짤 수 있다.

```java
@DataJpaTest
class RepositoryTest {
    @Autowired
    StudentRepository studentRepository;

    @Autowired
    DepartmentRepository departmentRepository;

    @Test
    @Sql("classpath:test.sql")
    void findStudentByIdTest() {
        assertThat(studentRepository.findById(2L)).isPresent()
            .get().hasFieldOrPropertyWithValue("name", "김아무개")
            .matches(student -> student.getDepartment().getId() == 2L);
    }

    @Test
    @Sql("classpath:test.sql")
    void findDepartmentByIdTest() {
        assertThat(departmentRepository.findById(1L)).isPresent()
            .get().hasFieldOrPropertyWithValue("name", "컴퓨터공학과");
    }
}
```
