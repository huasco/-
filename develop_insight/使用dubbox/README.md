# dubbox+zookeeper的使用开发

### 说在前面

dubbo很早就停止服务了，现在维持dubbox服务的是当当在做，里面用的很多第三方依赖有点老，导致很多地方会有兼容性问题，要不是公司开发需要使用dubbox，我觉得我更加偏向使用spring cloud，毕竟这个长期维护，又有spring这么庞大的生态圈，更加好用。关于dubbox的架构思想可以参照原来的[dubbo](http://dubbo.io/User+Guide-zh.htm)

### 准备工作

由于这个使用的是zookeeper+dubbox的一个架构，所以我们先下载zookeeper。

直接google搜索zookeeper，[下载](http://mirror.bit.edu.cn/apache/zookeeper/zookeeper-3.4.9/)3.4.9版本的，3.5.x版本的都是alpha版本，有关区别可以自己搜搜看吧。

下载完之后解压到自己的一个熟悉的目录，然后终端进入zookeeper/bin 目录下，然后执行 ./zkServer.sh start 开启zookeeper就好了。

### dubbox的搭建

这一步工作是重中之重了。我仿照官网的demo例子写了一个demo，可以git clone我的，也可以用dubbox的那个例子。现在我来从零开始写一个。

0. 开发环境：idea + jdk8 + maven

1. idea创建maven项目，不选择模板，修改自己项目的信息，即可

2. 在这个项目下创建三个module，这三个module都是maven项目，说白了，就是一个大项目下面包含三个项目。这三个项目名字和功能分别是：zookeeper-dubbox-api（提供接口服务），zookeeper-dubbox-provider（提供接口实现，提供者），zookeeper-dubbox-consumer（服务消费者），真实的情况下，针对一个api会有多个provider实现，这样如果一个服务挂了，还可以有其他的可以顶上去。不会导致整个项目崩了。

3. 先实现api。具体代码参照工程，我说一下思路。样例里面实现的是一个获取用户信息的服务，所以需要一个用户实体类，以及提供服务的借口。因为涉及到网络的传输，所以我们需要使用一个序列化，也就是可以看到的SerializationOptimizerImpl，这个类，关于序列化可以看看[这个](https://dangdangdotcom.github.io/dubbox/serialization.html)，这样api这个就写好了。

4. 实现provider。这个是服务提供者，是整个项目中最为重要的几个地方。主要实现的也是api中提供的接口。有几个地方需要注意的是，api和provider之间的连通是通过pom.xml中的依赖实现的，如下：

   ```Xml
   <!--公用的服务接口 -->
           <dependency>
               <groupId>cn.limbo</groupId>
               <artifactId>zookeeper-dubbox-api</artifactId>
               <version>1.0-SNAPSHOT</version>
           </dependency>
   ```

   然后按照工程代码实现就可以了。然后需要在resources下面建立META-INF文件夹，里面放置provider的spring文件，用于创建容器的。具体可以看配置，都有注释的。

5. 实现consumer。同样的，需要调取api的接口的话，也要在pom.xml指定依赖：

   ```Xml
    <!--公用的服务接口 -->
           <dependency>
               <groupId>cn.limbo</groupId>
               <artifactId>zookeeper-dubbox-api</artifactId>
               <version>1.0-SNAPSHOT</version>
           </dependency>
   ```

   同样需要建立META-INF里面的spring配置文件，写一个测试程序就好咯。

6. 这样整个项目就搭建好了，也实现了dubbox+zookeeper的搭建，现在可以跑一下了，先启动provider再启动consumer，看到控制台输出就好了

7. 想要可视化的查看服务和消费的情况该怎么办？很简单，去dubbo的github上拉取整个项目，然后我们需要的是dubbo-admin这个模块，这是一个java web的项目，先用maven打包，然后扔到tomcat下面就好了，这里讲的不够清楚，放上一个我搭建时候参考的[文章](http://blog.csdn.net/evankaka/article/details/47858707)当然也会有问题，因为dubbo当初搭建的是jdk7的版本，相信很多人用的都是jdk8，这中间会有问题，主要是pom.xml的问题，如果是jdk8的再看看[这个](http://www.cnblogs.com/ziyouchutuwenwu/p/6292070.html)，这样就可以都看到了

### 总结

dubbo这个东西在分布式的环境下有大用处，但是如果项目不大或者是单机情况下完全够用了，可以不用上dubbo的，不要盲目使用，可能会造成负担。说几个坑的地方吧。

1. 网上的教程很多都不是很靠谱，大多数的教程都是写给自己看的只有自己看的懂，我的建议是先git 官方的demo自己先跑一跑看一看，有一个大概的了解，然后看教程也好，看源码也好，都会轻松很多。

2. maven真的不要以为只是一个简单的添加依赖的东西，有时候项目的连接互通也可以使用maven。

3. 使用rest的时候，如果用的是原生的javax里面的包的话，一定要注意注解的使用，我当初就是少了一个注解，死活跑不起来，错误原因如下：

   ```Java
   	@GET
   	@Path("{id : \\d+}")
   	//就是好了一个@PathParam("id")，就出问题了
   	public User getUser(@PathParam("id") Long id) {
   		if (RpcContext.getContext().getRequest(HttpServletRequest.class) != null) {
   			System.out.println("Client IP address from RpcContext: " + 		    RpcContext.getContext().getRequest(HttpServletRequest.class).getRemoteAddr());
   		}
   		if (RpcContext.getContext().getResponse(HttpServletResponse.class) != null) {
   			System.out.println("Response object from RpcContext: " + RpcContext.getContext().getResponse(HttpServletResponse.class));
   		}
   		return userService.getUser(id);
   	}
   ```


   看到一个关于javax rest讲解挺不错的[文章](http://dangdangdotcom.github.io/dubbox/rest.html)

4. 这只是简单的单机应用，分布式要复杂的多

   ​