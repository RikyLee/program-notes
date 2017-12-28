---
title: Windows下Tomcat多实例配置图解
date: 2016-3-20 16:30:00
tags: 
  - tomcat 
categories:
  - tomcat
---

- 官网下载免安装版的Tomcat ，以apache-tomcat-8.0.26 为例。

- 解压到随便一个盘中，例如我解压到了D盘test目录下，在test目录下载新建两个文件夹，名字随便取，暂命名为tomcat1，tomcat2，现在test目录下有如下文件。

  ![](http://i.imgur.com/Hbloujh.png)

- 进入apache-tomcat-8.0.26目录，有如下图文件夹。

  ![](http://i.imgur.com/FRKlWdQ.png)

- 复制其中的bin、conf、logs、temp、webapps、work目录到tomcat1，tomcat2中，tomcat1和tomcat2 中应该有如下文件，删除bin目录下的文件：

  ![](http://i.imgur.com/KayBJXZ.png)

- 分别修改tomcat1和tomcat2的conf目录下的server.xml文件中的shutdown、http、AJP 访问端口号，端口号可以随意修个，但是不能与其他服务监听端口相同，以下为默认配置。

  ![](http://i.imgur.com/0pTcVag.png)

  ![](http://i.imgur.com/ECXyU3l.png)

  ![](http://i.imgur.com/ALnimXt.png)

  tomcat1中修改之后的配置

    ![](http://i.imgur.com/6RIFGOW.png)

    ![](http://i.imgur.com/RtCGi9f.png)

    ![](http://i.imgur.com/EPVUng3.png)

  tomcat2中修改之后的配置

    ![](http://i.imgur.com/GC56Hmg.png)

    ![](http://i.imgur.com/a4T8T3E.png)

    ![](http://i.imgur.com/2fMN3tk.png)

  修改完成之后，分别启动tomcat即可。

- 在tomcat1\bin中新建一个文件，编辑文件输入以下内容：

  ```
  rem startup.bat
  set CATALINA_BASE=D:\test\tomcat1
  for %%x in ("%CATALINA_BASE%") do set CATALINA_BASE=%%~sx
  set CATALINA_HOME=D:\test\apache-tomcat-8.0.26
  for %%x in ("%CATALINA_HOME%") do set CATALINA_HOME=%%~sx
  call %CATALINA_HOME%\bin\startup.bat
  ```

  其中CATALINA_BASE表示tomcat1所在路径，CATALINA_HOME表示apache-tomcat-8.0.26所在路径，for的用途为如果所在路径有空格，用作空格处理，保存文件为后缀名为bat的文件，例如 start.bat,这个文件的用于启动tomcat1服务.

- 再次在tomcat1\bin中新建一个文件，编辑文件输入以下内容：

  ```
  rem shutdown.bat
  set CATALINA_BASE=D:\test\tomcat1
  for %%x in ("%CATALINA_BASE%") do set CATALINA_BASE=%%~sx
  set CATALINA_HOME=D:\test\apache-tomcat-8.0.26
  for %%x in ("%CATALINA_HOME%") do set CATALINA_HOME=%%~sx
  call %CATALINA_HOME%\bin\shutdown.bata
  ```

  保存文件为后缀名为bat的文件，例如 shutdown.bat,这个文件的用于关闭tomcat1服务.

- 同理，在tomcat2 \bin中建立相应的文件，修改CATALINA_BASE 为tomcat2 所在路径。

- 修改完成之后，命令行分别进入各个tomcat的bin目录下早到 start.bat文件，运行即可。

- 如果需要把以上三个tomcat以服务的方式启动，需要调用apache-tomcat-8.0.26\bin下service.bat文件写入windows 服务。

- tomcat1\bin中新建一个文件

  ```
  set JAVA_HOME=E:\dev\Java\jdk1.7.0_80
  set CATALINA_BASE=D:\test\tomcat1
  set CATALINA_HOME=D:\test\apache-tomcat-8.0.26
  %CATALINA_HOME%\bin\service.bat install "tomcat1"
  ```
  tomcat1为服务名。其他类似。

- 如果想要修个 JVM大小，可以在D:\test\apache-tomcat-8.0.26\bin\service.bat 中找到

  ![](http://i.imgur.com/NLKnSwd.png) 

  调整区中的 JvmMs 和JvmMx的大小为所需大小即可。