---
title: Spring Boot MongoDB多数据源配置
date: 2018/3/21 17:07:00
tags: 
  - MongoDB
  - Spring Boot  
categories:
  - Java
---

## 多数据源配置

- 创建一个基础的Spring Boot项目

- 在pom.xml中引入两个依赖

  ```xml
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-mongodb</artifactId>
  </dependency>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-configuration-processor</artifactId>
    <optional>true</optional>
  </dependency>
  ```

  其中spring-boot-starter-data-mongodb，是Spring Boot封装的对MongoDB自动化配置的实现，spring-boot-configuration-processor是因为在使用ConfigurationProperties注解的使用自定义配置。

- 在src/main/resources/application.properties中添加以下配置文件

  ```properties
  #Authentication database1
  spring.data.mongodb.userdb1.authentication-database=userdb1
  # Database name.
  spring.data.mongodb.userdb1.database=userdb1
  # Mongo server host.
  spring.data.mongodb.userdb1.host=mongo.rikylee.com
  # Mongo server port.
  spring.data.mongodb.userdb1.port=27020
  # Login password of the mongo server.
  spring.data.mongodb.userdb1.password=userdb1
  # Login user of the mongo server.
  spring.data.mongodb.userdb1.username=userdb1


  #Authentication database2
  spring.data.mongodb.userdb2.authentication-database=userdb2
  # Database name.
  spring.data.mongodb.userdb2.database=userdb2
  # Mongo server host.
  spring.data.mongodb.userdb2.host=mongo.rikylee.com
  # Mongo server port.
  spring.data.mongodb.userdb2.port=27020
  # Login password of the mongo server.
  spring.data.mongodb.userdb2.password=userdb2
  # Login user of the mongo server.
  spring.data.mongodb.userdb2.username=userdb2
  ```

- 创建抽象类AbstractMongoConfig

  ```java
  package com.riky.mongo.config;

  import com.mongodb.MongoClient;
  import com.mongodb.MongoClientOptions;
  import com.mongodb.MongoCredential;
  import com.mongodb.ServerAddress;
  import org.springframework.data.mongodb.MongoDbFactory;
  import org.springframework.data.mongodb.core.MongoTemplate;
  import org.springframework.data.mongodb.core.SimpleMongoDbFactory;
  import org.springframework.data.mongodb.core.mapping.MongoMappingContext;

  /**
  * @author Riky Li
  * @create 2018-03-21 13:52:28
  * @desciption:
  */
  public abstract class AbstractMongoConfig {
      //定义相关连接数据库参数
      private String host, database, username, password;
      private int port;

      public String getHost() {
          return host;
      }

      public void setHost(String host) {
          this.host = host;
      }

      public String getDatabase() {
          return database;
      }

      public void setDatabase(String database) {
          this.database = database;
      }

      public String getUsername() {
          return username;
      }

      public void setUsername(String username) {
          this.username = username;
      }

      public String getPassword() {
          return password;
      }

      public void setPassword(String password) {
          this.password = password;
      }

      public int getPort() {
          return port;
      }

      public void setPort(int port) {
          this.port = port;
      }
      /**
      * 创建MongoDbFactory，不同数据源继承该方法创建对应的MongoDbFactory。
      * @return
      * @throws Exception
      */
      public MongoDbFactory mongoDbFactory() throws Exception {
          ServerAddress serverAddress = new ServerAddress(host, port);
          MongoCredential mongoCredential = MongoCredential.createCredential(username, database, password.toCharArray());
          MongoClientOptions options = MongoClientOptions.builder()
                  .connectionsPerHost(100)
                  .socketTimeout(30000)
                  .connectTimeout(3000)
                  .build();
          return new SimpleMongoDbFactory(new MongoClient(serverAddress, mongoCredential, options), database);
      }
      /**
      * 抽象方法，用于返回MongoTemplate
      * @param context
      * @return
      * @throws Exception
      */
      abstract public MongoTemplate getMongoTemplate(MongoMappingContext context) throws Exception;
  }

  ```

- 创建多个数据源对象

  创建UserDb1数据源

  ```java
  package com.riky.mongo.config;

  import org.springframework.boot.context.properties.ConfigurationProperties;
  import org.springframework.context.annotation.Bean;
  import org.springframework.context.annotation.Configuration;
  import org.springframework.context.annotation.Primary;
  import org.springframework.data.mongodb.core.MongoTemplate;
  import org.springframework.data.mongodb.core.convert.DefaultDbRefResolver;
  import org.springframework.data.mongodb.core.convert.DefaultMongoTypeMapper;
  import org.springframework.data.mongodb.core.convert.MappingMongoConverter;
  import org.springframework.data.mongodb.core.mapping.MongoMappingContext;

  /**
  * @author Riky Li
  * @create 2018-03-21 14:04:17
  * @desciption:
  */
  @Configuration
  @ConfigurationProperties(prefix = "spring.data.mongodb.userdb1")
  public class UserDb1Config extends AbstractMongoConfig {
      //设置为默认数据源，如果定义mongoTemplate的时候未指定数据源，将默认为此数据源
      @Primary
      @Override
      public @Bean(name = "userDb1MongoTemplate") MongoTemplate getMongoTemplate(MongoMappingContext context) throws Exception {
          //去除保存实体时，spring data mongodb 自动添加的_class字段
          MappingMongoConverter converter = new MappingMongoConverter(new DefaultDbRefResolver(mongoDbFactory()), context);
          converter.setTypeMapper(new DefaultMongoTypeMapper(null));

          MongoTemplate mongoTemplate = new MongoTemplate(mongoDbFactory(), converter);

          return mongoTemplate;
      }
  }

  ```

  创建UserDb2数据源

  ```java
  package com.riky.mongo.config;

  import org.springframework.boot.context.properties.ConfigurationProperties;
  import org.springframework.context.annotation.Bean;
  import org.springframework.context.annotation.Configuration;
  import org.springframework.data.mongodb.core.MongoTemplate;
  import org.springframework.data.mongodb.core.convert.DefaultDbRefResolver;
  import org.springframework.data.mongodb.core.convert.DefaultMongoTypeMapper;
  import org.springframework.data.mongodb.core.convert.MappingMongoConverter;
  import org.springframework.data.mongodb.core.mapping.MongoMappingContext;

  /**
  * @author Riky Li
  * @create 2018-03-21 14:05:02
  * @desciption:
  */
  @Configuration
  @ConfigurationProperties(prefix = "spring.data.mongodb.userdb2")
  public class UserDb2Config extends AbstractMongoConfig {

      @Override
      public @Bean(name = "userDb2MongoTemplate") MongoTemplate getMongoTemplate(MongoMappingContext context) throws Exception {
           //去除保存实体时，spring data mongodb 自动添加的_class字段
          MappingMongoConverter converter = new MappingMongoConverter(new DefaultDbRefResolver(mongoDbFactory()), context);
          converter.setTypeMapper(new DefaultMongoTypeMapper(null));

          MongoTemplate mongoTemplate = new MongoTemplate(mongoDbFactory(), converter);

          return mongoTemplate;
      }
  }

  ```

- 在UserDb1和UserDb2中各创建一个collention名为user，建立相应的实体映射

  ```java
  package com.riky.mongo.entity;

  import org.springframework.data.annotation.Id;
  import org.springframework.data.mongodb.core.mapping.Document;

  /**
  * @author Riky Li
  * @create 2018-03-21 14:12:53
  * @desciption:
  */
  @Document(collection = "user")
  public class User {
      @Id
      private String id;
      private String username;
      private String email;
      private String passwd;

      public String getId() {
          return id;
      }

      public void setId(String id) {
          this.id = id;
      }

      public String getUsername() {
          return username;
      }

      public void setUsername(String username) {
          this.username = username;
      }

      public String getEmail() {
          return email;
      }

      public void setEmail(String email) {
          this.email = email;
      }

      public String getPasswd() {
          return passwd;
      }

      public void setPasswd(String passwd) {
          this.passwd = passwd;
      }
  }

  ```

- 使用MongoTemplate连接mongo数据库

  ```java
  package com.riky.mongo.dao;

  import com.riky.mongo.entity.User;
  import org.springframework.beans.factory.annotation.Autowired;
  import org.springframework.beans.factory.annotation.Qualifier;
  import org.springframework.data.mongodb.core.MongoTemplate;
  import org.springframework.data.mongodb.core.query.Criteria;
  import org.springframework.data.mongodb.core.query.Query;
  import org.springframework.stereotype.Repository;

  /**
  * @author Riky Li
  * @create 2018-03-21 14:29:00
  * @desciption:
  */
  @Repository
  public class UserDao {
      @Autowired
      @Qualifier(value = "userDb1MongoTemplate")
      private MongoTemplate userDb1MongoTemplate;

      @Autowired
      @Qualifier(value = "userDb2MongoTemplate")
      private MongoTemplate userDb2MongoTemplate;

      public User findUserById_UserDb1(String id) {
          User user = userDb1MongoTemplate.findOne(new Query(Criteria.where("id").is(id)), User.class);
          return user;
      }
      public User findUserById_UserDb2(String id) {
          User user = userDb2MongoTemplate.findOne(new Query(Criteria.where("id").is(id)), User.class);
          return user;
      }
  }

  ```

- 创建Service，进行业务调用

  ```java
  package com.riky.mongo.service;

  import com.riky.mongo.dao.UserDao;
  import com.riky.mongo.entity.User;
  import org.springframework.beans.factory.annotation.Autowired;
  import org.springframework.stereotype.Service;

  /**
  * @author Riky Li
  * @create 2018-03-21 14:36:19
  * @desciption:
  */
  @Service
  public class UserService {
      @Autowired
      private UserDao dao;

      public User findUserById_UserDb1(String id){
          return  dao.findUserById_UserDb1(id);
      }

      public User findUserById_UserDb2(String id){
          return  dao.findUserById_UserDb2(id);
      }

  }

  ```

- 创建单元测试用例读取用户信息

  ```java
    package com.riky.mongo.service;

    import com.riky.mongo.entity.User;
    import org.junit.Test;
    import org.junit.runner.RunWith;
    import org.springframework.beans.factory.annotation.Autowired;
    import org.springframework.boot.test.context.SpringBootTest;
    import org.springframework.test.context.junit4.SpringRunner;

    /**
    * @author Riky Li
    * @create 2018-03-21 15:33:18
    * @desciption:
    */
    @RunWith(SpringRunner.class)
    @SpringBootTest
    public class UserServiceTest {
        @Autowired
        private UserService userService;

        @Test
        public void findUserById_UserDb1Test() {
          User user = userService.findUserById_UserDb1("58d6735d84ae4de8b2ab42b3")
          System.out.println("user: " + user.getUsername());
        }

        @Test
        public void findUserById_UserDb2Test() {
           User user = userService.findUserById_UserDb2("58d8694f84aedcc7676738e5");
           System.out.println("user: " + user.getUsername());
        }
    }
    ```