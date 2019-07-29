---
title: Spring Boot 结合 MongoDB 简单使用
date: 2019-07-29 11:39:48
tags:
---
### 1、什么是MongoDB ?

MongoDB 是一个基于分布式文件存储的数据库。由 C++ 语言编写。旨在为 WEB 应用提供可扩展的高性能数据存储解决方案。

MongoDB 是一个介于关系数据库和非关系数据库之间的产品，是非关系数据库当中功能最丰富，最像关系数据库的。

MongoDB 是由C++语言编写的，是一个基于分布式文件存储的开源数据库系统。

在高负载的情况下，添加更多的节点，可以保证服务器性能。

MongoDB 将数据存储为一个文档，数据结构由键值(key=>value)对组成。

MongoDB 文档类似于 JSON 对象。字段值可以包含其他文档，数组及文档数组。

### MongoDB和关系数据库的对比
![](https://upload-images.jianshu.io/upload_images/11462107-bb81470cbd4b4c80.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


### mongodb的增删改查

Spring Boot对各种流行的数据源都进行了封装，当然也包括了mongodb,下面给大家介绍如何在spring boot中使用mongodb：

- 1、pom包配置

pom包里面添加spring-boot-starter-data-mongodb包引用

```xml
<dependencies>
    <dependency> 
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-mongodb</artifactId>
    </dependency> 
</dependencies>
```

- 2、在application.properties中添加配置
pom包里面添加spring-boot-starter-data-mongodb包引用
```xml
<dependencies>
    <dependency> 
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-mongodb</artifactId>
    </dependency> 
</dependencies>
```

- 2、在application.properties中添加配置
```
spring.data.mongodb.uri=mongodb://name:password@localhost:27017/admin
```

2、创建数据实体

```
@Data
@ToString
public class User implements Serializable {
    private static final long serialVersionUID = -3258839839160856613L;
    @Id
    private Long id;

    private String username;
    private Integer age;

    public User(Long id, String username, Integer age) {
        this.id = id;
        this.username = username;
        this.age = age;
    }
}
```

- 实现User的数据访问对象：UserRepository
```java
public interface UserRepository extends MongoRepository<User, Long> {

    User findByUsername(String username);
    
    User findByUsernameAndAge(String username, Integer age);

    User findByAge(Integer age);
}
```

- 在单元测试中调用
```java
@RunWith(SpringRunner.class)
@SpringBootTest
public class MongodbApplicationTests {
    @Autowired
    private UserRepository userRepository;
    
    @Test
    public void test1() throws Exception {
//        User user = userRepository.findByAge(40);
//        System.out.println(user.toString());
        
        User user1 = userRepository.findByUsernameAndAge("mama", 40);
        System.out.println(user1.toString());
    }

    @Test
    public void test() throws Exception {

        // 创建三个User，并验证User总数
        userRepository.save(new User(1L, "didi", 30));
        userRepository.save(new User(2L, "mama", 40));
        userRepository.save(new User(3L, "kaka", 50));

        Assert.assertEquals(3, userRepository.findAll().size());

        // 删除一个User，再验证User总数
        User u = userRepository.findById(1L).get();
        System.out.println(u.toString());
        userRepository.delete(u);
        Assert.assertEquals(2, userRepository.findAll().size());

        // 删除一个User，再验证User总数
        u = userRepository.findByUsername("mama");
        System.out.println(u.toString());
//        userRepository.delete(u);
        Assert.assertEquals(2, userRepository.findAll().size());
    }
}
```


#### 使用 MongoTemplate  操作 

- 创建 `UserEntity `
```java
@Data
public class UserEntity implements Serializable {
        private static final long serialVersionUID = -3258839839160856613L;
        private String userName;
        private String passWord;

}
```

 - 创建实体dao的增删改查操作
dao层实现了 `UserEntity ` 对象的增删改查
```java
 @Component
public class UserEntityDao implements IUserEntityDao {

    @Autowired
    private MongoTemplate mongoTemplate;

    public void save() {
        UserEntity userEntity = new UserEntity();
        userEntity.setUserName("mengma");
        userEntity.setPassWord("33333");
        mongoTemplate.save(userEntity);
    }


    public UserEntity findByUserName(String username) {
        //  使用 query 对象 声明查询 条件
        Query query = new Query(Criteria.where("userName").is(username));
        UserEntity userEntity = mongoTemplate.findOne(query, UserEntity.class);
        return userEntity;
    }

    /**
     * 通过 userName 更新  passWord
     *
     * @param:
     * @return:
     * @auther: Tangzhiqiang
     * @date: 2019/1/14 9:21
     */
    public void updateUser(String userName, String password) {

        // 通过 query 对象 声明更新的条件
        Query query = new Query(Criteria.where("userName").is(userName));

        // 通过 update 对象 声明更新字段 值
        Update update = new Update().set("passWord", password);

        UpdateResult updateResult = mongoTemplate.updateFirst(query, update, UserEntity.class);
    }

    public void delByUsername(String username) {
        Query query = new Query(Criteria.where("userName").is(username));
        mongoTemplate.remove(query, UserEntity.class);
    }

    /**
     * 查询分页
     *
     * @param:
     * @return:
     * @auther: Tangzhiqiang
     * @date: 2019/1/14 10:42
     */
    public void findPage(int size, int page, String username) {
        // 通过 pageable 对象设置 分页条件
        Pageable pageable = new PageRequest(page - 1, size);

        // 分页具体逻辑条件
        Query query = new Query(Criteria.where("userName").is(username));
        List<UserEntity> userEntityList = mongoTemplate.find(query.with(pageable), UserEntity.class);
        System.out.println(userEntityList.toString());
    }

}

```
- 测试类
```java
@RunWith(SpringRunner.class)
@SpringBootTest
public class UserEntityDaoTest {

    @Autowired
    private UserEntityDao userEntityDao;

    @Test
    public void save() {
        userEntityDao.save();
    }

    @Test
    public void findByUserName() {
        UserEntity userEntity = userEntityDao.findByUserName("mengma");
        System.out.println(userEntity.toString());
    }

    @Test
    public void updateUser() {
        userEntityDao.updateUser("mengma", "99999999");
    }

    @Test
    public void delByUserName() {
        userEntityDao.delByUsername("mengma");
    }

    @Test
    public void findPage() {
        userEntityDao.findPage(2, 1, "mengma");
    }

}
```

- 分页查询 

```java
public void findPage(int size, int page, String username) {
        // 通过 pageable 对象设置 分页条件
        Pageable pageable = new PageRequest(page - 1, size);

        // 分页具体逻辑条件
        Query query = new Query(Criteria.where("userName").is(username));
        List<UserEntity> userEntityList = mongoTemplate.find(query.with(pageable), UserEntity.class);
        System.out.println(userEntityList.toString());
    }
```