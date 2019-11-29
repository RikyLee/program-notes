# Tomcat原理

## Tomcat顶层架构

- 最外层为Server，其为Service提供生存环境，一个Server包含至少一个Service
- Service主要包含两个部分：Connector和Container
  - Connector用于处理连接相关的事情，并提供Socket与接受请求并将请求封装成Request和Response来具体处理
  - Container用于封装和管理Servlet，以及具体处理Request请求

一个请求发送到Tomcat之后，首先经过Service然后会交给我们的Connector，Connector用于接收请求并将接收的请求封装为Request和Response来具体处理，Request和Response封装完之后再交由Container进行处理，Container处理完请求之后再返回给Connector，最后在由Connector通过Socket将处理的结果返回给客户端，这样整个请求的就处理完了！

Connector最底层使用的是Socket来进行连接的，Request和Response是按照HTTP协议来封装的，所以Connector同时需要实现TCP/IP协议和HTTP协议！

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Server port="8005" shutdown="SHUTDOWN">
  <Listener className="org.apache.catalina.startup.VersionLoggerListener" />

  <Listener className="org.apache.catalina.core.AprLifecycleListener" SSLEngine="on" />
  <Listener className="org.apache.catalina.core.JreMemoryLeakPreventionListener" />
  <Listener className="org.apache.catalina.mbeans.GlobalResourcesLifecycleListener" />
  <Listener className="org.apache.catalina.core.ThreadLocalLeakPreventionListener" />

  <GlobalNamingResources>
    <Resource name="UserDatabase" auth="Container"
              type="org.apache.catalina.UserDatabase"
              description="User database that can be updated and saved"
              factory="org.apache.catalina.users.MemoryUserDatabaseFactory"
              pathname="conf/tomcat-users.xml" />
  </GlobalNamingResources>

  <Service name="Catalina">
    <Connector port="8080" protocol="HTTP/1.1"
               connectionTimeout="20000"
               redirectPort="8443" />
    <Connector port="8009" protocol="AJP/1.3" redirectPort="8443" />
    <Engine name="Catalina" defaultHost="localhost">
      <Realm className="org.apache.catalina.realm.LockOutRealm">
        <Realm className="org.apache.catalina.realm.UserDatabaseRealm"
               resourceName="UserDatabase"/>
      </Realm>

      <Host name="localhost"  appBase="webapps" unpackWARs="true" autoDeploy="true">
        <Valve className="org.apache.catalina.valves.AccessLogValve" directory="logs"
               prefix="localhost_access_log" suffix=".txt"
               pattern="%h %l %u %t &quot;%r&quot; %s %b" />
      </Host>
    </Engine>
  </Service>
</Server>
```

## Connector架构分析

Connector就是使用ProtocolHandler来处理请求的，不同的ProtocolHandler代表不同的连接类型，比如：Http11Protocol使用的是普通Socket来连接的，Http11NioProtocol使用的是NioSocket来连接的。其中ProtocolHandler由包含了三个部件：Endpoint、Processor、Adapter。

- Endpoint用来处理底层Socket的网络连接，Processor用于将Endpoint接收到的Socket封装成Request，Adapter用于将Request交给Container进行具体的处理

- Endpoint由于是处理底层的Socket网络连接，因此Endpoint是用来实现TCP/IP协议的，而Processor用来实现HTTP协议的，Adapter将请求适配到Servlet容器进行具体的处理

- Endpoint的抽象实现AbstractEndpoint里面定义的Acceptor和AsyncTimeout两个内部类和一个Handler接口。Acceptor用于监听请求，AsyncTimeout用于检查异步Request的超时，Handler用于处理接收到的Socket，在内部调用Processor进行处理

## Container架构分析

Container用于封装和管理Servlet，以及具体处理Request请求，在Connector内部包含了4个子容器,作用分别是

- Engine：引擎，用来管理多个站点，一个Service最多只能有一个Engine
- Host：代表一个站点，也可以叫虚拟主机，通过配置Host就可以添加站点
- Context：代表一个应用程序，对应着平时开发的一套程序，或者一个WEB-INF目录以及下面的web.xml文件
- Wrapper：每一Wrapper封装着一个Servlet

Context和Host的区别是Context表示一个应用，我们的Tomcat中默认的配置下webapps下的每一个文件夹目录都是一个Context，其中ROOT目录中存放着主应用，其他目录存放着子应用，而整个webapps就是一个Host站点。

## Container如何处理请求的

Container处理请求是使用Pipeline-Value管道来处理，Pipeline-Value是责任链模式，责任链模式是指在一个请求处理的过程中有很多处理者依次对请求进行处理，每个处理者负责做自己相应的处理，处理完之后将处理后的请求返回，再让下一个处理着继续处理。

Pipeline-Value使用的责任链模式和普通的责任链模式有些不同，区别主要有以下两点：

- 每个Pipeline都有特定的Value，而且是在管道的最后一个执行，这个Value叫做BaseValue，BaseValue是不可删除的
- 在上层容器的管道的BaseValue中会调用下层容器的管道

Container包含四个子容器，而这四个子容器对应的BaseValue分别为StandardEngineValue、StandardHostValue、StandardContextValue、StandardWrapperValue。

Pipeline的处理流程

- Container在接收到请求后会首先调用最顶层容器的Pipeline来处理，这里的最顶层容器的Pipeline就是EnginePipeline（Engine的管道）；

- 在Engine的管道中依次会执行EngineValue1、EngineValue2等等，最后会执行StandardEngineValue，在StandardEngineValue中会调用Host管道，然后再依次执行Host的HostValue1、HostValue2等，最后在执行StandardHostValue，然后再依次调用Context的管道和Wrapper的管道，最后执行到StandardWrapperValue。

- 当执行到StandardWrapperValue的时候，会在StandardWrapperValue中创建FilterChain，并调用其doFilter方法来处理请求，这个FilterChain包含着我们配置的与请求相匹配的Filter和Servlet，其doFilter方法会依次调用所有的Filter的doFilter方法和Servlet的service方法，这样请求就得到了处理！

- 当所有的Pipeline-Value都执行完之后，并且处理完了具体的请求，这个时候就可以将返回的结果交给Connector了，Connector在通过Socket的方式将结果返回给客户端。