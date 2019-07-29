---
title: spring data jpa 使用
date: 2019-07-29 11:39:32
tags:
---
#### spring data jpa介绍

首先了解JPA是什么？
JPA(Java Persistence API)是Sun官方提出的Java持久化规范。它为Java开发人员提供了一种对象/关联映射工具来管理Java应用中的关系数据。他的出现主要是为了简化现有的持久化开发工作和整合ORM技术，结束现在Hibernate，TopLink，JDO等ORM框架各自为营的局面。值得注意的是，JPA是在充分吸收了现有Hibernate，TopLink，JDO等ORM框架的基础上发展而来的，具有易于使用，伸缩性强等优点。从目前的开发社区的反应上看，JPA受到了极大的支持和赞扬，其中就包括了Spring与EJB3.0的开发团队。

注意:JPA是一套规范，不是一套产品，那么像Hibernate,TopLink,JDO他们是一套产品，如果说这些产品实现了这个JPA规范，那么我们就可以叫他们为JPA的实现产品。

#### spring data jpa
Spring Data JPA 是 Spring 基于 ORM 框架、JPA 规范的基础上封装的一套JPA应用框架，可使开发者用极简的代码即可实现对数据的访问和操作。它提供了包括增删改查等在内的常用功能，且易于扩展！学习并使用 Spring Data JPA 可以极大提高开发效率！

#### 添加spring-data-jpa的支持
```xml
<!--引入JPA的依赖关系-->
<dependency>
 <groupId>org.springframework.boot</groupId>
 <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>
```

spring data jpa让我们解脱了DAO层的操作，基本上所有CRUD都可以依赖于它来实现

 ### 基本查询
基本查询也分为两种，一种是spring data默认已经实现，一种是根据查询的方法来自动解析成SQL。

预先生成方法
spring data jpa 默认预先生成了一些基本的CURD的方法，例如：增、删、改等等

- 实体类
```java
@Entity 
@Table(name = "t_user") 
public class User implements Serializable { 

@Id
private Long id; 
private Integer balance; 
 
// 此处省略 getter 和 setter 方法。
}
```

- 1 继承JpaRepository
```java
public interface UserRepository extends JpaRepository<User, Long> {
}
```
- 2 使用默认方法
```java
@Test
public void testBaseQuery() throws Exception {
    User user=new User();
    userRepository.findAll();
    userRepository.findOne(1l);
    userRepository.save(user);
    userRepository.delete(user);
    userRepository.count();
    userRepository.exists(1l);
    // ...
}
```
就不解释了根据方法名就看出意思来

自定义简单查询

自定义的简单查询就是根据方法名来自动生成SQL，主要的语法是
`findXXBy,readAXXBy,queryXXBy,countXXBy, getXXBy`后面跟属性名称：

也使用一些加一些关键字 `And`、 `Or`
```java
User findByUserNameOrEmail(String username, String email);
```
修改、删除、统计也是类似语法
```java
Long deleteById(Long id);

Long countByUserName(String userName)
```
基本上SQL体系中的关键词都可以使用，例如：`LIKE、 OrderBy`。
```java
List<User> findByEmailLike(String email);
    
List<User> findByUserNameOrderByEmailDesc(String email);
```

具体的关键字，使用方法和生产成SQL如下表所示
![1](https://upload-images.jianshu.io/upload_images/11462107-50f19ae61cc1d406.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![2](https://upload-images.jianshu.io/upload_images/11462107-fc8ea30aebbf07e8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 复杂查询

在实际的开发中我们需要用到分页、删选、连表等查询的时候就需要特殊的方法或者自定义SQL

分页查询
分页查询在实际使用中非常普遍了，`spring data jpa` 已经帮我们实现了分页的功能，在查询的方法中，需要传入参数 `Pageable`
,当查询中有多个参数的时候 `Pageable `建议做为最后一个参数传入
```java
Page<User> findALL(Pageable pageable);
    
Page<User> findByUserName(String userName,Pageable pageable);
```
`Pageable` 是spring封装的分页实现类，使用的时候需要传入页数、每页条数和排序规则
```java
@Test
public void testPageQuery() throws Exception {
    int page=1,size=10;
    Sort sort = new Sort(Direction.DESC, "id");
    Pageable pageable = new PageRequest(page, size, sort);
    userRepository.findALL(pageable);
    userRepository.findByUserName("testName", pageable);
}
```
- 限制查询

有时候我们只需要查询前N个元素，或者支取前一个实体。
 ```java
ser findFirstByOrderByLastnameAsc();

User findTopByOrderByAgeDesc();

Page<User> queryFirst10ByLastname(String lastname, Pageable pageable);

List<User> findFirst10ByLastname(String lastname, Sort sort);

List<User> findTop10ByLastname(String lastname, Pageable pageable);
```
- 自定义SQL查询

其实Spring data 觉大部分的SQL都可以根据方法名定义的方式来实现，但是由于某些原因我们想使用自定义的SQL来查询，spring data也是完美支持的；在SQL的查询方法上面使用 `@Query`注解，如涉及到删除和修改在需要加上`@Modifying`.也可以根据需要添加 `@Transactional `对事物的支持，查询超时的设置等
```java
@Modifying
@Query("update User u set u.userName = ?1 where c.id = ?2")
int modifyByIdAndUserId(String  userName, Long id);
    
@Transactional
@Modifying
@Query("delete from User where id = ?1")
void deleteByUserId(Long id);
  
@Transactional(timeout = 10)
@Query("select u from User u where u.emailAddress = ?1")
    User findByEmailAddress(String emailAddress);
```


