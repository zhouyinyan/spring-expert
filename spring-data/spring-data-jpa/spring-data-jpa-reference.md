[官网链接](https://docs.spring.io/spring-data/jpa/docs/2.0.5.RELEASE/reference/html/#reference)

# Spring data jpa reference

## JPA repositorys

1. Introduction (介绍)
    1.1 jpa namespace(jpa 命名空间-xml)
    JPA module包含了自定义的命名空间，可以用来指定repository beans扫描包，以及包含了JPA特有的特性。一般可以通过repositorie元素来设置JPA repository。 
    Setting up JPA repositories using the namespace
  ```java
  <?xml version="1.0" encoding="UTF-8"?>
  <beans xmlns="http://www.springframework.org/schema/beans"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xmlns:jpa="http://www.springframework.org/schema/data/jpa"
  xsi:schemaLocation="http://www.springframework.org/schema/beans
    http://www.springframework.org/schema/beans/spring-beans.xsd
    http://www.springframework.org/schema/data/jpa
    http://www.springframework.org/schema/data/jpa/spring-jpa.xsd">

  <jpa:repositories base-package="com.acme.repositories" />

</beans>
  ```
  <jpa:repositories/>除了会在"com.acme.repositories"包中查找定义的repository之外，还会自动启用持久化异常转换（JPA持久化异常 转换为 spring DataAccessException体系）。

  <jpa:repositories/>除了通用的设置外，还额外提供了属性（PS://大部分场景不会用，所以了解即可）。 
  Custom JPA-specific attributes of the repositories element

| `entity-manager-factory-ref` | Explicitly wire the `EntityManagerFactory` to be used with the repositories being detected by the `repositories` element. Usually used if multiple `EntityManagerFactory` beans are used within the application. If not configured we will automatically lookup the `EntityManagerFactory` bean with the name `entityManagerFactory` in the `ApplicationContext`. |
| ---------------------------- | ------------------------------------------------------------ |
| `transaction-manager-ref`    | Explicitly wire the `PlatformTransactionManager` to be used with the repositories being detected by the `repositories` element. Usually only necessary if multiple transaction managers and/or `EntityManagerFactory` beans have been configured. Default to a single defined `PlatformTransactionManager` inside the current `ApplicationContext`. |

  1.2 Annotation based configuration(基于注解配置 PS://这才是用得最多的)

   Spring Data JPA repositories using JavaConfig
   ```java
@Configuration
@EnableJpaRepositories  //启用JPA，和XML的命名空间标签所用一样
@EnableTransactionManagement  //启用事务(ps://参考spring-freamwork之data access章节)
class ApplicationConfig {

  @Bean
  public DataSource dataSource() { //使用内置数据源
    
    EmbeddedDatabaseBuilder builder = new EmbeddedDatabaseBuilder();
    return builder.setType(EmbeddedDatabaseType.HSQL).build();
  }

  @Bean
  public LocalContainerEntityManagerFactoryBean entityManagerFactory() {  //配置entityManagerFactory， 注意使用LocalContainerEntityManagerFactoryBean类，而不是直接使用EntityManagerFactory，这是因为前者参与了异常转换机制。

    HibernateJpaVendorAdapter vendorAdapter = new HibernateJpaVendorAdapter();
    vendorAdapter.setGenerateDdl(true);

    LocalContainerEntityManagerFactoryBean factory = new LocalContainerEntityManagerFactoryBean();
    factory.setJpaVendorAdapter(vendorAdapter);
    factory.setPackagesToScan("com.acme.domain");
    factory.setDataSource(dataSource());
    return factory;
  }

  @Bean
  public PlatformTransactionManager transactionManager() { //配置transactionManager

    JpaTransactionManager txManager = new JpaTransactionManager();
    txManager.setEntityManagerFactory(entityManagerFactory());
    return txManager;
  }
}
   ```

2. Persisting entities(持久化实体)

 2.1 saveing entities(保存实体)

  通过CrudRepository.save(…)保存一个实体，它会调用底层的EntityManager persist或者merge方法。 如果实体还没有被JPA持久化，会通过entityManager.persist(…)持久化，否者会调用entityManager.merge(…)更新。  
  实体状态（是否持久化）监测策略：  (PS://使用默认足已)
  Options for detection whether an entity is new in Spring Data JPA

| Id-Property inspection (**default**) | By default Spring Data JPA inspects the identifier property of the given entity. If the identifier property is `null`, then the entity will be assumed as new, otherwise as not new. |
| ------------------------------------ | ------------------------------------------------------------ |
| Implementing `Persistable`           | If an entity implements `Persistable`, Spring Data JPA will delegate the new detection to the `isNew(…)` method of the entity. See the [JavaDoc](https://docs.spring.io/spring-data/data-commons/docs/current/api/index.html?org/springframework/data/domain/Persistable.html) for details. |
| Implementing `EntityInformation`     | You can customize the `EntityInformation` abstraction used in the `SimpleJpaRepository`implementation by creating a subclass of `JpaRepositoryFactory` and overriding the `getEntityInformation(…)` method accordingly. You then have to register the custom implementation of `JpaRepositoryFactory` as a Spring bean. Note that this should be rarely necessary. See the [JavaDoc](https://docs.spring.io/spring-data/data-jpa/docs/current/api/index.html?org/springframework/data/jpa/repository/support/JpaRepositoryFactory.html) for details. |

3. query methods(查询方法)

 3.1 query lookup strategies (查询策略)
 JPA module支持两种方式，一种是手工定义查询串，一种是通过方法名来推导。

 尽管使用方法名推导查询非常方便，但很多时候方法名推导方式不能满足，而且方法名也会太长太丑，鉴于此，可以使用JPA的命名查询方式 或者 直接在查询方法上使用@Query注解。 

 3.2 Query creation (查询创建-即通过方法名推导查询)

  JPA的方法名推导机制和Spring data common中的一样(可以查看对应文档了解详情)，比如：
  Query creation from method names
  ```java
  public interface UserRepository extends Repository<User, Long> {

  List<User> findByEmailAddressAndLastname(String emailAddress, String lastname); //在构建查询的时候会使用JPA的criteria API，但本质上会转化为 "select u from User u where u.emailAddress = ?1 and u.lastname = ?2" 这样的查询。
}
  ```
  JPA 支持的关键字如下：
  Supported keywords inside method names

| Keyword             | Sample                                                       | JPQL snippet                                                 |
| ------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| `And`               | `findByLastnameAndFirstname`                                 | `… where x.lastname = ?1 and x.firstname = ?2`               |
| `Or`                | `findByLastnameOrFirstname`                                  | `… where x.lastname = ?1 or x.firstname = ?2`                |
| `Is,Equals`         | `findByFirstname`,`findByFirstnameIs`,`findByFirstnameEquals` | `… where x.firstname = ?1`                                   |
| `Between`           | `findByStartDateBetween`                                     | `… where x.startDate between ?1 and ?2`                      |
| `LessThan`          | `findByAgeLessThan`                                          | `… where x.age < ?1`                                         |
| `LessThanEqual`     | `findByAgeLessThanEqual`                                     | `… where x.age <= ?1`                                        |
| `GreaterThan`       | `findByAgeGreaterThan`                                       | `… where x.age > ?1`                                         |
| `GreaterThanEqual`  | `findByAgeGreaterThanEqual`                                  | `… where x.age >= ?1`                                        |
| `After`             | `findByStartDateAfter`                                       | `… where x.startDate > ?1`                                   |
| `Before`            | `findByStartDateBefore`                                      | `… where x.startDate < ?1`                                   |
| `IsNull`            | `findByAgeIsNull`                                            | `… where x.age is null`                                      |
| `IsNotNull,NotNull` | `findByAge(Is)NotNull`                                       | `… where x.age not null`                                     |
| `Like`              | `findByFirstnameLike`                                        | `… where x.firstname like ?1`                                |
| `NotLike`           | `findByFirstnameNotLike`                                     | `… where x.firstname not like ?1`                            |
| `StartingWith`      | `findByFirstnameStartingWith`                                | `… where x.firstname like ?1`(parameter bound with appended `%`) |
| `EndingWith`        | `findByFirstnameEndingWith`                                  | `… where x.firstname like ?1`(parameter bound with prepended `%`) |
| `Containing`        | `findByFirstnameContaining`                                  | `… where x.firstname like ?1`(parameter bound wrapped in `%`) |
| `OrderBy`           | `findByAgeOrderByLastnameDesc`                               | `… where x.age = ?1 order by x.lastname desc`                |
| `Not`               | `findByLastnameNot`                                          | `… where x.lastname <> ?1`                                   |
| `In`                | `findByAgeIn(Collection<Age> ages)`                          | `… where x.age in ?1`                                        |
| `NotIn`             | `findByAgeNotIn(Collection<Age> ages)`                       | `… where x.age not in ?1`                                    |
| `True`              | `findByActiveTrue()`                                         | `… where x.active = true`                                    |
| `False`             | `findByActiveFalse()`                                        | `… where x.active = false`                                   |
| `IgnoreCase`        | `findByFirstnameIgnoreCase`                                  | `… where UPPER(x.firstame) = UPPER(?1)`                      |

3.3 Using JPA NamedQueries (使用jpa命名查询)

  注：JPA的命名查询时，使用<named-query />(在xml中)或者@NamedQuery（在实体类中），需要使用JPA query language(JPQL)。当然你也可以使用<named-native-query /> 或者 @NamedNativeQuery，运行你定义本地sql（这会了具体的数据库平台绑定）。

在xml中，使用<named-query />来定义命名的查询：
```xml
<named-query name="User.findByLastname">  <!-- 命名规则：User表示实体， findByLastname表示方法；运行时查找命名查询时会使用该规则-->
  <query>select u from User u where u.lastname = ?1</query>
</named-query>
```
而通常我们使用基于注解的命名查询方式，因为这样不需要单独的定义xml文件，减少维护负担。 
Annotation based named query configuration
```java
@Entity
@NamedQuery(name = "User.findByEmailAddress",   //命名规则同xml一样
  query = "select u from User u where u.emailAddress = ?1")
public class User {

}
```
命名查询定义好了之后，现在就需要在你的repository中定义查询方法（注意查询方法名要和命名查询匹配）:
Query method declaration in UserRepository
```java
public interface UserRepository extends JpaRepository<User, Long> {

  List<User> findByLastname(String lastname);  //spring data 会尝试先使用命名查询，规则就是 实体的简单名称.方法名，比如这儿就会解析到上面xml中定义的命名查询上。 

  User findByEmailAddress(String emailAddress);  // 这个方法会解析到注解上定义的命名查询上
  
  User findByFristname(String firstname); //尝试解析命名查询时，找不到名称为User.findByFristname的查询定义，所以会使用方法名推导方式。
}
```

3.4 Using @Query (使用@Query注解)  (ps://笔者最推荐的方式)

当应用中只有少量的查询时，使用命名查询很好。但因为查询方法本身需要依赖查询语句执行查询，所以你更希望在查询方法上直接使用@Query定义查询语句，而不是在实体类或者xml中定义。这样会让实体类和具体的持久化方式解耦，而且让查询语句位于对应的repository接口中。 (巴拉巴拉一堆好处)

在方法上使用@Query注解定义的查询会比命名查询优先级更高(ps://即如果有@Query注解，那么直接忽略命名查询)

Declare query at the query method using @Query
```java
public interface UserRepository extends JpaRepository<User, Long> {

  @Query("select u from User u where u.emailAddress = ?1")  //JPQL
  User findByEmailAddress(String emailAddress);
}
```

Using advanced `LIKE` expressions  (使用高级的LIKE表达式)

在@Query定义的查询语句中，可以使用LIKE表达式。

Advanced like-expressions in @Query
```
public interface UserRepository extends JpaRepository<User, Long> {

  @Query("select u from User u where u.firstname like %?1")
  List<User> findByFirstnameEndsWith(String firstname);
}
```
注意：在上面的示例中，LIKE的分割符“%”是需要的，这会让查询语句不符合JPQL语法,所以该查询语句会自动的去掉“%”来变成一个有效的JPQL查询，在实际调用该方法时，会自动的将传入的参数增强(根据定义的like模式，在参数的前或者后，或者前后都加上“%”)。

(PS://大家在调试时，可以开启jpa的show sql选项，用于观察执行的具体sql。除此之外，因为jpa使用的提供者hibernate打印的sql仅包含执行语句，没有打印运行时数据，可能在某些场景下不方面调试，给大家推荐一款google出的工具 log4jdbc, 它可以将完整的执行的sql输出出来。使用方法非常简单，自行查询)

Native queries(本地查询 ) (PS://好像用得挺少的)  
通过设置@Query注解的nativeQuery属性为true。  
Declare a native query at the query method using @Query
```java
public interface UserRepository extends JpaRepository<User, Long> {

  @Query(value = "SELECT * FROM USERS WHERE EMAIL_ADDRESS = ?1", nativeQuery = true)
  User findByEmailAddress(String emailAddress);
}
```
注：目前本地查询不支持动态的排序（原因就是框架能够处理JPQL，但是框架不能处理本地SQL）。但是本地查询可以通过指定count query来支持分页。
Declare native count queries for pagination at the query method using @Query
```java
public interface UserRepository extends JpaRepository<User, Long> {

  @Query(value = "SELECT * FROM USERS WHERE LASTNAME = ?1",
    countQuery = "SELECT count(*) FROM USERS WHERE LASTNAME = ?1",
    nativeQuery = true)
  Page<User> findByLastname(String lastname, Pageable pageable);
}
```

3.5 Using Sort (排序)

可以使用Pageable(可以从中获取Sort)或者直接使用Sort来排序。 在Sort中的Order，其property属性需要匹配实体中的中的属性名或者查询语句中的别名。直接看示例：
Using Sort and JpaSort
```java
public interface UserRepository extends JpaRepository<User, Long> {

  @Query("select u from User u where u.lastname like ?1%")
  List<User> findByAndSort(String lastname, Sort sort);

  @Query("select u.id, LENGTH(u.firstname) as fn_len from User u where u.lastname like ?1%")
  List<Object[]> findByAsArrayAndSort(String lastname, Sort sort);
}

repo.findByAndSort("lannister", new Sort("firstname"));     //"firstname" 匹配User中的firstname属性，是有效的          
repo.findByAndSort("stark", new Sort("LENGTH(firstname)"));  //默认不能在order中包含方法调用，会抛出异常 
repo.findByAndSort("targaryen", JpaSort.unsafe("LENGTH(firstname)"));  //可以使用JpaSort来实现不安全的排序，有效
repo.findByAsArrayAndSort("bolton", new Sort("fn_len"));  //可以使用查询语句中的别名，有效      
```

 3.6 Using named parameters(命名参数使用)

  默认spring data jpa会使用基于位置的参数绑定，你可以使用@Param注解来为参数命名，然后在查询语句中使用命名的参数绑定。
  Using named parameters
  ```java
  public interface UserRepository extends JpaRepository<User, Long> {

  @Query("select u from User u where u.firstname = :firstname or u.lastname = :lastname")
  User findByLastnameOrFirstname(@Param("lastname") String lastname,
                                 @Param("firstname") String firstname);
}
  ```
  3.7 Using SpEL expressions (使用SpEL表达式)

  spring data jpa 1.4支持在@Query定义的查询语句中使用SpEL表达式。支持的变量如下：
  Supported variables inside SpEL based query templates  

| Variable     | Usage                            | Description                                                  |
| ------------ | -------------------------------- | ------------------------------------------------------------ |
| `entityName` | `select x from #{#entityName} x` | Inserts the `entityName` of the domain type associated with the given Repository. The `entityName` is resolved as follows: If the domain type has set the name property on the `@Entity` annotation then it will be used. Otherwise the simple class-name of the domain type will be used. |

Using SpEL expressions in repository query methods - entityName
```java
@Entity
public class User {

  @Id
  @GeneratedValue
  Long id;

  String lastname;
}

public interface UserRepository extends JpaRepository<User,Long> {

  @Query("select u from #{#entityName} u where u.lastname = ?1")
  List<User> findByLastname(String lastname);
}
```
这样使用的一个好处是，你要修改你的domain名称(比如重构)，此时不需要修改@Query定义的查询。 
另一个使用场景是，在有通用父类domain class和通用的父类repository接口时，可以复用@Query定义。 结合示例说明：
Using SpEL expressions in repository query methods - entityName with inheritance
```java
@MappedSuperclass  //该注解用在domian class的父类上，不会当做具体的entity处理
public abstract class AbstractMappedType {
  …
  String attribute
}

@Entity
public class ConcreteType extends AbstractMappedType { … }  //具体的实体类继承通用父类

@NoRepositoryBean
public interface MappedTypeRepository<T extends AbstractMappedType>
  extends Repository<T, Long> {         //父类的repository，这里面就可以定义可复用的查询方法

  @Query("select t from #{#entityName} t where t.attribute = ?1")   //该查询可以被所有子repository复用
  List<T> findAllByAttribute(String attribute);    
}

public interface ConcreteRepository
  extends MappedTypeRepository<ConcreteType> { … }   //调用该接口的findAllByAttribute方法时，使用父类中@Query定义的查询语句，并解析为“select t from ConcreteType t where t.attribute = ?1”。 
```
(PS://看起来高大上，但实际场景中不常用，节约不了多少工作量，反而会让代码可读性降低)

 3.8 Modifying queries(修改查询)

前面介绍了如何定义查询来获取实体或者实体集合，定义查询语句来更新实体仅仅需要在查询方法加上@Modifying。 
Declaring manipulating queries
```java
@Modifying
@Query("update User u set u.firstname = ?1 where u.lastname = ?2")
int setFixedFirstnameFor(String firstname, String lastname);
```
对于使用@Modifying删除和使用方法名推导方式删除的说明
Using a derived delete query
```java
interface UserRepository extends Repository<User, Long> {

  void deleteByRoleId(long roleId);  //名称推导

  @Modifying
  @Query("delete from User u where user.role.id = ?1")  //显式声明
  void deleteInBulkByRoleId(long roleId);
}
```
虽然这两个方法都是删除同样的记录。但是在运行时执行的方式不一样。使用第二种方式（@Modifying加@Query）仅会触发一次JPQL查询，即仅会和数据库交互一次。这就意味着，当前已经加载的User实例不会被执行生命周期的回调。

为了确保生命周期被回调，deleteByRoleId(…)会先执行一次查询然后在依次一个一个的删除查询返回的实例，一次持久化提供商能够真正的调用实体上定义的生命周期回调方法。(比如在实体类上用@PreRemove注解的方法)。 (PS://JpaRepository.deleteInBatch 和 CrudRepository的delete 也同理，前一个效率高，但不会触发回调，后一个效率低，但会回调。实际应用场景中，很少有需要定义删除的，基本上记录都是只增不删的。）

3.9 Applying query hints(查询提示？)
(略， 待老夫实战后再补充。 不过目前老夫还没用过，功力还不够)

3.10 Configuring Fetch- and LoadGraphs (配置抓取-加载关联实体)
(略，需要先研究研究JPA 2.1规范先。)

3.11 Projections(投影)
(略， 同spring data commons中的完全一样)

4. stored procedures(存储过程)
(略：几乎不使用了，也建议大家不要使用，业务逻辑不应该放到数据库的存储过程中实现)

5. specifications(ps://规格？规范？)

JPA2 引入了criteria API，可以通过编程方式来构建查询。你可以染给你的repository 接口继承JpaSpecificationExecutor来支持该特性：
```java
public interface CustomerRepository extends CrudRepository<Customer, Long>, JpaSpecificationExecutor {
 …
}
```
现在你就可以使用criteria API来构建查询了， 比如findAll方法会返回所有符合指定规格的实体：
```java
List<T> findAll(Specification<T> spec);
```
Specification接口的定义如下：
```java
public interface Specification<T> {
  Predicate toPredicate(Root<T> root, CriteriaQuery<?> query,
            CriteriaBuilder builder);
}
```
我们来看看怎么使用？
Specifications for a Customer
```java
public class CustomerSpecs {

  public static Specification<Customer> isLongTermCustomer() {
    return new Specification<Customer>() {
      public Predicate toPredicate(Root<Customer> root, CriteriaQuery<?> query,CriteriaBuilder builder) {

         LocalDate date = new LocalDate().minusYears(2);
         return builder.lessThan(root.get(_Customer.createdAt), date);
      }
    };
  }

  public static Specification<Customer> hasSalesOfMoreThan(MontaryAmount value) {
    return new Specification<Customer>() {
      public Predicate toPredicate(Root<T> root, CriteriaQuery<?> query,
            CriteriaBuilder builder) {

         // build query here
      }
    };
  }
}
```
Using a simple Specification
```java
List<Customer> customers = customerRepository.findAll(isLongTermCustomer());
```
Combined Specifications
```java
MonetaryAmount amount = new MonetaryAmount(200.0, Currencies.DOLLAR);
List<Customer> customers = customerRepository.findAll(
  where(isLongTermCustomer()).or(hasSalesOfMoreThan(amount)));
```

(为什么这样做？好处是.......此处神略200字, 好吧，老夫承认没怎么用过，功力不深，没发言权)



6. Query by example(QBE)
(略， 同spring data commons中的几乎一样)

7. Transactionality(事务)

repository提供的CRUD方法默认是事务的。读取操作，事务的readOnly属性设置为ture, 所有其他的方法配置了@Transactional，因此使用了默认的@Transactional属性。如果你想调整SimpleJpaRepository中默认定义的事务配置，那么在你的repository中重新定义方法，并配置事务，比如：
```java
public interface UserRepository extends CrudRepository<User, Long> {

  @Override
  @Transactional(timeout = 10)  //此时findAll方法事务配置是10s超时，并且readOnly标记为true(注解默认)
  public List<User> findAll();

  // Further query method declarations
}
```

通常，我们会在服务层实现中配置事务(非常可能会多repository参与)，比如：
```java
@Service
class UserManagementImpl implements UserManagement {

  private final UserRepository userRepository;
  private final RoleRepository roleRepository;

  @Autowired
  public UserManagementImpl(UserRepository userRepository,
    RoleRepository roleRepository) {
    this.userRepository = userRepository;
    this.roleRepository = roleRepository;
  }

  @Transactional
  public void addRoleToAllUsers(String roleName) {     //该方法会在事务中运行， 关于spring 事务的细节，参考spring freamwork dataaccess章节。

    Role role = roleRepository.findByName(roleName);

    for (User user : userRepository.findAll()) {
      user.addRole(role);
      userRepository.save(user);
    }
}
```

如果要对多个方法应用相同的事务配置，可以在repository接口上上指定事务配合，同时如果在方法上可以个性化事务配置。(PS://实际上JpaRepository的实现类SimpleJpaRepository就是这么玩的。 )
```java
@Transactional(readOnly = true)
public interface UserRepository extends JpaRepository<User, Long> {

  List<User> findByLastname(String lastname);

  @Modifying
  @Transactional
  @Query("delete from User u where u.active = false")
  void deleteInactiveUsers();
}

```
注：对查询方式设置只读事务是有好处的，事务的只读标记会传到底层JDBC驱动，JDBC会对只读做优化。同时spring也会对只读场景做一些优化(基于其底层的JPA实现)。


8. Locking(锁)

直接在查询方法上使用@Lock注解来指定lock模式。
Defining lock metadata on query methods
```java
interface UserRepository extends Repository<User, Long> {

  // Plain query method
  @Lock(LockModeType.READ)
  List<User> findByLastname(String lastname);  //读锁查询
}
```

9. Auditing(审计)

9.1 Basics(基础)

spring data 提供了审计特性，可以让你知道实体是“谁”在“什么时候”创建的，又是“谁”在“什么时候”最后修改的。(因为审计信息是和实体一起的，属于一条记录，所以中间修改过程，以及删除就做不了审计，这种情况下，你可能需要分开的实体来做审计。)

基于注解：  
提供了 @CreatedBy, @LastModifiedBy  @CreatedDate 和 @LastModifiedDate 这4个注解来获取审计信息。 比如：
An audited entity
```java
class Customer {

  @CreatedBy
  private User user;

  @CreatedDate
  private DateTime createdDate;

  // … further properties omitted
}
```
可以看到，上面4个注解可以独立使用，取决于你想要什么信息。 

基于接口：
如果不想使用注解的方式，你可以让实体类实现Auditable接口(该接口暴露了审计信息的设置接口），还有一个更方便的基类AbstractAuditable，继承它时可以让实体类不需要手工实现接口方法，通常我们不会使用基于接口的方式，因为会让你的entity类和spring data耦合。 

在使用@CreatedBy 或者 @LastModifiedBy注解时，框架需要知道当前操作人的身份。为此，提供了AuditorAware<T> SPI 接口，你需要实现该接口来告诉框架当前与应用交互的身份是谁，此处的泛型T需要和使用@CreatedBy 或者 @LastModifiedBy注解的属性类型一致。

比如，使用spring security时，可以使用Authentication来获取当前用户身份：
```java
class SpringSecurityAuditorAware implements AuditorAware<User> {

  public User getCurrentAuditor() {

    Authentication authentication = SecurityContextHolder.getContext().getAuthentication();

    if (authentication == null || !authentication.isAuthenticated()) {
      return null;
    }

    return ((MyUserDetails) authentication.getPrincipal()).getUser();
  }
}
```

10. JPA Auditing(JPA的审计)
  (ps://xml配置略了啊，需要的参考原文档)
  1. 在实体类上使用@EntityListeners注解，并指定AuditingEntityListener
  ```java
@Entity
@EntityListeners(AuditingEntityListener.class)
public class MyEntity {

}
  ```
  2. 在配置类上使用@EnableJpaAuditing注解，启用spring jap auditing 特性
  ```java
@Configuration
@EnableJpaAuditing
class Config {

  @Bean
  public AuditorAware<AuditableUser> auditorProvider() {
    return new AuditorAwareImpl();
  }
}
  ```
  以上两步就能支持@CreatedDate 和 @LastModifiedDate这两个时间的审计了。如果要支持身份审计，需要自定义实现AuditorAware接口，并将其配置为Spring bean(框架会自动找到该bean，完成注入；如果有多个，需要在@EnableJpaAuditing中使用auditorAwareRef指定)


## Miscellaneous (其他)
(略，几乎不会使用到)