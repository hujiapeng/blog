1. Spring通过接口PlatformTransactionManager来抽象事务，不同的数据库操作框架都实现该接口，实现接口中方法。如下
 ```
public interface PlatformTransactionManager extends TransactionManager {
    TransactionStatus getTransaction(@Nullable TransactionDefinition var1) throws TransactionException;
    void commit(TransactionStatus var1) throws TransactionException;
    void rollback(TransactionStatus var1) throws TransactionException;
}
 ```
 方法getTransaction可以获取当前事务，用于提交或回滚等操作。
2. Spring的7中事务传播特性：
 - PROPAGATION_REQUIRED：当前有事务就用当前的，没有就创建新的；
 - PROPAGATION_SUPPORTS：事务可有可无，不是必须的；
 - PROPAGATION_MANDATORY：当前一定要有事务，不然抛出异常；
 - PROPAGATION_REQUIRES_NEW：无论当前是否有事务，都新起事务；
 - PROPAGATION_NOT_SUPPORTED：不支持事务，按非事务运行；
 - PROPAGATION_NEVER：不支持事务，如果有事务就抛出异常；
 - PROPAGATION_NESTED：当前有事务，就在当前事务里起一个子事务，嵌套事务。子事务不影响主事务，主事务会影响子事务。

3. 事务隔离特性，Spring默认-1，即依赖数据库隔离特性
 - READ_UNCOMMITTED：值为1，读未提交；会有脏读、不可重复读和幻读
 - READ_COMMITTED：值为2，读已提交；会有不可重复读和幻读
 - REPEATABLE_READ：值为3，可重复读；会有幻读
 - SERIALIZABLE：值为4，串读；