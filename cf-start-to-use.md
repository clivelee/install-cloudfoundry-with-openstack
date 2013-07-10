
#Cloudfoundry使用手册

* Published: 2013年7月10日 上午11:20  
* CreateBy: Clive Lee 
* Version: 1.0

##1.安装cf客户端

需要预装ruby 1.9.3, 使用在cf.yml中scim中配置的管理员账号和密码来登录

```
gem install cf --no-ri --no-rdoc
cf target api.mydomain.com
cf login admin

ubuntu@ubuntu:~$ cf login admin
target: http://api.dc.cfos.org

Password> ********

Authenticating... OK
There are no spaces. You may want to create one with create-space.
```

默认创建的组织没有挂上域名,所以必须再创建一个新的组织.

```
ubuntu@ubuntu:~$ cf create-org cstnet
Creating organization cstnet... OK
Switching to organization cstnet... OK
There are no spaces. You may want to create one with create-space.

ubuntu@ubuntu:~$ cf orgs
Getting organizations... OK

name             spaces   domains       
cstnet           none     dc.cfos.org
dc.cfos.org   none     none    
```

命令行会提示需要创建一个space,运行命令`cf create-space [SPACE-NAME]` :

```
ubuntu@ubuntu:~$ cf create-space software
Creating space software... OK
Adding you as a manager... OK
Adding you as a developer... OK
Space created! Use `cf switch-space software` to target it.
```
切换到software下面:

```
ubuntu@ubuntu:~$ cf switch-space software
Switching to space software... OK

target: http://api.dc.cfos.org
organization: cstnet
space: software
```


##2.创建服务

当cf刚装好的时候,cf内部还没有任何服务的实例,可以通过create-service命令来创建数据服务. 但在创建数据服务之前需要创建访问服务的token,否则将无法创建成功:

```
ubuntu@ubuntu:~$ cf create-service-auth-token redis core --token c1oudc0wc1oudc0w
Creating service auth token... OK
ubuntu@ubuntu:~$ cf create-service-auth-token mysql core --token c1oudc0wc1oudc0w
Creating service auth token... OK
ubuntu@ubuntu:~$ cf create-service-auth-token mongodb core --token c1oudc0wc1oudc0w
Creating service auth token... OK
ubuntu@ubuntu:~$ cf service-auth-tokens
redis:
  guid: 19d5b137-4d02-4040-8dcf-021f20209135
  provider: core

mysql:
  guid: a9e9e49a-c1f4-4d9e-a0d2-2ffeba7ff54b
  provider: core

mongodb:
  guid: 90be185f-3bf7-428d-9249-500b65885855
  provider: core
```
接下来就可以创建应用了,例如创建一个mongodb的服务

```
ubuntu@ubuntu:~$ cf create-service
1: mongodb 2.2
2: mysql 5.5
3: redis 2.6
What kind?> 1

Name?> mongo-test   

1: free: Mongodb Service
Which plan?> 1

Creating service mongo-test... OK
```

如何使用刚创建的服务呢?cf中提供了一种叫做tunnel的命令,可以让用户穿过防火墙直接操作数据服务.运行这个命令之前需要在本地安装对应服务的客户端.比如刚创建的mongodb服务,如果需要访问的话需要预先把mongo,mongodump等客户端放在path下面.

```
ubuntu@ubuntu:~$ cf tunnel mongo-test
1: none
2: mongo
3: mongodump
4: mongorestore
Which client would you like to start?> 2

Opening tunnel on port 10000... OK
Waiting for local tunnel to become available... OK
MongoDB shell version: 2.2.2
connecting to: localhost:10000/db
> show collections;
system.indexes
system.users
> db.createCollection("hello");
{ "ok" : 1 }
> show collections;
hello
system.indexes
system.users
```
注意由于cf里面做了严格的权限控制,因此cf只能在当前数据库下进行操作,不允许创建新的数据库.

由于在cf中默认是用的free配额,所以只能创建2个服务,同时内存也有限制.具体的配额信息只能去ccdb数据库查看:

| name | non-basic-service | total_services | memory_limit | trial_db_allowed |
|--------| ------------ | ------------- | ------------ |
| free | f | 2 | 1024	|f|
|paid	| t	| 32	|204800	|f|
|runaway| t	| 500	|204800|	f|
|trial | f	| 10	|2048	|t|


这样我们就可以改变当前组织的配额了:

```	
ubuntu@ubuntu:~$ cf set-quota paid cstnet
Setting quota of cstnet to paid... OK
```



##3.创建应用

安装cf的根本目标就是为了跑应用,前面做了一大堆准备工作后终于可以安装应用了.一般的应用都需要使用各种数据服务,所以在这里就跑给一个使用数据库的示例`hello-spring-mysql`.这个例子的下载地址是[cloudfoundry-samples](https://github.com/SpringSource/cloudfoundry-samples.git).

hello-spring-mysql是一个maven项目,功能很简单就是创建一个叫current_state的表.然后插入几条数据,最后把刚才插入到表中的数据打印到网页上.

需要说明的是这个项目现在还不能直接用,需要做一些修改之后才能跑起来:

1.将pom.xml中的spring-release版本从3.1.0M2换成3.2.0-RELEASE:

```
	<properties>
		<java-version>1.6</java-version>
		<org.springframework-version>3.2.0.RELEASE</org.springframework-version>
		<org.slf4j-version>1.6.1</org.slf4j-version>
	</properties>
```
2.在pom.xml中添加一个dependency:

```
		<dependency>
			<groupId>org.cloudfoundry</groupId>
			<artifactId>cloudfoundry-runtime</artifactId>
			<version>0.8.4</version>
		</dependency>
```

3.修改root-context.xml,替换后的内容如下:

```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:context="http://www.springframework.org/schema/context"
	xmlns:jdbc="http://www.springframework.org/schema/jdbc" xmlns:cloud="http://schema.cloudfoundry.org/spring"
	xsi:schemaLocation="http://www.springframework.org/schema/beans
        http://www.springframework.org/schema/beans/spring-beans-3.2.xsd
        http://www.springframework.org/schema/jdbc
        http://www.springframework.org/schema/jdbc/spring-jdbc-3.2.xsd
        http://www.springframework.org/schema/context
        http://www.springframework.org/schema/context/spring-context-3.2.xsd
        http://schema.cloudfoundry.org/spring
        http://schema.cloudfoundry.org/spring/cloudfoundry-spring.xsd">
	<!-- Root Context: defines shared resources visible to all other web components -->

	<cloud:properties id="cloudProperties" />


	<bean id="dataSource" class="org.apache.commons.dbcp.BasicDataSource"
		destroy-method="close">
		<property name="driverClassName" value="com.mysql.jdbc.Driver" />
		<!-- <property name="url" value="jdbc:mysql://127.0.0.1:3306/test" /> -->
		<!-- <property name="username" value="spring" /> -->
		<!-- <property name="password" value="spring" /> -->

		<property name="url"
			value="jdbc:mysql://${cloud.services.msql.connection.host}:${cloud.services.msql.connection.port}/${cloud.services.msql.connection.name}" />
		<property name="username" value="${cloud.services.msql.connection.username}" />
		<property name="password" value="${cloud.services.msql.connection.password}" />

	</bean>

	<jdbc:initialize-database data-source="dataSource">
		<jdbc:script location="classpath:db-schema.sql" />
	</jdbc:initialize-database>

</beans>
```
由于在之前我创建的mysql的实例名称叫做msql,所以在配置文件中所有有关数据库访问的参数中都带有msql:

* cloud.services.msql.connection.host 主机名
* cloud.services.msql.connection.port 端口名
* cloud.services.msql.connection.name 数据库名称 
* cloud.services.msql.connection.username 数据库用户名
* cloud.services.msql.connection.password 数据库密码


在完成上面三步的修改之后就可以编译打包了,运行`mvn package`得到war包.

该命令运行完毕后可以得到一个war包,找到此war包,然后用cf客户端部署应用:

```
ubuntu@ubuntu:~$ cf push hello-spring-mysql.war 
Instances> 1

1: 128M
2: 256M
3: 512M
4: 1G
Memory Limit> 4   

Creating hello-spring-mysql.war... OK

1: hello-spring-mysql.war
2: none
Subdomain> test                  

1: dc.cfos.org
2: none
Domain> 1             

Creating route test.dc.cfos.org... OK
Binding test.dc.cfos.org to hello-spring-mysql.war... OK

Create services for application?> n

Bind other services to application?> y

1: mongo-test
2: redis-test
3: msql
Which service?> 3

Binding msql to hello-spring-mysql.war... OK
Bind another service?> n

Save configuration?> y

Saving to manifest.yml... OK
Uploading hello-spring-mysql.war... OK
Starting hello-spring-mysql.war... OK
-----> Downloaded app package (6.3M)
Installing java.
Downloading JDK...
Copying openjdk-1.7.0_25.tar.gz from the buildpack cache ...
Unpacking JDK to .jdk
Downloading Tomcat: apache-tomcat-7.0.41.tar.gz
Copying apache-tomcat-7.0.41.tar.gz from the buildpack cache ...
Unpacking Tomcat to .tomcat
Copying mysql-connector-java-5.1.12.jar from the buildpack cache ...
Copying postgresql-9.0-801.jdbc4.jar from the buildpack cache ...
Copying auto-reconfiguration-0.6.8.jar from the buildpack cache ...
-----> Uploading droplet (45M)
-----> Uploaded droplet
Checking hello-spring-mysql.war...
Staging in progress...
Staging in progress...
  0/1 instances: 1 starting
  0/1 instances: 1 starting
  0/1 instances: 1 starting
  0/1 instances: 1 starting
  0/1 instances: 1 starting
  0/1 instances: 1 starting
  0/1 instances: 1 starting
  0/1 instances: 1 starting
  1/1 instances: 1 running
OK
```
在上述部署的过程中,命令行会提示用户:选择创建几个实例,使用多大内存,使用什么子域名,是否创建新服务,绑定哪个服务等等;用户只需要根据实际情况选择就行了.在上面的示例中,我选择的是创建1个实例,使用test.dc.cfos.org这个域名,并且绑定名为msql的服务.

在片刻等待后就完成了整个应用的部署,进入浏览器输入test.dc.cfos.org,就会看到输出页面.



##5.调试应用

当程序运行出错时,我们还需要查看日志来调试.下面两个命令对调试很有帮助:

`cf logs --app [APPNAME]`查看应用的日志


`cf files`查看应用下面的目录文件





