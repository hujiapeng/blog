一、小心Spring的事务可能没有生效
1. 使用@Transactional注解开启声明式事务时，要注意两个原则：
 - 除非特殊配置(比如使用AspectJ静态织入实现AOP)，否则只有定义在public方法上的@Transactional才能生效。Spring默认通过动态代理的方式实现AOP，对目标方法增强。private方法无法代理到，所以Spring无法动态增强事务处理逻辑
 - 必须通过代理过来的类从外部调用目标方法才能生效

二、事务即便生效也不一定能回滚
1. 通过AOP实现事务处理，可以理解为用try···catch将标记了@Transactional注解的方法包裹了一下，当方法出现了异常并且满足一定条件的时候，在catch里设置事务回滚，没有异常的话则直接提交事务
2. 上述一定条件包括两点：
 - 只有异常传播出了标记了@Transactional注解的方法，事务才能回滚
 - 默认情况下，出现RuntimeException(非受检异常)或Error的时候，Spring才会回滚事务
 - 事务相关源码如下
    - 执行事务方法入口TransactionAspectSupport#invokeWithinTransaction
    - 触发回滚事务的代码在该方法内如下
```
			catch (Throwable ex) {
				// target invocation exception
				completeTransactionAfterThrowing(txInfo, ex);
				throw ex;
			}
```
     - 进入方法completeTransactionAfterThrowing(txInfo, ex)，可看到判断是否抛出异常的代码如下
```
if (txInfo.transactionAttribute != null && txInfo.transactionAttribute.rollbackOn(ex)) {
				try {
					txInfo.getTransactionManager().rollback(txInfo.getTransactionStatus());
				}
```
     - 进入方法txInfo.transactionAttribute.rollbackOn(ex)，即DefaultTransactionAttribute#rollbackOn，可看到根据异常条件处理事务的逻辑
```
	public boolean rollbackOn(Throwable ex) {
		return (ex instanceof RuntimeException || ex instanceof Error);
	}
```

三、请确认事务传播配置是否符合自己的业务逻辑
1. 下面就罗列下Spring事务传播特性吧
 - Propagation.REQUIRED：Support a current transaction, create a new one if none exists.如果当前有事务就加入当前事务，否则就创建一个新事务；
 - Propagation.SUPPORTS：Support a current transaction, execute non-transactionally if none exists.如果当前有事务就加入当前事务，否则就非事务运行；
 - Propagation.MANDATORY：Support a current transaction, throw an exception if none exists.必须要有事务，否则抛出异常；
 - Propagation.REQUIRES_NEW：Create a new transaction, and suspend the current transaction if one exists.使用新事务，如果当前有事务就挂起当前事务。新事务和当前事务之间互不影响；
 - Propagation.NOT_SUPPORTED：Execute non-transactionally, suspend the current transaction if one exists.以非事务方式运行，如果当前有事务就挂起当前事务；
 - Propagation.NEVER：Execute non-transactionally, throw an exception if a transaction exists.以非事务方式运行，如果当前有事务就抛出异常；
 - Propagation.NESTED：Execute within a nested transaction if a current transaction exists,behave like REQUIRED otherwise.如果当前有事务，以内嵌事务方式运行，否则就像REQUIRED，新建事务。内嵌事务依赖外部事务，如果外部事务回滚，则内嵌事务回滚。外部事务不受内嵌事务影响