---
layout:     post
title:      "事务-02丨@Transactional源码"
date:       2021-01-11 21:43:19
author:     "jiefang"
header-style: text
tags:
    - 事务
    - Spring
---
# Transactional源码

![](https://s3.ax1x.com/2021/01/13/stTLkj.png)

## TransactionInterceptor

位于`spring-tx-5.2.9.RELEASE.jar`中，继承类`TransactionAspectSupport`：其实对其进行了增强（模板方法模式），实现接口`MethodInterceptor`：方法拦截器，执行代理类的目标方法，会触发`invoke()`方法执行。

```java
package org.springframework.transaction.interceptor;
@SuppressWarnings("serial")
public class TransactionInterceptor extends TransactionAspectSupport implements MethodInterceptor, Serializable {
    @Override
	@Nullable
	public Object invoke(MethodInvocation invocation) throws Throwable {
		// Work out the target class: may be {@code null}.
		// The TransactionAttributeSource should be passed the target class
		// as well as the method, which may be from an interface.
		Class<?> targetClass = (invocation.getThis() != null ? AopUtils.getTargetClass(invocation.getThis()) : null);

		// Adapt to TransactionAspectSupport's invokeWithinTransaction...
		return invokeWithinTransaction(invocation.getMethod(), targetClass, invocation::proceed);
	}
}
```

## TransactionAspectSupport#invokeWithinTransaction()

```java
public abstract class TransactionAspectSupport implements BeanFactoryAware, InitializingBean {
    @Nullable
    protected Object invokeWithinTransaction(Method method, @Nullable Class<?> targetClass,
          final InvocationCallback invocation) throws Throwable {

       // If the transaction attribute is null, the method is non-transactional.
       //获取对应事务属性.如果事务属性为空（则目标方法不存在事务）
       TransactionAttributeSource tas = getTransactionAttributeSource();
       final TransactionAttribute txAttr = (tas != null ? tas.getTransactionAttribute(method, targetClass) : null);
       final TransactionManager tm = determineTransactionManager(txAttr);
	   //反应式编程
       if (this.reactiveAdapterRegistry != null && tm instanceof ReactiveTransactionManager) {
          ReactiveTransactionSupport txSupport = this.transactionSupportCache.computeIfAbsent(method, key -> {
             if (KotlinDetector.isKotlinType(method.getDeclaringClass()) && KotlinDelegate.isSuspend(method)) {
                throw new TransactionUsageException(
                      "Unsupported annotated transaction on suspending function detected: " + method +
                      ". Use TransactionalOperator.transactional extensions instead.");
             }
             ReactiveAdapter adapter = this.reactiveAdapterRegistry.getAdapter(method.getReturnType());
             if (adapter == null) {
                throw new IllegalStateException("Cannot apply reactive transaction to non-reactive return type: " +
                      method.getReturnType());
             }
             return new ReactiveTransactionSupport(adapter);
          });
          return txSupport.invokeWithinTransaction(
                method, targetClass, invocation, txAttr, (ReactiveTransactionManager) tm);
       }

       PlatformTransactionManager ptm = asPlatformTransactionManager(tm);
       final String joinpointIdentification = methodIdentification(method, targetClass, txAttr);

       if (txAttr == null || !(ptm instanceof CallbackPreferringPlatformTransactionManager)) {
          //如果有必要创建事务，根据传播行为
          TransactionInfo txInfo = createTransactionIfNecessary(ptm, txAttr, joinpointIdentification);

          Object retVal;
          try {
             // This is an around advice: Invoke the next interceptor in the chain.
             // This will normally result in a target object being invoked.
             retVal = invocation.proceedWithInvocation();
          }
          catch (Throwable ex) {
             //处理完成事务，提交或回滚
             completeTransactionAfterThrowing(txInfo, ex);
             throw ex;
          }
          finally {
             //重置txInfo 
             cleanupTransactionInfo(txInfo);
          }

          if (retVal != null && vavrPresent && VavrDelegate.isVavrTry(retVal)) {
             // Set rollback-only in case of Vavr failure matching our rollback rules...
             TransactionStatus status = txInfo.getTransactionStatus();
             if (status != null && txAttr != null) {
                retVal = VavrDelegate.evaluateTryFailure(retVal, txAttr, status);
             }
          }
		  //提交或者回滚事务
          commitTransactionAfterReturning(txInfo);
          return retVal;
       }
	   //属于编程式事务
       else {
          Object result;
          final ThrowableHolder throwableHolder = new ThrowableHolder();

          // It's a CallbackPreferringPlatformTransactionManager: pass a TransactionCallback in.
          try {
             result = ((CallbackPreferringPlatformTransactionManager) ptm).execute(txAttr, status -> {
                TransactionInfo txInfo = prepareTransactionInfo(ptm, txAttr, joinpointIdentification, status);
                try {
                   Object retVal = invocation.proceedWithInvocation();
                   if (retVal != null && vavrPresent && VavrDelegate.isVavrTry(retVal)) {
                      // Set rollback-only in case of Vavr failure matching our rollback rules...
                      retVal = VavrDelegate.evaluateTryFailure(retVal, txAttr, status);
                   }
                   return retVal;
                }
                catch (Throwable ex) {
                   if (txAttr.rollbackOn(ex)) {
                      // A RuntimeException: will lead to a rollback.
                      if (ex instanceof RuntimeException) {
                         throw (RuntimeException) ex;
                      }
                      else {
                         throw new ThrowableHolderException(ex);
                      }
                   }
                   else {
                      // A normal return value: will lead to a commit.
                      throwableHolder.throwable = ex;
                      return null;
                   }
                }
                finally {
                   cleanupTransactionInfo(txInfo);
                }
             });
          }
          catch (ThrowableHolderException ex) {
             throw ex.getCause();
          }
          catch (TransactionSystemException ex2) {
             if (throwableHolder.throwable != null) {
                logger.error("Application exception overridden by commit exception", throwableHolder.throwable);
                ex2.initApplicationException(throwableHolder.throwable);
             }
             throw ex2;
          }
          catch (Throwable ex2) {
             if (throwableHolder.throwable != null) {
                logger.error("Application exception overridden by commit exception", throwableHolder.throwable);
             }
             throw ex2;
          }

          // Check result state: It might indicate a Throwable to rethrow.
          if (throwableHolder.throwable != null) {
             throw throwableHolder.throwable;
          }
          return result;
       }
    }
}
```

### 开启事务

#### TransactionAspectSupport#createTransactionIfNecessary()

```java
protected TransactionInfo createTransactionIfNecessary(@Nullable PlatformTransactionManager tm,
      @Nullable TransactionAttribute txAttr, final String joinpointIdentification) {

   // If no name specified, apply method identification as transaction name.
   if (txAttr != null && txAttr.getName() == null) {
      txAttr = new DelegatingTransactionAttribute(txAttr) {
         @Override
         public String getName() {
            return joinpointIdentification;
         }
      };
   }

   TransactionStatus status = null;
   if (txAttr != null) {
      if (tm != null) {
         //开启事务
         status = tm.getTransaction(txAttr);
      }
      else {
         if (logger.isDebugEnabled()) {
            logger.debug("Skipping transactional joinpoint [" + joinpointIdentification +
                  "] because no transaction manager has been configured");
         }
      }
   }
   //设置并绑定到当前线程
   return prepareTransactionInfo(tm, txAttr, joinpointIdentification, status);
}
```

#### AbstractPlatformTransactionManager#getTransaction()

```java
public abstract class AbstractPlatformTransactionManager implements PlatformTransactionManager, Serializable {
    	public final TransactionStatus getTransaction(@Nullable TransactionDefinition definition)
			throws TransactionException {

		// Use defaults if no transaction definition given.
		TransactionDefinition def = (definition != null ? definition : TransactionDefinition.withDefaults());
		//DataSourceTransactionManager获取事务对象
		Object transaction = doGetTransaction();
		boolean debugEnabled = logger.isDebugEnabled();
		//是否已经存在事务，根据事务传播行为进行处理
		if (isExistingTransaction(transaction)) {
			// Existing transaction found -> check propagation behavior to find out how to behave.
			return handleExistingTransaction(def, transaction, debugEnabled);
		}

		//超时判断
		if (def.getTimeout() < TransactionDefinition.TIMEOUT_DEFAULT) {
			throw new InvalidTimeoutException("Invalid transaction timeout", def.getTimeout());
		}

		//MANDATORY传播行为需要在事务中
		if (def.getPropagationBehavior() == TransactionDefinition.PROPAGATION_MANDATORY) {
			throw new IllegalTransactionStateException(
					"No existing transaction found for transaction marked with propagation 'mandatory'");
		}//REQUIRED、REQUIRES_NEW和NESTED需要开启新事务
		else if (def.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRED ||
				def.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRES_NEW ||
				def.getPropagationBehavior() == TransactionDefinition.PROPAGATION_NESTED) {
			SuspendedResourcesHolder suspendedResources = suspend(null);
			if (debugEnabled) {
				logger.debug("Creating new transaction with name [" + def.getName() + "]: " + def);
			}
			try {
				return startTransaction(def, transaction, debugEnabled, suspendedResources);
			}
			catch (RuntimeException | Error ex) {
				resume(null, suspendedResources);
				throw ex;
			}
		}
		else {
			// Create "empty" transaction: no actual transaction, but potentially synchronization.
			if (def.getIsolationLevel() != TransactionDefinition.ISOLATION_DEFAULT && logger.isWarnEnabled()) {
				logger.warn("Custom isolation level specified but no actual transaction initiated; " +
						"isolation level will effectively be ignored: " + def);
			}
			boolean newSynchronization = (getTransactionSynchronization() == SYNCHRONIZATION_ALWAYS);
			return prepareTransactionStatus(def, null, true, newSynchronization, debugEnabled, null);
		}
	}
}
```

#### AbstractPlatformTransactionManager#handleExistingTransaction()

```java
private TransactionStatus handleExistingTransaction(
      TransactionDefinition definition, Object transaction, boolean debugEnabled)
      throws TransactionException {
   //NEVER直接抛异常
   if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_NEVER) {
      throw new IllegalTransactionStateException(
            "Existing transaction found for transaction marked with propagation 'never'");
   }
   //NOT_SUPPORTED以非事务方式执行，如果当前存在事务则将当前事务挂起
   if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_NOT_SUPPORTED) {
      if (debugEnabled) {
         logger.debug("Suspending current transaction");
      }
      Object suspendedResources = suspend(transaction);
      boolean newSynchronization = (getTransactionSynchronization() == SYNCHRONIZATION_ALWAYS);
      return prepareTransactionStatus(
            definition, null, false, newSynchronization, debugEnabled, suspendedResources);
   }
   //REQUIRES_NEW开启新事务
   if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRES_NEW) {
      if (debugEnabled) {
         logger.debug("Suspending current transaction, creating new transaction with name [" +
               definition.getName() + "]");
      }
      SuspendedResourcesHolder suspendedResources = suspend(transaction);
      try {
         return startTransaction(definition, transaction, debugEnabled, suspendedResources);
      }
      catch (RuntimeException | Error beginEx) {
         resumeAfterBeginException(transaction, suspendedResources, beginEx);
         throw beginEx;
      }
   }
   //NESTED,嵌套事务，通过savePoint实现
   if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_NESTED) {
      //如果当前TransactionManager不支持嵌套事务,直接抛异常 
      if (!isNestedTransactionAllowed()) {
         throw new NestedTransactionNotSupportedException(
               "Transaction manager does not allow nested transactions by default - " +
               "specify 'nestedTransactionAllowed' property with value 'true'");
      }
      if (debugEnabled) {
         logger.debug("Creating nested transaction with name [" + definition.getName() + "]");
      }
      //判断当前TransactionManager实现是否是采用保存点实现嵌套事务 
      if (useSavepointForNestedTransaction()) {
         // Create savepoint within existing Spring-managed transaction,
         // through the SavepointManager API implemented by TransactionStatus.
         // Usually uses JDBC 3.0 savepoints. Never activates Spring synchronization.
         DefaultTransactionStatus status =
               prepareTransactionStatus(definition, transaction, false, false, debugEnabled, null);
         //创建savePoint 
         status.createAndHoldSavepoint();
         return status;
      }
      else {
         // Nested transaction through nested begin and commit/rollback calls.
         // Usually only for JTA: Spring synchronization might get activated here
         // in case of a pre-existing JTA transaction.
         return startTransaction(definition, transaction, debugEnabled, null);
      }
   }

   // Assumably PROPAGATION_SUPPORTS or PROPAGATION_REQUIRED.
   if (debugEnabled) {
      logger.debug("Participating in existing transaction");
   }
   if (isValidateExistingTransaction()) {
      if (definition.getIsolationLevel() != TransactionDefinition.ISOLATION_DEFAULT) {
         Integer currentIsolationLevel = TransactionSynchronizationManager.getCurrentTransactionIsolationLevel();
         if (currentIsolationLevel == null || currentIsolationLevel != definition.getIsolationLevel()) {
            Constants isoConstants = DefaultTransactionDefinition.constants;
            throw new IllegalTransactionStateException("Participating transaction with definition [" +
                  definition + "] specifies isolation level which is incompatible with existing transaction: " +
                  (currentIsolationLevel != null ?
                        isoConstants.toCode(currentIsolationLevel, DefaultTransactionDefinition.PREFIX_ISOLATION) :
                        "(unknown)"));
         }
      }
      if (!definition.isReadOnly()) {
         if (TransactionSynchronizationManager.isCurrentTransactionReadOnly()) {
            throw new IllegalTransactionStateException("Participating transaction with definition [" +
                  definition + "] is not marked as read-only but existing transaction is");
         }
      }
   }
   boolean newSynchronization = (getTransactionSynchronization() != SYNCHRONIZATION_NEVER);
   return prepareTransactionStatus(definition, transaction, false, newSynchronization, debugEnabled, null);
}
```

#### AbstractPlatformTransactionManager#startTransaction()

```java
private TransactionStatus startTransaction(TransactionDefinition definition, Object transaction,
      boolean debugEnabled, @Nullable SuspendedResourcesHolder suspendedResources) {
   boolean newSynchronization = (getTransactionSynchronization() != SYNCHRONIZATION_NEVER);
   DefaultTransactionStatus status = newTransactionStatus(
         definition, transaction, true, newSynchronization, debugEnabled, suspendedResources);
   //调用事务管理器开启事务
   doBegin(transaction, definition);
   prepareSynchronization(status, definition);
   return status;
}
```

##### DataSourceTransactionManager#doGetTransaction()

根据`dataSource`来获取`ConnectionHolder`，这个`ConnectionHolder`是在`TransactionSynchronizationManager`的`ThreadLocal`中持有的，如果是第一次来获取，肯定得到是null。

```java
@Override
protected Object doGetTransaction() {
   DataSourceTransactionObject txObject = new DataSourceTransactionObject();
   //就是默认支持嵌套事务,而嵌套事务又是采用savepoint实现的
   txObject.setSavepointAllowed(isNestedTransactionAllowed());
   ConnectionHolder conHolder =
         (ConnectionHolder) TransactionSynchronizationManager.getResource(obtainDataSource());
   txObject.setConnectionHolder(conHolder, false);
   return txObject;
}
```

##### DataSourceTransactionManager#doBegin()

```java
@Override
protected void doBegin(Object transaction, TransactionDefinition definition) {
   DataSourceTransactionObject txObject = (DataSourceTransactionObject) transaction;
   Connection con = null;

   try {
      if (!txObject.hasConnectionHolder() ||
            txObject.getConnectionHolder().isSynchronizedWithTransaction()) {
         Connection newCon = obtainDataSource().getConnection();
         if (logger.isDebugEnabled()) {
            logger.debug("Acquired Connection [" + newCon + "] for JDBC transaction");
         }
         txObject.setConnectionHolder(new ConnectionHolder(newCon), true);
      }

      txObject.getConnectionHolder().setSynchronizedWithTransaction(true);
      con = txObject.getConnectionHolder().getConnection();

      Integer previousIsolationLevel = DataSourceUtils.prepareConnectionForTransaction(con, definition);
      txObject.setPreviousIsolationLevel(previousIsolationLevel);
      txObject.setReadOnly(definition.isReadOnly());

      // Switch to manual commit if necessary. This is very expensive in some JDBC drivers,
      // so we don't want to do it unnecessarily (for example if we've explicitly
      // configured the connection pool to set it already).
      if (con.getAutoCommit()) {
         txObject.setMustRestoreAutoCommit(true);
         if (logger.isDebugEnabled()) {
            logger.debug("Switching JDBC Connection [" + con + "] to manual commit");
         }
         //开启事务，设置autoCommit为false
         con.setAutoCommit(false);
      }

      prepareTransactionalConnection(con, definition);
      //设置为true，用于判断是否已经存在事务
      txObject.getConnectionHolder().setTransactionActive(true);

      int timeout = determineTimeout(definition);
      if (timeout != TransactionDefinition.TIMEOUT_DEFAULT) {
         txObject.getConnectionHolder().setTimeoutInSeconds(timeout);
      }

      // Bind the connection holder to the thread.
      if (txObject.isNewConnectionHolder()) {
         //当前的connection放入TransactionSynchronizationManager中持有，下次调用可以判断为已有的事务
         TransactionSynchronizationManager.bindResource(obtainDataSource(), txObject.getConnectionHolder());
      }
   }

   catch (Throwable ex) {
      if (txObject.isNewConnectionHolder()) {
         DataSourceUtils.releaseConnection(con, obtainDataSource());
         txObject.setConnectionHolder(null, false);
      }
      throw new CannotCreateTransactionException("Could not open JDBC Connection for transaction", ex);
   }
}
```

### TransactionAspectSupport#completeTransactionAfterThrowing()

```java
protected void completeTransactionAfterThrowing(@Nullable TransactionInfo txInfo, Throwable ex) {
   if (txInfo != null && txInfo.getTransactionStatus() != null) {
      if (logger.isTraceEnabled()) {
         logger.trace("Completing transaction for [" + txInfo.getJoinpointIdentification() +
               "] after exception: " + ex);
      }
      //如果是要回滚的异常，进行回滚操作，默认是RuntimeException和Error回滚。
      if (txInfo.transactionAttribute != null && txInfo.transactionAttribute.rollbackOn(ex)) {
         try {
            txInfo.getTransactionManager().rollback(txInfo.getTransactionStatus());
         }
         catch (TransactionSystemException ex2) {
            logger.error("Application exception overridden by rollback exception", ex);
            ex2.initApplicationException(ex);
            throw ex2;
         }
         catch (RuntimeException | Error ex2) {
            logger.error("Application exception overridden by rollback exception", ex);
            throw ex2;
         }
      }
      else {
         // We don't roll back on this exception.
         // Will still roll back if TransactionStatus.isRollbackOnly() is true.
         try {
            //不在这里处理的回滚，进行commit操作
            txInfo.getTransactionManager().commit(txInfo.getTransactionStatus());
         }
         catch (TransactionSystemException ex2) {
            logger.error("Application exception overridden by commit exception", ex);
            ex2.initApplicationException(ex);
            throw ex2;
         }
         catch (RuntimeException | Error ex2) {
            logger.error("Application exception overridden by commit exception", ex);
            throw ex2;
         }
      }
   }
}
```

### 提交事务

#### TransactionAspectSupport#commitTransactionAfterReturning()

```java
protected void commitTransactionAfterReturning(@Nullable TransactionInfo txInfo) {
   if (txInfo != null && txInfo.getTransactionStatus() != null) {
      if (logger.isTraceEnabled()) {
         logger.trace("Completing transaction for [" + txInfo.getJoinpointIdentification() + "]");
      }
      //调用事务管理者提交事务
      txInfo.getTransactionManager().commit(txInfo.getTransactionStatus());
   }
}
```

#### AbstractPlatformTransactionManager#commit()

```java
public final void commit(TransactionStatus status) throws TransactionException {
   if (status.isCompleted()) {
      throw new IllegalTransactionStateException(
            "Transaction is already completed - do not call commit or rollback more than once per transaction");
   }

   DefaultTransactionStatus defStatus = (DefaultTransactionStatus) status;
   //判断TransactionStatus，是否回滚
   if (defStatus.isLocalRollbackOnly()) {
      if (defStatus.isDebug()) {
         logger.debug("Transactional code has requested rollback");
      }
      processRollback(defStatus, false);
      return;
   }
   //全局设置是否进行回滚
   if (!shouldCommitOnGlobalRollbackOnly() && defStatus.isGlobalRollbackOnly()) {
      if (defStatus.isDebug()) {
         logger.debug("Global transaction is marked as rollback-only but transactional code requested commit");
      }
      processRollback(defStatus, true);
      return;
   }
   //提交事务
   processCommit(defStatus);
}
```

#### AbstractPlatformTransactionManager#processCommit()

```java
private void processCommit(DefaultTransactionStatus status) throws TransactionException {
   try {
      boolean beforeCompletionInvoked = false;

      try {
         boolean unexpectedRollback = false;
         prepareForCommit(status);
         triggerBeforeCommit(status);
         triggerBeforeCompletion(status);
         beforeCompletionInvoked = true;
		 //如果有savePoint,说明是嵌套事务，释放savePoint
         if (status.hasSavepoint()) {
            if (status.isDebug()) {
               logger.debug("Releasing transaction savepoint");
            }
            unexpectedRollback = status.isGlobalRollbackOnly();
            //释放savePoint 
            status.releaseHeldSavepoint();
         }//是新的事务
         else if (status.isNewTransaction()) {
            if (status.isDebug()) {
               logger.debug("Initiating transaction commit");
            }
            unexpectedRollback = status.isGlobalRollbackOnly();
            //数据库提交事务
            doCommit(status);
         }
         else if (isFailEarlyOnGlobalRollbackOnly()) {
            unexpectedRollback = status.isGlobalRollbackOnly();
         }

         // Throw UnexpectedRollbackException if we have a global rollback-only
         // marker but still didn't get a corresponding exception from commit.
         if (unexpectedRollback) {
            throw new UnexpectedRollbackException(
                  "Transaction silently rolled back because it has been marked as rollback-only");
         }
      }
      catch (UnexpectedRollbackException ex) {
         // can only be caused by doCommit
         triggerAfterCompletion(status, TransactionSynchronization.STATUS_ROLLED_BACK);
         throw ex;
      }
      catch (TransactionException ex) {
         // can only be caused by doCommit
         if (isRollbackOnCommitFailure()) {
            doRollbackOnCommitException(status, ex);
         }
         else {
            triggerAfterCompletion(status, TransactionSynchronization.STATUS_UNKNOWN);
         }
         throw ex;
      }
      catch (RuntimeException | Error ex) {
         if (!beforeCompletionInvoked) {
            triggerBeforeCompletion(status);
         }
         doRollbackOnCommitException(status, ex);
         throw ex;
      }

      // Trigger afterCommit callbacks, with an exception thrown there
      // propagated to callers but the transaction still considered as committed.
      try {
         triggerAfterCommit(status);
      }
      finally {
         triggerAfterCompletion(status, TransactionSynchronization.STATUS_COMMITTED);
      }
   }
   finally {
      cleanupAfterCompletion(status);
   }
}
```

#### DataSourceTransactionManager#doCommit()

```java
@Override
protected void doCommit(DefaultTransactionStatus status) {
   DataSourceTransactionObject txObject = (DataSourceTransactionObject) status.getTransaction();
   Connection con = txObject.getConnectionHolder().getConnection();
   if (status.isDebug()) {
      logger.debug("Committing JDBC transaction on Connection [" + con + "]");
   }
   try {
      con.commit();
   }
   catch (SQLException ex) {
      throw new TransactionSystemException("Could not commit JDBC transaction", ex);
   }
}
```

#### AbstractPlatformTransactionManager#cleanupAfterCompletion()

```java
//清除操作
private void cleanupAfterCompletion(DefaultTransactionStatus status) {
   status.setCompleted();
   if (status.isNewSynchronization()) {
      //清除当前线程的整个事务同步状态：已注册的同步以及各种事务特征。 
      TransactionSynchronizationManager.clear();
   }
   if (status.isNewTransaction()) {
      doCleanupAfterCompletion(status.getTransaction());
   }
   if (status.getSuspendedResources() != null) {
      if (status.isDebug()) {
         logger.debug("Resuming suspended transaction after completion of inner transaction");
      }
      Object transaction = (status.hasTransaction() ? status.getTransaction() : null);
      resume(transaction, (SuspendedResourcesHolder) status.getSuspendedResources());
   }
}
```

#### DataSourceTransactionManager#doCleanupAfterCompletion()

```java
@Override
protected void doCleanupAfterCompletion(Object transaction) {
   DataSourceTransactionObject txObject = (DataSourceTransactionObject) transaction;

   // Remove the connection holder from the thread, if exposed.
   if (txObject.isNewConnectionHolder()) {
      //清除当前线程绑定的connectionHolder
      TransactionSynchronizationManager.unbindResource(obtainDataSource());
   }
   //重置数据库连接
   Connection con = txObject.getConnectionHolder().getConnection();
   try {
      if (txObject.isMustRestoreAutoCommit()) {
         con.setAutoCommit(true);
      }
      DataSourceUtils.resetConnectionAfterTransaction(
            con, txObject.getPreviousIsolationLevel(), txObject.isReadOnly());
   }
   catch (Throwable ex) {
      logger.debug("Could not reset JDBC Connection after transaction", ex);
   }
   if (txObject.isNewConnectionHolder()) {
      if (logger.isDebugEnabled()) {
         logger.debug("Releasing JDBC Connection [" + con + "] after transaction");
      }
      //关闭或归还数据库连接
      DataSourceUtils.releaseConnection(con, this.dataSource);
   }
   //transactionActive设置为false 
   txObject.getConnectionHolder().clear();
}
```

### 回滚事务

#### AbstractPlatformTransactionManager#rollback()

```java
@Override
public final void rollback(TransactionStatus status) throws TransactionException {
   if (status.isCompleted()) {
      throw new IllegalTransactionStateException(
            "Transaction is already completed - do not call commit or rollback more than once per transaction");
   }

   DefaultTransactionStatus defStatus = (DefaultTransactionStatus) status;
   processRollback(defStatus, false);
}
```

#### AbstractPlatformTransactionManager#processRollback()

```java
private void processRollback(DefaultTransactionStatus status, boolean unexpected) {
   try {
      boolean unexpectedRollback = unexpected;

      try {
         triggerBeforeCompletion(status);
		 //如果有savePoint，回滚至savePoint
         if (status.hasSavepoint()) {
            if (status.isDebug()) {
               logger.debug("Rolling back transaction to savepoint");
            }
            status.rollbackToHeldSavepoint();
         }//是新的事务
         else if (status.isNewTransaction()) {
            if (status.isDebug()) {
               logger.debug("Initiating transaction rollback");
            }
            //回滚操作 
            doRollback(status);
         }
         else {
            // Participating in larger transaction
            if (status.hasTransaction()) {
               if (status.isLocalRollbackOnly() || isGlobalRollbackOnParticipationFailure()) {
                  if (status.isDebug()) {
                     logger.debug("Participating transaction failed - marking existing transaction as rollback-only");
                  }
                  doSetRollbackOnly(status);
               }
               else {
                  if (status.isDebug()) {
                     logger.debug("Participating transaction failed - letting transaction originator decide on rollback");
                  }
               }
            }
            else {
               logger.debug("Should roll back transaction but cannot - no transaction available");
            }
            // Unexpected rollback only matters here if we're asked to fail early
            if (!isFailEarlyOnGlobalRollbackOnly()) {
               unexpectedRollback = false;
            }
         }
      }
      catch (RuntimeException | Error ex) {
         triggerAfterCompletion(status, TransactionSynchronization.STATUS_UNKNOWN);
         throw ex;
      }

      triggerAfterCompletion(status, TransactionSynchronization.STATUS_ROLLED_BACK);

      // Raise UnexpectedRollbackException if we had a global rollback-only marker
      if (unexpectedRollback) {
         throw new UnexpectedRollbackException(
               "Transaction rolled back because it has been marked as rollback-only");
      }
   }
   finally {
      //清理操作 
      cleanupAfterCompletion(status);
   }
}
```

##### AbstractTransactionStatus#rollbackToHeldSavepoint()

```java
public void rollbackToHeldSavepoint() throws TransactionException {
   Object savepoint = getSavepoint();
   if (savepoint == null) {
      throw new TransactionUsageException(
            "Cannot roll back to savepoint - no savepoint associated with current transaction");
   }
   //调用JdbcTransactionObjectSupport回滚至savePoint
   getSavepointManager().rollbackToSavepoint(savepoint);
   //调用JdbcTransactionObjectSupport删除此savePoint 
   getSavepointManager().releaseSavepoint(savepoint);
   setSavepoint(null);
}
```

###### JdbcTransactionObjectSupport#rollbackToSavepoint()

```
@Override
public void rollbackToSavepoint(Object savepoint) throws TransactionException {
   ConnectionHolder conHolder = getConnectionHolderForSavepoint();
   try {
      conHolder.getConnection().rollback((Savepoint) savepoint);
      conHolder.resetRollbackOnly();
   }
   catch (Throwable ex) {
      throw new TransactionSystemException("Could not roll back to JDBC savepoint", ex);
   }
}
```

###### JdbcTransactionObjectSupport#releaseSavepoint()

```java
@Override
public void releaseSavepoint(Object savepoint) throws TransactionException {
   ConnectionHolder conHolder = getConnectionHolderForSavepoint();
   try {
      conHolder.getConnection().releaseSavepoint((Savepoint) savepoint);
   }
   catch (Throwable ex) {
      logger.debug("Could not explicitly release JDBC savepoint", ex);
   }
}
```

##### DataSourceTransactionManager#doRollback

```java
@Override
protected void doRollback(DefaultTransactionStatus status) {
   DataSourceTransactionObject txObject = (DataSourceTransactionObject) status.getTransaction();
   Connection con = txObject.getConnectionHolder().getConnection();
   if (status.isDebug()) {
      logger.debug("Rolling back JDBC transaction on Connection [" + con + "]");
   }
   try {
      con.rollback();
   }
   catch (SQLException ex) {
      throw new TransactionSystemException("Could not roll back JDBC transaction", ex);
   }
}
```