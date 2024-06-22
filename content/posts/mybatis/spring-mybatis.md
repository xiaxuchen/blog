---
title: "Spring Mybatis"
date: 2024-03-25T13:16:31+08:00
draft: true
---
# Spring是如何以同一个Mapper使用不同SqlSession的？
Spring通过MapperScan注解，注册PostProcessor，然后在PostProcessor中替换掉mapper的接口的BeanClass为MapperFactoryBean,并将Mapper原来的Class以参数传递进去，这样构建的Mapper都由MapperFactoryBean进行创建，而Mapper的SqlSession实现类则是SqlSessionTemplate,其是一个代理类，内部代理的SqlSession通过JDK动态代理拦截方法的调用，每次方法调用会获取SqlSession，`SqlSession sqlSession = SqlSessionUtils.getSqlSession(SqlSessionTemplate.this.sqlSessionFactory, SqlSessionTemplate.this.executorType, SqlSessionTemplate.this.exceptionTranslator);
`,而内部则是通过Holder进行存储和获取，如下:
```
    public static SqlSession getSqlSession(SqlSessionFactory sessionFactory, ExecutorType executorType, PersistenceExceptionTranslator exceptionTranslator) {
        Assert.notNull(sessionFactory, "No SqlSessionFactory specified");
        Assert.notNull(executorType, "No ExecutorType specified");
        SqlSessionHolder holder = (SqlSessionHolder)TransactionSynchronizationManager.getResource(sessionFactory);
        SqlSession session = sessionHolder(executorType, holder);
        if (session != null) {
            return session;
        } else {
            LOGGER.debug(() -> {
                return "Creating a new SqlSession";
            });
            session = sessionFactory.openSession(executorType);
            registerSessionHolder(sessionFactory, executorType, exceptionTranslator, session);
            return session;
        }
    }
```
而sessionHolder实现则是如下:
```
private static SqlSession sessionHolder(ExecutorType executorType, SqlSessionHolder holder) {
        SqlSession session = null;
        if (holder != null && holder.isSynchronizedWithTransaction()) {
            if (holder.getExecutorType() != executorType) {
                throw new TransientDataAccessResourceException("Cannot change the ExecutorType when there is an existing transaction");
            }

            holder.requested();
            LOGGER.debug(() -> {
                return "Fetched SqlSession [" + holder.getSqlSession() + "] from current transaction";
            });
            session = holder.getSqlSession();
        }

        return session;
    }
```
整体来说并不复杂。当开启新的事务以及关闭事务的时候就需要切换SqlSession了。这也能回答用 SpringFramework / SpringBoot 整合 MyBatis 时，当 Service 的方法没有标注 @Transactional 注解，或者没有被事务增强器的通知切入时，两次查询同一条数据时，会发送两次 SQL 到数据库，这样看上去像是一级缓存失效了！
