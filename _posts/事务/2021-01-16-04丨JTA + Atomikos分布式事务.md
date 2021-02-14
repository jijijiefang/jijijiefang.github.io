---
layout:     post
title:      "事务-04丨JTA + Atomikos分布式事务"
date:       2021-01-16 21:53:22
author:     "jiefang"
header-style: text
tags:
    - 事务
---

# JTA + Atomikos分布式事务

## Atomikos使用

### 引入依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-jta-atomikos</artifactId>
</dependency>
```

### 修改配置文件

```yaml
activity:
  datasource:
type: com.alibaba.druid.pool.xa.DruidXADataSource
jta:
  log-dir: classpath:tx-logs
  transaction-manager-id: txManager
```

### 重构Druid数据源配置

```java
DruidXADataSource datasource = new DruidXADataSource();
//设置datasource属性
...
AtomikosDataSourceBean atomikosDataSource = new AtomikosDataSourceBean();
atomikosDataSource.setXaDataSource(datasource);
```

### 配置JAT事务管理器

```java
@Bean(name = "xatx")
@Primary
public JtaTransactionManager activityTransactionManager() {
    UserTransactionManager userTransactionManager = new UserTransactionManager();
    UserTransaction userTransaction = new UserTransactionImp();
    return new JtaTransactionManager(userTransaction, userTransactionManager);
}
```

### @Transactional设置事务管理器

```java
@Transactional(transactionManager = "xatx", rollbackFor = Exception.class)
```

## 原理

![](https://s3.ax1x.com/2021/01/17/srH8vF.png)

### Atomikos数据源

创建数据库连接代理对象**AtomikosConnectionProxy**。

`AbstractDataSourceBean#getConnection()->ConnectionPool#borrowConnection()->ConnectionPool#findExistingOpenConnectionForCallingThread()->ConnectionPool#recycleConnectionIfPossible()->AbstractXPooledConnection#createConnectionProxy()->AtomikosXAPooledConnection#doCreateConnectionProxy()->AtomikosConnectionProxy#newInstance`

### MySQL XA事务基本语法

`XA {START|BEGIN} xid [JOIN|RESUME]` 启动xid事务 (xid 必须是一个唯一值; 不支持[JOIN|RESUME]子句)
`XA END xid [SUSPEND [FOR MIGRATE]]` 结束xid事务 ( 不支持[SUSPEND [FOR MIGRATE]] 子句)
`XA PREPARE xid` 准备、预提交xid事务
`XA COMMIT xid [ONE PHASE]` 提交xid事务
`XA ROLLBACK xid` 回滚xid事务
`XA RECOVER` 查看处于PREPARE 阶段的所有事务

### 开启事务

`JtaTransactionManager#doBegin()-> JtaTransactionManager#doJtaBegin()->UserTransactionImp#begin()-> UserTransactionManager#begin()->TransactionManagerImp#begin()`

#### TransactionManagerImp#begin()

```java
public void begin () throws NotSupportedException, SystemException{
    begin ( getTransactionTimeout() );
}
public void begin ( int timeout ) throws NotSupportedException,SystemException{
    CompositeTransaction ct = null;
    ResumePreviousTransactionSubTxAwareParticipant resumeParticipant = null;

    ct = compositeTransactionManager.getCompositeTransaction();
    if ( ct != null && ct.getProperty (  JTA_PROPERTY_NAME ) == null ) {
        LOGGER.logWarning ( "JTA: temporarily suspending incompatible transaction: " + ct.getTid() +
                           " (will be resumed after JTA transaction ends)" );
        ct = compositeTransactionManager.suspend();
        resumeParticipant = new ResumePreviousTransactionSubTxAwareParticipant ( ct );
    }

    try {
        ct = compositeTransactionManager.createCompositeTransaction ( ( ( long ) timeout ) * 1000 );
        if ( resumeParticipant != null ) ct.addSubTxAwareParticipant ( resumeParticipant );
        if ( ct.isRoot () && getDefaultSerial () )
            ct.setSerial ();
        ct.setProperty ( JTA_PROPERTY_NAME , "true" );
    } catch ( SysException se ) {
        String msg = "Error in begin()";
        LOGGER.logError( msg , se );
        throw new ExtendedSystemException ( msg , se );
    }
    recreateCompositeTransactionAsJtaTransaction(ct);
}
```

#### CompositeTransactionManagerImp#getCompositeTransaction()

```java
public CompositeTransaction getCompositeTransaction () throws SysException{
    CompositeTransaction ct = null;
    ct = getCurrentTx ();
    if ( ct != null ) {
       if(LOGGER.isTraceEnabled()){
           LOGGER.logTrace("getCompositeTransaction()  returning instance with id "
                    + ct.getTid ());
       }
    } else{
       if(LOGGER.isTraceEnabled()){
          LOGGER.logTrace("getCompositeTransaction() returning NULL!");
       }
    }
    return ct;
}
```

##### CompositeTransactionManagerImp#getCurrentTx()

```java
private CompositeTransaction getCurrentTx (){
    Thread thread = Thread.currentThread ();
    synchronized ( threadtotxmap_ ) {
        //获取当前线程对应的分布式事务集合
        Stack<CompositeTransaction> txs = threadtotxmap_.get ( thread );
        if ( txs == null )
            return null;
        else
            return txs.peek ();
    }
}
```

#### CompositeTransactionManagerImp#createCompositeTransaction()

```java
public CompositeTransaction createCompositeTransaction ( long timeout ) throws SysException{
    CompositeTransaction ct = null , ret = null;
    
    ct = getCurrentTx ();
    if ( ct == null ) {
        //创建分布式事务
        ret = getTransactionService().createCompositeTransaction ( timeout );
        if(LOGGER.isDebugEnabled()){
           LOGGER.logDebug("createCompositeTransaction ( " + timeout + " ): "
                + "created new ROOT transaction with id " + ret.getTid ());
        }
    } else {
        if(LOGGER.isDebugEnabled()) LOGGER.logDebug("createCompositeTransaction ( " + timeout + " )");
        ret = ct.createSubTransaction ();
    }
    Thread thread = Thread.currentThread ();
    setThreadMappings ( ret, thread );

    return ret;
}
```

##### CompositeTransactionManagerImp#setThreadMappings()

```java
private void setThreadMappings ( CompositeTransaction ct , Thread thread )
        throws IllegalStateException, SysException{
    //case 21806: callbacks to ct to be made outside synchronized block
   ct.addSubTxAwareParticipant ( this ); //step 1

    synchronized ( threadtotxmap_ ) {
       //between step 1 and here, intermediate timeout/rollback of the ct
       //may have happened; make sure to check or we add a thread mapping
       //that will never be removed!
       if ( TxState.ACTIVE.equals ( ct.getState() )) {
          Stack<CompositeTransaction> txs = threadtotxmap_.get ( thread );
          if ( txs == null )
             txs = new Stack<CompositeTransaction>();
          txs.push ( ct );
          threadtotxmap_.put ( thread, txs );
          txtothreadmap_.put ( ct, thread );
       }
    }
}
```

### XA START指令

分支事务发送**XA START**命令，调用prepareStatement，分支事务加入全局事务。

`AtomikosConnectionProxy#invoke()->AtomikosConnectionProxy#enlist()->SessionHandleState#notifyBeforeUse()->`
`TransactionContext#checkEnlistBeforeUse()->NotInBranchStateHandler#checkEnlistBeforeUse()->new BranchEnlistedStateHandler()->XAResourceTransaction#resume()->XAResource#start()->XA START 指令`



拦截`"createStatement", "prepareStatement", "prepareCall"`，分支事务加入分布式事务-全局事务。
`AtomikosConnectionProxy#invoke()->AtomikosConnectionProxy#enlist()->CompositeTransaction#registerSynchronization()->TransactionStateHandler#registerSynchronization()->CoordinatorImp#registerSynchronization()->CoordinatorImp#rememberSychronizationForAfterCompletion()`

### XA END指令

分支事务发送**XA END**指令。
`AbstractPlatformTransactionManager#processCommit->AbstractPlatformTransactionManager#triggerBeforeCompletion(status)`
`JtaTransactionManager#doCommit()->``TransactionSynchronizationUtils#triggerBeforeCompletion()->SqlSessionUtils.SqlSessionSynchronization#beforeCompletion()`
拦截close()方法
`AtomikosConnectionProxy#invoke()->AtomikosConnectionProxy#close()->SessionHandleState#notifySessionClosed()->`
`TransactionContext#sessionClosed()->BranchEnlistedStateHandler#sessionClosed()->new BranchEndedStateHandler()->XAResourceTransaction#suspend()->XAResource#end()->XA END 指令`

### 准备

`AbstractPlatformTransactionManager#processCommit->JtaTransactionManager#doCommit()->UserTransactionImp#commit()`
`->UserTransactionManager#commit()->TransactionManagerImp#commit()->TransactionImp#commit()->CompositeTransactionImp#commit()`
`->CoordinatorImp#terminate()->CoordinatorImp#prepare()->XAResourceTransaction#prepare()->XAResource#prepare()`

#### CoordinatorImp#terminate()

```
protected void terminate ( boolean commit ) throws HeurRollbackException,
        HeurMixedException, SysException, java.lang.SecurityException,
        HeurCommitException, HeurHazardException, RollbackException,
        IllegalStateException{
   synchronized ( fsm_ ) {
      if ( commit ) {
         if ( participants_.size () <= 1 ) {
            commit ( true );
         } else {
            int prepareResult = prepare ();
            // make sure to only do commit if NOT read only
            if ( prepareResult != Participant.READ_ONLY )
               commit ( false );
         }
      } else {
         rollback ();
      }
   }
}
```

#### CoordinatorImp#prepare()

```java
public int prepare () throws RollbackException,
        java.lang.IllegalStateException, HeurHazardException,
        HeurMixedException, SysException{
    // FIRST, TAKE CARE OF DUPLICATE PREPARES
    // Recursive prepare-calls should be avoided for not deadlocking rollback/commit methods
    // If a recursive prepare re-enters, then it will see a voting state -> reject.
    // Note that this may also avoid some legal prepares, but only rarely
    if ( getState ().equals ( TxState.PREPARING ) )
        throw new RollbackException ( "Recursion detected" );

    int ret = Participant.READ_ONLY + 1;
    synchronized ( fsm_ ) {
       ret = stateHandler_.prepare ();
       if ( ret == Participant.READ_ONLY ) {
           if ( LOGGER.isTraceEnabled() ) LOGGER.logTrace (  "prepare() of Coordinator  " + getCoordinatorId ()
                + " returning READONLY" );
       } else {
           if ( LOGGER.isTraceEnabled() ) LOGGER.logTrace ( "prepare() of Coordinator  " + getCoordinatorId ()
                + " returning YES vote");
       }
   }
    return ret;
}
```

#### XAResourceTransaction#prepare()

```
@Override
public synchronized int prepare() throws RollbackException,
      HeurHazardException, HeurMixedException, SysException {
   int ret = 0;
   terminateInResource();

   if (TxState.ACTIVE == this.state) {
      // tolerate non-delisting apps/servers
      suspend();
   }

   // duplicate prepares can happen for siblings in serial subtxs!!!
   // in that case, the second prepare just returns READONLY
   if (this.state == TxState.IN_DOUBT)
      return Participant.READ_ONLY;
   else if (!(this.state == TxState.LOCALLY_DONE))
      throw new SysException("Wrong state for prepare: " + this.state);
   try {
      // refresh xaresource for MQSeries: seems to close XAResource after
      // suspend???
      testOrRefreshXAResourceFor2PC();
      if (LOGGER.isTraceEnabled()) {
         LOGGER.logTrace("About to call prepare on XAResource instance: "
               + this.xaresource);
      }
      ret = this.xaresource.prepare(this.xid);
   } catch (XAException xaerr) {
      ...
   }
   setState(TxState.IN_DOUBT);
   ...
}
```

### 事务提交

`CoordinatorImp#commit()->XAResourceTransaction#commit()->Xaresource.commit()`

### 事务回滚

`CoordinatorImp#rollback()->XAResourceTransaction#rollback()->Xaresource.rollback()`