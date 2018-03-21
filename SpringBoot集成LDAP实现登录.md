---
title: Spring Boot集成LDAP实现登录
date: 2018/3/21 15:42:00
tags: 
  - LDAP
  - Spring Boot  
categories:
  - Java
---

## LDAP简介

- LDAP（轻量级目录访问协议，Lightweight Directory Access Protocol)是实现提供被称为目录服务的信息服务。

  目录服务是一种特殊的数据库系统，其专门针对读取，浏览和搜索操作进行了特定的优化。目录一般用来包含描述性的，基于属性的信息并支持精细复杂的过滤能力。目录一般不支持通用数据库针对大量更新操作操作需要的复杂的事务管理或回卷策略。而目录服务的更新则一般都非常简单。这种目录可以存储包括个人信息、web链结、jpeg图像等各种信息。为了访问存储在目录中的信息，就需要使用运行在TCP/IP 之上的访问协议—LDAP。

  LDAP目录中的信息是是按照树型结构组织，具体信息存储在条目(entry)的数据结构中。条目相当于关系数据库中表的记录；条目是具有区别名DN （Distinguished Name）的属性（Attribute），DN是用来引用条目的，DN相当于关系数据库表中的关键字（Primary Key）。属性由类型（Type）和一个或多个值（Values）组成，相当于关系数据库中的字段（Field）由字段名和数据类型组成，只是为了方便检索的需要，LDAP中的Type可以有多个Value，而不是关系数据库中为降低数据的冗余性要求实现的各个域必须是不相关的。LDAP中条目的组织一般按照地理位置和组织关系进行组织，非常的直观。LDAP把数据存放在文件中，为提高效率可以使用基于索引的文件数据库，而不是关系数据库。类型的一个例子就是mail，其值将是一个电子邮件地址。

  LDAP的信息是以树型结构存储的，在树根一般定义国家(c=CN)或域名(dc=com)，在其下则往往定义一个或多个组织 (organization)(o=Acme)或组织单元(organizational units) (ou=People)。一个组织单元可能包含诸如所有雇员、大楼内的所有打印机等信息。此外，LDAP支持对条目能够和必须支持哪些属性进行控制，这是有一个特殊的称为对象类别(objectClass)的属性来实现的。该属性的值决定了该条目必须遵循的一些规则，其规定了该条目能够及至少应该包含哪些属性。例如：inetorgPerson对象类需要支持sn(surname)和cn(common name)属性，但也可以包含可选的如邮件，电话号码等属性。

## LDAP简称对应

  ```text
  o：organization（组织-公司）
  ou：organization unit（组织单元-部门）
  c：countryName（国家）
  dc：domainComponent（域名）
  sn：surname（姓氏）
  cn：common name（常用名称）
  sAMAccountName:英文名(RTX账号,唯一)
  userPrincipalName:登陆用户名 和 英文名一致
  ```

以上内容参考自：[LDAP快速入门](https://www.cnblogs.com/obpm/archive/2010/08/28/1811065.html)

## 登录实现

- 创建一个基础的Spring Boot项目
- 在pom.xml中引入Spring Ldap依赖

  ```xml
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-data-ldap</artifactId>
    </dependency>
  ```

  spring-boot-starter-data-ldap是Spring Boot封装的对LDAP自动化配置的实现，它是基于spring-data-ldap来对LDAP服务端进行具体操作的。

- 在src/main/resources/application.properties中添加以下配置文件

  ```properties
  # ldap config
  #连接地址
  spring.ldap.urls=ldap://ldap.rikylee.com:389
  spring.ldap.username=rikylee
  spring.ldap.password=rikylee
  spring.ldap.base=ou=employee,dc=rikylee,dc=com
  spring.ldap.domainName=@rikylee.com
  spring.ldap.referral=follow
  ```

  以上参数为ldap的相关连接配置。

- 建立一个用户对象，用于实体关系的映射

  ```java
  package com.riky.ldap.entity;

  import java.util.List;

  /**
  * @author Riky Li
  */

  public class Person {
    //用户名
    private String name;
    //登录名
    private String sAMAccountName;
    //所属组列表
    private List<String> role;

      public String getName() {
          return name;
      }

      public void setName(String name) {
          this.name = name;
      }

      public String getsAMAccountName() {
          return sAMAccountName;
      }

      public void setsAMAccountName(String sAMAccountName) {
          this.sAMAccountName = sAMAccountName;
      }

      public List<String> getRole() {
          return role;
      }

      public void setRole(List<String> role) {
          this.role = role;
      }

      @Override
      public String toString() {

          StringBuffer stringBuffer = new StringBuffer();
          stringBuffer.append("name:"+name+",");
          stringBuffer.append("sAMAccountName:"+sAMAccountName);
          if(role!=null && role.size()>0){
              stringBuffer.append(",role:"+String.join(",",role));
          }

          return stringBuffer.toString();
      }
  }
  ```

- 建立一个Service，实现登录授权，已经用户信息的查询

  ```java
  package com.riky.ldap.service;

  import com.riky.ldap.entity.Person;
  import org.springframework.beans.factory.annotation.Autowired;
  import org.springframework.beans.factory.annotation.Value;
  import org.springframework.ldap.core.AttributesMapper;
  import org.springframework.ldap.core.LdapTemplate;
  import org.springframework.ldap.support.LdapUtils;
  import org.springframework.stereotype.Service;
  import org.springframework.util.StringUtils;

  import javax.naming.NamingException;
  import javax.naming.directory.Attributes;
  import javax.naming.directory.DirContext;
  import java.util.ArrayList;
  import java.util.List;

  import static org.springframework.ldap.query.LdapQueryBuilder.query;
  /**
  * @author Riky Li
  */
  @Service
  public class LdapPersonService {
      @Autowired
      private LdapTemplate ldapTemplate;

      @Value("${spring.ldap.domainName}")
      private String ldapDomainName;

      public Person findByUsername(String username, String password) throws NamingException {
         //这里注意用户名加域名后缀  userDn格式：abcd@rikylee.com
          String userDn = username + ldapDomainName;
          //使用用户名、密码验证域用户
          DirContext ctx = ldapTemplate.getContextSource().getContext(userDn, password);
          //如果验证成功根据sAMAccountName属性查询用户名和用户所属的组
          Person person = ldapTemplate.search(query().where("objectclass").is("person").and("sAMAccountName").is(username), new AttributesMapper<Person>() {
              @Override
              public Person mapFromAttributes(Attributes attributes) throws NamingException {
                  Person person = new Person();
                  person.setName(attributes.get("cn").get().toString());
                  person.setsAMAccountName(attributes.get("sAMAccountName").get().toString());

                  String memberOf = attributes.get("memberOf").toString().replace("memberOf: ", "");
                  List<String> list = new ArrayList<>();
                  String[] roles = memberOf.split(",");
                  for (String role : roles) {
                      if (StringUtils.startsWithIgnoreCase(role.trim(), "CN=")) {
                          list.add(role.trim().replace("CN=", ""));
                      }
                  }
                  person.setRole(list);

                  return person;
              }
          }).get(0);
          //关闭ldap连接
          LdapUtils.closeContext(ctx);

          return person;
      }
  }
  ```

- 创建单元测试用例测试登录以及读取用户信息

  ```java
  package com.riky.ldap.service;

  import com.riky.ldap.entity.Person;
  import org.junit.Test;
  import org.junit.runner.RunWith;
  import org.springframework.beans.factory.annotation.Autowired;
  import org.springframework.boot.test.context.SpringBootTest;
  import org.springframework.test.context.junit4.SpringRunner;

  /**
  * @author Riky Li
  * @create 2018-03-20 10:33:18
  * @desciption:
  */
  @RunWith(SpringRunner.class)
  @SpringBootTest
  public class LdapPersonServiceTest {
      @Autowired
      private LdapPersonService ldapPersonService;

      @Test
      public void findByUsernameTest() {
          try {
              Person person = ldapPersonService.findByUsername("rikylee", "rikylee");
              System.out.println("person: " + person.toString());
          } catch (Exception e) {
              e.printStackTrace();
          }
      }
  }
  ```

- 如出现以下Exception

  ```text
  error code 49

  ==========================================================================

  javax.naming.AuthenticationException: [LDAP: error code 49 - 80090308: LdapErr: DSID-0C090334, comment: AcceptSecurityContext error, data 52e, vece

  ```

  原因：用户名或密码错误
