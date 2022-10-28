使用Atomikos实现JTA分布式事务

本文全面的介绍了JTA分布式事务模型和接口规范，以及开源的分布式事务解决方案Atomikos。笔者认同"talk is cheap，show me the code"，因此在文章最后，给出一个完整的Atomikos与spring、mybatis整合的完整案例。
1 JTA规范
Java事务API（JTA：Java Transaction API）和它的同胞Java事务服务（JTS：Java Transaction Service），为J2EE平台提供了分布式事务服务（distributed transaction）的能力。 某种程度上，可以认为JTA规范是XA规范的Java版，其把XA规范中规定的DTP模型交互接口抽象成Java接口中的方法，并规定每个方法要实现什么样的功能。
1.1 JTA事务模型
在DTP模型中，规定了模型的五个组成元素：
应用程序(Application)
资源管理器(Resource Manager)
事务管理器(Transaction Manager)
通信资源管理器(Communication Resource Manager)、
通信协议(Communication Protocol)
而在JTA规范中，模型中又多了一个元素Application Server，如下所示：


各个组件的介绍如下：
事务管理器(transaction manager)：
处于图中最为核心的位置，其他的事务参与者都是与事务管理器进行交互。事务了管理器提供事务声明，事务资源管理，同步，事务上下文传播等功能。JTA规范定义了事务管理器与其他事务参与者交互的接口，而JTS规范定义了事务管理器的实现要求，因此我们看到事务管理器底层是基于JTS的。
应用服务器(application server)：
顾名思义，是应用程序运行的容器。JTA规范规定，事务管理器的功能应该由application server提供，如上图中的EJB Server。一些常见的其他web容器，如：jboss、weblogic、websphere等，都可以作为application server，这些web容器都实现了JTA规范。特别需要注意的是，并不是所有的web容器都实现了JTA规范，如tomcat并没有实现JTA规范，因此并不能提供事务管理器的功能。
应用程序(application)：
简单来说，就是我们自己编写的应用，部署到了实现了JTA规范的application server中，之后我们就可以我们JTA规范中定义的UserTransaction类来声明一个分布式事务。通常情况下，application server为了简化开发者的工作量，并不一定要求开发者使用UserTransaction来声明一个事务，开发者可以在需要使用分布式事务的方法上添加一个注解，就像spring的声明式事务一样，来声明一个分布式事务。
特别需要注意的是，JTA规范规定事务管理器的功能由application server提供。但是如果我们的应用不是一个web应用，而是一个本地应用，不需要被部署到application server中，无法使用application server提供的事务管理器功能。又或者我们使用的web容器并没有事务管理器的功能，如tomcat。对于这些情况，我们可以直接使用一些第三方的事务管理器类库，如JOTM和Atomikos。将事务管理器直接整合进应用中，不再依赖于application server。
资源管理器(resource manager)：
理论上任何可以存储数据的软件，都可以认为是资源管理器RM。最典型的RM就是关系型数据库了，如mysql。另外一种比较常见的资源管理器是消息中间件，如ActiveMQ、RabbitMQ等， 这些都是真正的资源管理器。
正常情况下，一个数据库驱动供应商只需要实现JDBC规范即可，一个消息中间件供应商只需要实现JMS规范即可。 引入了分布式事务的概念后，DB、MQ等在DTP模型中的作用都是RM，二者是等价的，需要由TM统一进行协调。
为此，JTA规范定义了一个XAResource接口，其定义RM必须要提供给TM调用的一些方法。之后，不管这个RM是DB，还是MQ，TM并不关心，因为其操作的是XAResource接口。而其他规范(如JDBC、JMS)的实现者，同时也对此接口进行实现。如MysqlXAConnection，就实现了XAResource接口。
通信资源管理器(Communication Resource Manager)：
这个是DTP模型中就已经存在的概念，对于需要跨应用的分布式事务，事务管理器彼此之间需要通信，这是就是通过CRM这个组件来完成的。JTA规范中，规定CRM需要实现JTS规范定义的接口。
下图更加直观的演示了JTA规范中各个模型组件之间是如何交互的：


说明如下：
* application 运行在application server中
* application 需要访问3个资源管理器(RM)上资源：1个MQ资源和2个DB资源。
* 由于这些资源服务器是独立部署的，如果需要同时进行更新数据的话并保证一致性的话，则需要使用到分布式事务，需要有一个事务管理器来统一协调。
* Application Server提供了事务管理器的功能
* 作为资源管理器的DB和MQ的客户端驱动包，都实现了XAResource接口，以供事务管理器调用。


2 JTA规范接口
JTA事务模型规定了一个分布式事务中有哪些组件，而JTA接口规范则规定这些组件之间如何交互。需要注意的是JTA规范定义的这些接口，并不需要应用程序的开发人员去实现，而是由各个厂商去实现，根据在DTP模型中扮演的不同角色，需要实现不同的接口。作为开发人员的我们只需要学会如何使用即可。
JTA规范是Java扩展包，在应用中需要额外引入相应的jar包依赖
<dependency>    <groupId>javax.transaction</groupId>    <artifactId>jta</artifactId>    <version>1.1</version></dependency>
JTA规范1.1中的源码非常少，如下所示：


可以看到，这里除了红色框中包含的接口定义之外，其他全部是异常(XxxException)，这里我们仅讨论JTA规范中定义的接口作用：
* javax.transaction.Status：事务状态，这个接口主要是定义一些表示事务状态的常量，此接口无需实现
* javax.transaction.Synchronization：同步
* javax.transaction.Transaction：事务
* javax.transaction.TransactionManager：事务管理器
* javax.transaction.UserTransaction：用于声明一个分布式事务
* javax.transaction.TransactionSynchronizationRegistry：事务同步注册
* javax.transaction.xa.XAResource：定义RM提供给TM操作的接口
* javax.transaction.xa.Xid：事务id
TM供应商：
实现UserTransaction、TransactionManager、Transaction、TransactionSynchronizationRegistry、Synchronization、Xid接口，通过与XAResource接口交互来实现分布式事务。此外，TM厂商如果要支持跨应用的分布式事务，那么还要实现JTS规范定义的接口。
常见的TM提供者包括我们前面提到的application server，包括:jboss、ejb server、weblogic等，以及一些以第三方类库形式提供事务管理器功能的jotm、Atomikos。
RM供应商：
XAResource接口需要由资源管理器者来实现，XAResource接口中定义了一些方法，这些方法将会被TM进行调用，如：
* start方法：开启事务分支
* end方法：结束事务分支
* prepare方法：准备提交
* commit方法：提交
* rollback方法：回滚
* recover方法：列出所有处于PREPARED状态的事务分支
一些RM提供者，可能也会提供自己的Xid接口的实现。
此外，不同的资源管理器有一些各自的特定接口要实现：
* 如JDBC2.0规范定义支持分布式事务的jdbc driver需要实现：javax.sql.XAConnection、javax.sql.XADataSource接口
* JMS1.0规范规定支持分布式事务的JMS厂商，需要实现javax.jms.XAConnection、javax.jms.XASession接口
作为DTP模型中Application开发者的我们，并不需要去实现任何JTA规范中定义的接口，只需要使用TM提供的UserTransaction实现，来声明、提交、回滚一个分布式事务即可。
以下案例演示了UserTransaction接口的基本使用：构建一个分布式事务，来操作位于2个不同的数据库的数据，假设这两个库中都有一个user表。
UserTransaction userTransaction=...
        try{           
                        //开启分布式事务            
                        userTransaction.begin();             
                        //执行事务分支1           
                         conn1 = db1.getConnection();           
                         ps1= conn1.prepareStatement("INSERT into user(name,age) VALUES ('tianshouzhi',23)");            
                         ps1.executeUpdate();           
                         //执行事务分支2            
                        conn2 = db2.getConnection();           
                         ps2 = conn2.prepareStatement("INSERT into user(name,age) VALUES ('tianshouzhi',23)");           
                         ps2.executeUpdate();            
                        //提交，两阶段提交发生在这个方法内部           
                        userTransaction.commit();        
            }catch (Exception e){            
                        try {                
                                userTransaction.rollback();//回滚            
                                } catch (SystemException ignore) {            } 
            }
需要注意的是，在分布式事务中，当我们需要提交或者回滚一个事务时，不应该再使用Connection接口提供的commit和rollback方法。而是应该使用UserTransaction接口的commit接口和rollback接口替代。
另外，这个案例只是用于说明如何使用UserTransaction类，事实上，在实际开发中，并没有这么复杂。一些开源的分布式事务解决方案，可以与spring声明式事务管理功能，因此我们可以通过一个简单@Transactional注解，即可实现分布式事务的功能。
例如，下面我们将要提到Atomikos，就支持与spring事务整合。
3 Atomikos分布式事务
Atomikos公司旗下有两款著名的分布事务产品：
* TransactionEssentials：开源的免费产品
* ExtremeTransactions：商业版，需要收费
这两个产品的关系如下图所示：


可以看到，在开源版本中支持JTA/XA、JDBC、JMS的分布式事务。
最简单的情况下，你只需要引入如下依赖：
<dependency>    <groupId>com.atomikos</groupId>    <artifactId>transactions-jdbc</artifactId>    <version>4.0.6</version></dependency>
atomikos也支持与spring事务整合。spring事务管理器的顶级抽象是PlatformTransactionManager接口，其提供了个重要的实现类：
* DataSourceTransactionManager：用于实现本地事务
* JTATransactionManager：用于实现分布式事务
显然，在这里，我们需要配置的是JTATransactionManager。
下面的代码片段了，演示了与spring事务整合后的分布式事务的案例代码。假设有两个mybatis映射器接口UserMapper和AccountMapper，操作不同的库。
public class JTAService {   
        @Autowired   
        private UserMapper userMapper;
        //操作db_user库   
        @Autowired   
        private AccountMapper accountMapper;
        //操作db_account库   
        @Transactional   
        public void insert() {      
                User user = new User();      
                user.setName("wangxiaoxiao");      
                userMapper.insert(user);      
                //模拟异常，spring回滚后，db_user库中user表中也不会插入记录      
                Account account = new Account();      
                account.setUserId(user.getId());      
                account.setMoney(123456789);      
                accountMapper.insert(account);   
         }
}
可以发现分布式事务的逻辑，与操作单库事务基本上是完全相同的，底层的复杂逻辑对应用程序开发者来说完全屏蔽。
