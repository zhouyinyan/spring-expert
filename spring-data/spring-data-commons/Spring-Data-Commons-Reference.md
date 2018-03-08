[官方文档](https://docs.spring.io/spring-data/commons/docs/current/reference/html/)



# Spring DATA Commons

## 1.依赖

## 2. 使用spring data repository

## 3. 投影(projections)
## 4. query by example(QBE)
## 5. 审计

## 附录

1. Dependencies(依赖)

Using the Spring Data release train BOM
```java
<dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>org.springframework.data</groupId>
      <artifactId>spring-data-releasetrain</artifactId>
      <version>${release-train}</version>
      <scope>import</scope>
      <type>pom</type>
    </dependency>
  </dependencies>
</dependencyManagement>
```
Declaring a dependency to a Spring Data module
```java
<dependencies>
  <dependency>
    <groupId>org.springframework.data</groupId>
    <artifactId>spring-data-jpa</artifactId>
  </dependency>
<dependencies>
```

如果使用spring-boot，spring boot已经选择了较为新的spring data 版本，如果你想升级，简单的配置spring-data-releasetrain.version到你想要的版本即可。 (一般用spring boot默认的足以)。 

2. Working with Spring Data Repositories

spring data repository 的目标是在使用不同的持久化存储时，减少样板代码。

2.1 核心概念

spring data repository的核心接口是 `Repository`. 它使用领域对象和领域对象的ID作为其类型参数。该接口是一个标记接口，主要作用是可以获取起参数类型(领域对象类型)和帮助发现继承之它的子接口。`CrudRepository`提供了CRUD方法。
CrudRepository interface
```java
public interface CrudRepository<T, ID extends Serializable>
  extends Repository<T, ID> {

  <S extends T> S save(S entity);       //保存传入的实体  

  Optional<T> findById(ID primaryKey);  //根据ID查询，适应optional包装

  Iterable<T> findAll();               //查询所有实体

  long count();                        //返回总数

  void delete(T entity);               //删除传入的实体

  boolean existsById(ID primaryKey);   //监测传入ID对应的实体是否存在

  // … more functionality omitted.
}
```
在`CrudRepository`之上，`PagingAndSortingRepository`添加了分页和排序的能力。 

PagingAndSortingRepository
```java
public interface PagingAndSortingRepository<T, ID extends Serializable>
  extends CrudRepository<T, ID> {

  Iterable<T> findAll(Sort sort);

  Page<T> findAll(Pageable pageable);
}
```
比如，分页查询用户第二页数据，page size为20：
```java
PagingAndSortingRepository<User, Long> repository = // … get access to a bean
Iterable<User> users = repository.findAll(new PageRequest(1,2));
```

另外，通过方法名推导查询时， count和delete 也是支持的。比如：
```java
interface UserRepository extends CrudRepository<User,Long> {
    long countByLastname(String lastname);
}
```
and:
```java
interface UserRepository extends CrudRepository<User,Long> {

    long deleteByLastname(String lastname);
    
    List<User> removeByLastname(String lastname);
}
```

2.2 查询方法

使用spirng data，使用4步来声明查询方法：

  1. 定义一个接口，继承至Repository接口或者它的子接口，并提供领域对象类型和ID类型作为类型参数。
  ```java
  interface PersonRepository extends Repository<Person, Long> { … }
  ```
  2. 在该接口上定义查询方法
  ```java
  interface PersonRepository extends Repository<Person, Long> {
     List<Person> findByLastname(String lastname);
  }
  ```
  3. 配置spring来定义的repitory接口创建代理对象。可通过JavaConfig:
  ```java
import     org.springframework.data.jpa.repository.config.EnableJpaRepositories;

@EnableJpaRepositories
class Config {}
  ```
  或者xml配置
  ```xml
  <?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
   xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
   xmlns:jpa="http://www.springframework.org/schema/data/jpa"
   xsi:schemaLocation="http://www.springframework.org/schema/beans
     http://www.springframework.org/schema/beans/spring-beans.xsd
     http://www.springframework.org/schema/data/jpa
     http://www.springframework.org/schema/data/jpa/spring-jpa.xsd">

   <jpa:repositories base-package="com.acme.repositories"/>

</beans>
  ```
  4. 注入repository实例，直接使用
  ```java
  class SomeClient {

  private final PersonRepository repository;

  SomeClient(PersonRepository repository) {
    this.repository = repository;
  }

  void doSomething() {
    List<Person> persons = repository.findByLastname("Matthews");
  }
}
  ```
下面详细说明这4个步骤。

2.3 定义repository接口  
  第一步是定义一个指定domain class的repository接口，该接口必须继承`Repository`，并提供Domain class和ID类型。 如果你想暴露CRUD方法，继承`CrudRepository`即可。  

  2.3.1 微调repository接口
  通常，你的 repository 接口会继承Repository, CrudRepository 或者 PagingAndSortingRepository 。此外，如果你不想继承spring data 的接口，你可以在你的Repository接口上打上@RepositoryDefinition (该注解的作用和继承Repository是一样的)。 继承CrudRepository会暴露一组完整的操作，如果你只想暴露部分操作，简单的拷贝CrudRepository中的方法到你的Repository接口。比如:
  ```java
@NoRepositoryBean
interface MyBaseRepository<T, ID extends Serializable> extends Repository<T, ID> {

  Optional<T> findById(ID id);

  <S extends T> S save(S entity);
}

interface UserRepository extends MyBaseRepository<User, Long> {
  User findByEmailAddress(EmailAddress emailAddress);
}
  ```
注意：@NoRepositoryBean通常会标记在中介Repository接口上，作用是让spring不会为它创建代理实例。  

  2.3.2 Repository 方法中的 Null 处理  

  Repository中的方法返回值可以定义为Java 8’s Optional类型。spring data也支持其他包装类型如: com.google.common.base.Optional, scala.Option, io.vavr.control.Option。 (使用java8时，推荐使用jdk自带的)。   
  当然，也可以选择不返回包装类型， 此时，当结果缺失时会返回null。 当方法定义中返回的是collections, collection alternatives, wrappers, 和 streams时，spring data会保证不会返回null，而是用相对应的"empty"表示。 在 返回值 一节详解。 

  你可以使用Spring Framework’s nullability annotations来约束 repository methods 。 
- @NonNullApi – to be used on the package level to declare that the default behavior for parameters and return values is to not accept or produce null values.

- @NonNull – to be used on a parameter or return value that must not be null (not needed on parameter and return value where @NonNullApi applies).

- @Nullable – to be used on a parameter or return value that can be null.

  ```java
  @org.springframework.lang.NonNullApi  //定义该包下的默认行为，参数和返回值会在运行时收到非空检查，违反时会抛出异常。
  package com.acme;
  ```
  ```java
  package com.acme;                                                       
  ```

import org.springframework.lang.Nullable;

interface UserRepository extends Repository<User, Long> {

  User getByEmailAddress(EmailAddress emailAddress);   //如果没有返回结果，则会抛出EmptyResultDataAccessException异常；如果传入参数emailAddress为null，则会抛出IllegalArgumentException                  

  @Nullable
  User findByEmailAddress(@Nullable EmailAddress emailAdress);  //如果没有查询到结果，返回null；也接受传入参数 emailAdress为null。      

  Optional<User> findOptionalByEmailAddress(EmailAddress emailAddress); //如果没有查询到结果，会返回Optional.empty()；如果传入参数emailAddress为null，则会抛出IllegalArgumentException              
}
  ```
  Nullability in Kotlin-based repositories (略，笔者没用过，感兴趣的同学可以自行学习)
	
  2.3.3 多spring data modules下的repository 
  使用单一的spring data module非常简单，所有的repository 接口都会绑定到该module。而有些应用需要使用多module，这种情况下，需要 repository 定义来区分不同的持久化技术。spring data 需要严格的repository配置，因为它会自动检查classpath下的多个repository factories。
  1. 如果repository定义是继承至指定module的repository接口，会自动绑定到指定的module。
 
  2. 如果domain class打上了指定module 类型的注解，那么处理该domain class的repository会绑定到该module。
  
  3. 使用repository base packages来区分。

  Repository definitions using Module-specific Interfaces
  ```java
interface MyRepository extends JpaRepository<User, Long> { }

@NoRepositoryBean
interface MyBaseRepository<T, ID extends Serializable> extends JpaRepository<T, ID> {
  …
}

interface UserRepository extends MyBaseRepository<User, Long> {
  …
}
  ```
  MyRepository和UserRepository继承至JpaRepository，因此会绑定到Spring Data JPA module。

  Repository definitions using generic Interfaces
  ```java
interface AmbiguousRepository extends Repository<User, Long> {
 …
}

@NoRepositoryBean
interface MyBaseRepository<T, ID extends Serializable> extends CrudRepository<T, ID> {
  …
}

interface AmbiguousUserRepository extends MyBaseRepository<User, Long> {
  …
}
  ```
AmbiguousRepository 和 AmbiguousUserRepository 继承至 Repository and CrudRepository， 若果是单spring data module下没问题，但如果是多module下，spring不知道绑定到具体的那个module。

Repository definitions using Domain Classes with Annotations
```java
interface PersonRepository extends Repository<Person, Long> {
 …
}

@Entity
class Person {
  …
}

interface UserRepository extends Repository<User, Long> {
 …
}

@Document
class User {
  …
}
```
PersonRepository 引用了Person，而Person使用了JPA的注解@Entity，因此PersonRepository会绑定到spring data jpa。UserRepository 引用的 User 使用了 MongoDB的注解@Document，因此UserRepository会绑定到spring data mongo。  

Repository definitions using Domain Classes with mixed Annotations
```java
interface JpaPersonRepository extends Repository<Person, Long> {
 …
}

interface MongoDBPersonRepository extends Repository<Person, Long> {
 …
}

@Entity
@Document
class Person {
  …
}
```
Person 即用了JPA的注解，也用了mongo的注解，此时spring data就不知道具体怎么绑定了。 

Annotation-driven configuration of base packages
```java
@EnableJpaRepositories(basePackages = "com.acme.repositories.jpa")
@EnableMongoRepositories(basePackages = "com.acme.repositories.mongo")
interface Configuration { }
```

2.4 定义Query Method

spring data创建repository 代理有两种方式获取查询语句。可以通过方法名称推导或者使用手动的定义查询。 spring data 具体使用哪种方式，依赖于query lookup strategy。

  2.4.1 query lookup strategy(查询策略)
  你可以在xml配置文件中，使用具体module 命名空间的query-lookup-strategy属性配置；或者在java config中使用Enable${store}Repositories 注解的queryLookupStrategy属性配置。支持的策略：
  - CREATE 通过方法名称来构建具体的查询。 通用的方式是去掉方法名称的前缀，解析剩下的部分。在query createion中详解。 
  - USE_DECLARED_QUER 查找和使用定义好的query，如果未定义会抛出异常。query可以通过注解定义。如果spring data 在启动时没有找到定义的query，直接启动失败。 
  - CREATE_IF_NOT_FOUND(默认)，CREATE 和 USE_DECLARED_QUERY 的组合。首先查询定义的query，如果未找到，在通过方法名来构建query。默认使用该策略。 

  2.4.2 Query creation (查询构建)
  query 构建原理很简单，通过find…By, read…By, query…By, count…By, 和 get…By切割方法名，然后解析剩下的。方法名可以包含如Distinct来设置distinct标记，也可以通过And 和 Or 来连接多个条件。 看例子:
  Query creation from method names
  ```java
interface PersonRepository extends Repository<User, Long> {

  List<Person> findByEmailAddressAndLastname(EmailAddress emailAddress, String lastname); //通过emailAddress和lastname查询Person， 

  // Enables the distinct flag for the query
  List<Person> findDistinctPeopleByLastnameOrFirstname(String lastname, String firstname);
  List<Person> findPeopleDistinctByLastnameOrFirstname(String lastname, String firstname);

  // Enabling ignoring case for an individual property
  List<Person> findByLastnameIgnoreCase(String lastname);
  // Enabling ignoring case for all suitable properties
  List<Person> findByLastnameAndFirstnameAllIgnoreCase(String lastname, String firstname);

  // Enabling static ORDER BY for a query
  List<Person> findByLastnameOrderByFirstnameAsc(String lastname);
  List<Person> findByLastnameOrderByFirstnameDesc(String lastname);
}
  ```

  2.4.3 Property expressions

  属性表达式必须引用实体中的属性。当然，也可以使用嵌套的属性。 比如一个Person有一个Address对象的属性，Address又包含ZipCode。这种情况下，一个查询方法名为:
  ```java
  List<Person> findByAddressZipCode(ZipCode zipCode);
  ```
  会解析为x.address.zipCode。
  解析算法：
  The resolution algorithm starts with interpreting the entire part (AddressZipCode) as the property and checks the domain class for a property with that name (uncapitalized). If the algorithm succeeds it uses that property. If not, the algorithm splits up the source at the camel case parts from the right side into a head and a tail and tries to find the corresponding property, in our example, AddressZip and Code. If the algorithm finds a property with that head it takes the tail and continue building the tree down from there, splitting the tail up in the way just described. If the first split does not match, the algorithm move the split point to the left (Address, ZipCode) and continues.(ps://简而言之，从右到左切割，从左到右依次匹配)。  
  这种算法可能会匹配到错误的属性上，假设Person还有一个addressZip，那么会直接匹配上addressZip属性，而addressZip没有code属性，最终就会失败。 
  为了解决这个问题，可以在方法名中使用_来指定分割点，比如:
```java
List<Person> findByAddress_ZipCode(ZipCode zipCode);
```
  虽然使用_可以解决匹配问题，但强烈建议方法命名要符合java 命名规范，即使用驼峰命名。 

  2.4.4 Special parameter handling(特殊参数处理)

  一些特殊的参数比如Pageable 和 Sort可以加入到查询方法上，以支持动态的分页和排序查询。  比如：
  Using Pageable, Slice and Sort in query methods
  ```java
Page<User> findByLastname(String lastname, Pageable pageable); //传入Pageable实例，分页查询用户，返回Page(包含总数和页数等信息，因此会自动触发一次count查询)。

Slice<User> findByLastname(String lastname, Pageable pageable); //传入Pageable实例，分页查询用户，返回Slice(因为返回Page会触发count查询，对存储来说可能是昂贵的操作，使用Slice返回时，仅知道是否还有下一个slice，这在非常大的结果集中可能更合适)

List<User> findByLastname(String lastname, Sort sort); //查询排序。(Pageale实例已经包含了Sort，但如果仅仅需要排序，简单的添加Sort参数)

List<User> findByLastname(String lastname, Pageable pageable); //分页查询，返回List(返回List时，仅仅就是查询该页范围中的实体即可，而不像返回Page时，会因为要计算一些元信息而触发的额外count查询。)
  ```

  2.4.5 Limiting query results(限制返回结果)

  查询返回结果可以通过关键字 first 或者 top。一个可选的数值可以附加在top/first关键字后，来指定最大的结果数。
Limiting the result size of a query with Top and First
```java
User findFirstByOrderByLastnameAsc(); //返回lastname正序后的第一个用户

User findTopByOrderByAgeDesc(); //返回age逆序后的第一个用户

Page<User> queryFirst10ByLastname(String lastname, Pageable pageable); //返回前10
 
Slice<User> findTop3ByLastname(String lastname, Pageable pageable); //返回前3

List<User> findFirst10ByLastname(String lastname, Sort sort); //返回前10

List<User> findTop10ByLastname(String lastname, Pageable pageable); //返回前10
```
  limit表达式也支持Distinct关键字，同样，限制返回一个实体实例时，返回结果可以通过Optional包装。 

  2.4.6 Streaming query results (流化返回结果)

  查询方法可以使用Java 8 Stream<T>作为返回值类型，不是使用Stream简单的包装查询结果，而是应用具体的存储提供的方法来执行流化。(因此，不是所有的spring data module都支持流化)
  Stream the result of a query with Java 8 Stream<T>
  ```java
@Query("select u from User u")
Stream<User> findAllByCustomQueryAndStream();

Stream<User> readAllByFirstnameNotNull();

@Query("select u from User u")
Stream<User> streamAllPaged(Pageable pageable);
  ```
  注：Stream在使用完后需要关闭。你可以手动的调用close()方法关闭，也可以使用java 7 try-with-resources 块。 比如
  ```java
try (Stream<User> stream = repository.findAllByCustomQueryAndStream()) {
  stream.forEach(…);
}
  ```

  2.4.7 Async query results(异步查询)
 查询可以通过使用Spring’s asynchronous method execution capability异步执行。这意味着调用查询方法时会立即返回，实际的查询操作会包装为task提交到spring taskexecutor中异步执行。
 ```java
@Async
Future<User> findByFirstname(String firstname);       //使用java.util.concurrent.Future作为返回值类型        

@Async
CompletableFuture<User> findOneByFirstname(String firstname);  //使用java8 java.util.concurrent.CompletableFuture 作为返回值类型

@Async
ListenableFuture<User> findOneByLastname(String lastname);   //使用org.springframework.util.concurrent.ListenableFuture 作为返回值类型
 ```

 2.5 Creating repository instances(创建repository实例)

可以在xml中使用spring data module提供的命名空间配置，而通常建议使用java-config风格配置。

  2.5.1 xml配置
  (查看文档说明，因为通常已经不适用xml配置了，所以略过)

  2.5.2 JavaConfig
  使用@Enable${store}Repositories注解。比如使用jpa module：
  ```java
@Configuration
@EnableJpaRepositories("com.acme.repositories")
class ApplicationConfiguration {

  @Bean
  EntityManagerFactory entityManagerFactory() {
    // …
  }
}
  ```

  2.5.3 Standalone usage (独立使用)
  你也可以在spring容器外使用repository设施，通常需要编程方式来设置repositories。 比如:
  Standalone usage of repository factory
  ```java
  RepositoryFactorySupport factory = … // Instantiate factory here
  UserRepository repository = factory.getRepository(UserRepository.class);
  ```

  2.6 Custom implementations for Spring Data repositories(自定义实现Repository)

  当默认的spring data提供的repository不能满足需求是，需要自定义repository实现。spring data运行你提供自定义的repository 代码，并且非常容易的和通用的CRUD和查询方法集成。 

   2.6.1 自定义特别的repository
  为了让repository使用自定义的方法，首先先定义一个片段接口(fragment interface, 片段可以理解为一部分功能)，并实现该接口中的自定义方法，然后让repository接口继承至该自定义接口即可。 
  Interface for custom repository functionality
  ```java
  interface CustomizedUserRepository {
    void someCustomMethod(User user);
  }
  ```
  Implementation of custom repository functionality
  ```java
class CustomizedUserRepositoryImpl implements CustomizedUserRepository {

  public void someCustomMethod(User user) {
    // Your custom implementation
  }
}
  ```
住：实现类非常重要的一点就是其名字是 接口名称 + ”Impl“后缀。
Changes to your repository interface
```java
interface UserRepository extends CrudRepository<User, Long>, CustomizedUserRepository {

  // Declare query methods here
}
```
添加repository继承至自定义接口，此时UserRepository就有了CRUD和自定义方法。

Spring data 的repository就是通过聚合不同的片段接口来实现的。片段接口可以是基础的repository(比如CrudRepository),和功能性的切面比如QueryDsl，以及自定义的片段接口。每次你的repository添加继承一个片段接口，该repository就会增强该片段接口的能力。spring data module都提供基础的片段接口和功能性的切面片段接口的实现。  

Fragments with their implementations（PS://原文的代码好像有点笔误。）
```java
interface HumanRepository {
  void someHumanMethod(User user);
}

class HumanRepositoryImpl implements HumanRepository {

  public void someHumanMethod(User user) {
    // Your custom implementation
  }
}

interface ContactRepository {

  void someContactMethod(User user);

  User anotherContactMethod(User user);
}

class ContactRepositoryImpl implements ContactRepository {

  public void someContactMethod(User user) {
    // Your custom implementation
  }

  public User anotherContactMethod(User user) {
    // Your custom implementation
  }
}
```
Changes to your repository interface
```java
interface UserRepository extends CrudRepository<User, Long>, HumanRepository, ContactRepository {

  // Declare query methods here
}
```

自定义的实现拥有比基础的respository和切面repository更高的优先级，这允许你覆盖基础和切面的方法，而如果有多个自定义片段接口中包含相同的方法签名时，会按照引用顺序，解决冲突。片段接口不限于只用于单个repository接口，多个repository可以使用同一个片段接口来复用。

Fragments overriding save(…)
```java
interface CustomizedSave<T> {
  <S extends T> S save(S entity);
}

class CustomizedSaveImpl<T> implements CustomizedSave<T> {

  public <S extends T> S save(S entity) {
    // Your custom implementation
  }
}
```
Customized repository interfaces
```java
interface UserRepository extends CrudRepository<User, Long>, CustomizedSave<User> {
}

interface PersonRepository extends CrudRepository<Person, Long>, CustomizedSave<Person> {
}
```

  spring data 在创建repository代理实例时，会通过其继承的自定义片段接口查找自定义实现，查找规则就是自定义片段接口+默认的后缀(Impl). 可以通过配置修改后缀。 比如:
```java
<repositories base-package="com.acme.repository" repository-impl-postfix="FooBar" />
```
同样在@Enable{Stroe}Repository注解中也有对应的属性配置。  

Resolution of ambiguity(略，实际场景中用不着)

Manual wiring (略，理论上也不需要这么使用)

  2.6.2  Customize the base repository (自定义基础repository)

  当你想要自定义基础repository行为时，你需要创建一个自定义实现，继承至具体的持久化技术的基础repository，在创建repository代理时，该自定义实现扮演基础repository。 

  Custom repository base class
  ```java
  class MyRepositoryImpl<T, ID extends Serializable>
  extends SimpleJpaRepository<T, ID> {

  private final EntityManager entityManager;

  MyRepositoryImpl(JpaEntityInformation entityInformation,
                          EntityManager entityManager) {
    super(entityInformation, entityManager);

    // Keep the EntityManager around to used from the newly introduced methods.
    this.entityManager = entityManager;
  }

  @Transactional
  public <S extends T> S save(S entity) {
    // implementation goes here
  }
}
  ```
  注意：自定义实现需要一个同父类一样的构造方法(框架用于构建父类)，如果父类有多个构造方法，选EntityInformation加上具体持久化技术的基础设施对象(比如EntityManager或者template)。  

  最后一步就是让spring data知道自定义的实现。在javaconfig可以通过@Enable…Repositories中的repositoryBaseClass属性指定。 
  ```java
@Configuration
@EnableJpaRepositories(repositoryBaseClass = MyRepositoryImpl.class)
class ApplicationConfiguration { … }
  ```
  xml中同样配置即可。

  2.7 Publishing events from aggregate roots(发布事件)
  (略，目前笔者还没使用该特性，也许以后会用)

  2.8 spring data extensions (spring data 扩展)

   2.8.1 Querdsl extension
   querydsl是一个框架，提供fluent api来构建查询，一些spring data modules提供了对querydsl的集成，通过QueryDslPredicateExecutor接口。
   QueryDslPredicateExecutor interface
   ```java
 public interface QueryDslPredicateExecutor<T> {

  Optional<T> findById(Predicate predicate);   //查询匹配传入predicate的单个实体

  Iterable<T> findAll(Predicate predicate); //查询匹配传入predicate的所有实体  

  long count(Predicate predicate);          //查询匹配传入predicate的实体数目

  boolean exists(Predicate predicate);        //查询是否存在实体匹配传入predicate

  // … more functionality omitted.
}
   ```
你可以让你的repository继承QueryDslPredicateExecutor接口来支持querydsl。
Querydsl integration on repositories
```java
interface UserRepository extends CrudRepository<User, Long>, QueryDslPredicateExecutor<User> {

}

Predicate predicate = user.firstname.equalsIgnoreCase("dave")
	.and(user.lastname.startsWithIgnoreCase("mathews"));

userRepository.findAll(predicate);
```

  2.8.2 web support 
  (略，目前还没怎么用过，不过看起来使用起来挺方便)

  2.8.3 Repository populators(数据填充)
  (略，还没使用过，做数据初始化用的，在自动化集成测试场景下比较有用)

  2.8.4 Legacy web support
  (略, 在spring data common 1.6,对web支持这块修改较多，所以原文档分成两块不同的部分来介绍。)


3. projections (投影)

查询方法通常返回一个或者多个实体（包含实体的所有属性），然后有时只需要返回实体的部分属性。比如一个实体类：
A sample aggregate and repository
```java
class Person {

  @Id UUID id;
  String firstname, lastname;
  Address address;

  static class Address {
    String zipCode, city, street;
  }
}

interface PersonRepository extends Repository<Person, UUID> {

  Collection<Person> findByLastname(String lastname);
}
```
现在，想象一下，我们只想获取person的name属性，该如何做？

3.1 Interface-based projections(基于接口投影)
最简单的限制查询返回的属性是通过定义仅暴露需要返回属性的accessor 方法的接口：
A projection interface to retrieve a subset of attributes
```java
interface NamesOnly {

  String getFirstname();
  String getLastname();
}
```
最重要的一点是这儿的属性定义必须严格匹配实体中的对应属性。然后在查询方法上使用该接口作为返回的实体类型：
A repository using an interface based projection with a query method
```java
interface PersonRepository extends Repository<Person, UUID> {

  Collection<NamesOnly> findByLastname(String lastname);
}
```
查询执行引擎会在运行时创建该接口的代理对象，并且当调用该接口暴露的的accessor方法时，转发调用到真正的目标实体对象上。  

投影可以递归使用。如果你想包含Address中的某些属性，创建一个Address的投影接口，并在接口中定义返回的属性，比如：

A projection interface to retrieve a subset of attributes
```java
interface PersonSummary {

  String getFirstname();
  String getLastname();
  AddressSummary getAddress();

  interface AddressSummary {
    String getCity();
  }
}
```

 3.1.1 Closed projections (ps://这个名词不知道怎么翻译)

 一个投影接口的accessor方法，全都和目标实体的属性完全匹配，称为“Closed projections”， 比如：
 A closed projection
 ```java
 interface NamesOnly {

  String getFirstname();
  String getLastname();
}
 ```
 使用Closed projections，spring data modules能够优化查询，因为我们精确的知道所有需要返回给投影代理的属性。 可以查看具体的module文档获取更多细节。  

 3.1.2 Open projections

 Accessor 方法也可以使用@Value来动态的计算值。使用@Value的投影接口称为“Open projections” 比如：
 An Open Projection
 ```java
interface NamesOnly {

  @Value("#{target.firstname + ' ' + target.lastname}")
  String getFullName();
  …
}
 ```
 在投影接口中，可以通过 target变量来引用实体。spring data不能应用查询优化，因为SpEL表达式可能使用实体中的任意属性。   

 @Value的表达式不应该太复杂，因为你应该避免使用String方式编程。对于表达式， 一种选择是凭借java8默认方法(default method)：
 A projection interface using a default method for custom logic
 ```java
 interface NamesOnly {

  String getFirstname();
  String getLastname();

  default String getFullName() {
    return getFirstname.concat(" ").concat(getLastname());
  }
}
 ```
 这种方式需要你仅能通过其他的accessor方法来实现逻辑。 另一种更灵活选择是使用一个spring bean来实现自定义逻辑，然后在SpEL表达式中简单的调用即可。 比如：
 ```java
 @Component
class MyBean {

  String getFullName(Person person) {
    …
  }
}

interface NamesOnly {

  @Value("#{@myBean.getFullName(target)}")
  String getFullName();
  …
}
 ```
在SpEL中引用MyBean的实例，调用起getFullName方法，并将实体对象作为参数传入。 SpEL表达式也可以使用方法参数，方法名为arg，值为Object数组。 比如：
```java
interface NamesOnly {

  @Value("#{args[0] + ' ' + target.firstname + '!'}")
  String getSalutation(String prefix);
}
```
再次提醒，对于复杂的表达式应该使用spring bean和在表达式中调用的方式。

3.2 Class-based projections (DTOs) (基于类映射)

另一种定义投影的方式是使用值对象DTOs。DTOs的使用方式和投影接口完全一样，除了不适用代理以及不能投影内嵌投影。

使用DTOs方式，spring data modules能够通过限制要查询的字段来优化查询，要查询的字段通过构造方法的参数来决定。 

A projecting DTO

```java
class NamesOnly {

  private final String firstname, lastname;

  NamesOnly(String firstname, String lastname) {

    this.firstname = firstname;
    this.lastname = lastname;
  }

  String getFirstname() {
    return this.firstname;
  }

  String getLastname() {
    return this.lastname;
  }

  // equals(…) and hashCode() implementations
```
注：对于DTOs的样板代码(比如getter setter hashcode equals)，可使用lombok来消除。(ps://本人很少使用)

3.3 Dynamic projections(动态投影)

在之前使用的具体的投影类型或者实体类型返回。有时可能需要调用方法时来选择类型。比如：
A repository using a dynamic projection parameter
```java
interface PersonRepository extends Repository<Person, UUID> {

  Collection<T> findByLastname(String lastname, Class<T> type);
}
```
Using a repository with dynamic projections
```java
void someMethod(PersonRepository people) {

  Collection<Person> aggregates =
    people.findByLastname("Matthews", Person.class);

  Collection<NamesOnly> aggregates =
    people.findByLastname("Matthews", NamesOnly.class);
}
```

4. Query by Example(ps://不知道怎么翻译，范例查询？？)

4.1 Introduction(介绍)

Buery by Example(QBE) 是一种用户友好的查询技术。它可以动态查询创建，并且不需要写包含着字段名称的查询。事实上，QBE不需要使用具体的存储查询语言。 

4.2 Usage(使用)

QBE API 由三部分构成：
- Probe：就是实际的Domain实例。
- ExampleMatcher ：ExampleMatcher带有怎么匹配属性的细节信息。可以被多个Examples复用
- Example：Example 由Probe和ExampleMatcher组成。它用来构建查询。 

QBE实用于很多场景，但也有一些限制:  
实用场景：   
  - 使用一组静态或者动态的约束(条件)来查询
  - 重构domain对象时，不需要担心破坏已经存在的查询
  - 不依赖底层的数据村粗API

限制:  
  - 不支持内嵌/分组属性约束，比如firstname = ?0 or (firstname = ?1 and lastname = ?2)
  - 仅支持字符串 开始/包含/结束/正则 匹配和其他属性的严格匹配  

开始QBE之前，先创建一个domain对象和你自己的repository。
Sample Person object
```java
public class Person {

  @Id
  private String id;
  private String firstname;
  private String lastname;
  private Address address;

  // … getters and setters omitted
}
```
这是一个简单的domain对象，你可使用它来创建Example。 默认，属性的值为null时会被忽略，字符串则使用具体的存储默认匹配方式。Example即可通过of工厂方法构建，也可以使用ExampleMatcher。 Example是不可变对象。   
Simple Example
```java
Person person = new Person();          //创建新的domain对象
person.setFirstname("Dave");      //设置用于查询的属性   

Example<Person> example = Example.of(person);    //构建Example      
```
现在，让你的repository接口继承QueryByExampleExecutor<T>，就能在你的repository中使用QBE了。 
The QueryByExampleExecutor
```java
public interface QueryByExampleExecutor<T> {

  <S extends T> S findOne(Example<S> example);

  <S extends T> Iterable<S> findAll(Example<S> example);

  // … more functionality omitted.
}
```

4.3 Example mathers (Example匹配器)

Example不限于默认设置，你能够使用ExampleMatcher指定自己的匹配设定。 
xample matcher with customized matching
```java
Person person = new Person();   //创建domain对象实例                 
person.setFirstname("Dave");         //设置属性                  

//构建匹配器，并设置”忽略lastname属性“，”包含null值“， ”使用字符串后缀匹配“ . 
ExampleMatcher matcher = ExampleMatcher.matching()     
  .withIgnorePaths("lastname")                         
  .withIncludeNullValues()                             
  .withStringMatcherEnding();                          


Example<Person> example = Example.of(person, matcher);  //使用domain 对象和ExampleMatcher 构建Example。
```

默认，ExampleMatcher会期望probe所有值都匹配(理解为查询语句的and关系)。如果你想匹配任一一个值，使用ExampleMatcher.matchingAny().

你可以为个别的属性(比如"firstname" 和 "lastname",内嵌属性 "address.city")设定匹配。
Configuring matcher options
```java
ExampleMatcher matcher = ExampleMatcher.matching()
  .withMatcher("firstname", endsWith())
  .withMatcher("lastname", startsWith().ignoreCase());
}
```
另一种配置matcher风格是使用java8的lambdas。
Configuring matcher options with lambdas
```java
ExampleMatcher matcher = ExampleMatcher.matching()
  .withMatcher("firstname", match -> match.endsWith())
  .withMatcher("firstname", match -> match.startsWith());
}
```
Example会合并ExampleMatcher级别的默认匹配设定和属性级别的特殊匹配设定来构建查询。 ExampleMatcher级别的匹配设定会被属性继承，除非在属性上有特殊设定。属性上的设定比ExampleMatcher级别的优先级高。  

 Scope of ExampleMatcher settings

| Setting              | Scope                              |
| -------------------- | ---------------------------------- |
| Null-handling        | `ExampleMatcher`                   |
| String matching      | `ExampleMatcher` and property path |
| Ignoring properties  | Property path                      |
| Case sensitivity     | `ExampleMatcher` and property path |
| Value transformation | Property path                      |

5. Auditing (审计) (ps://待老夫实战一把了补充)

5.1 Basics(基础)

5.1.1 Annotation based auditing metadata基于注解的审计元数据

5.1.2  Interface-based auditing metadata(基于接口的审计元信息)

5.1.3 AuditorAware (审计员感知)



附录:

参考源文档。 


