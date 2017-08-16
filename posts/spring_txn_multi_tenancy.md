## Using Spring JDBC's @Transactional in a multi-tenant environment
One of our customer has a multi-tenant platform to deploy REST web services where by all the data is sharded based on a tenant id. Our webservices has to connect to different databases at runtime based on this tenant id which is available in each web request. 

We use Spring JDBC over other ORMs since we feel it is lightweight and less complicated, also the services most of the time are NOT simple CRUD. In spite of having to write more code especially for writes we were happy. As you might know any web service needs DB transaction management. 

AFAIK there are two options in Spring JDBC for transaction management.

* `TransactionTemplate`
* `@Transactional` annotation

With `TransactionTemplate` we have more control over the code, there is no magic (read AOP) happening behind the scenes. With Java 8 and higher order functions writing callbacks is easier now. So we wrote our own implementation of helper functions like 
```java
    withTransaction(Runnable r);
    transactional(Runnable r);
```
These methods will be invoked like 
```java
    transactional(() -> {
        dbOp1();
        dbOp2();
        .
        .
        .
    });
```

The rough implementation of transactional method is as follows: (P.S let me know if there are errors since it is a rough implementation and not actual code)
```java
    public void transactional(Runnable r) {
        TransactionTemplate template = ...;
        template.execute(status -> {
            r.run();
        });
    }
```
The `status -> ...` lambda translates to the SAM [TransactionCallback](https://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/transaction/support/TransactionCallback.html) used by Spring JDBC.

With `@Transactional` annotation there is no need for Higher Order Functions, writing our own methods etc. But some code weaving happens in the background due to AOP (it is magic). I mean there is code that runs which will not be available in the source file. With Java, bytecode enhancement is inevitable with most of the frameworks. In spite of these magic it simplifies the reading portion. All we have to do is to annotate our method which has to be transactional. E.g.

```java
    @Transactional
    public void someServiceMethod() {
        dbOp1();
        dbOp2();
        .
        .
        .
    }
```

That is it, the code running within this method is automagically encapsulated within a DB transaction. But for `@Transactional` to work we need two beans within the Spring configuration of types `javax.sql.DataSource` and `org.springframework.transaction.PlatformTransactionManager`. Since our Datasource is not static and depends on the tenant id creating a bean at app initialisation was impossible. This lead me to try a foolish solution as show below:

### Attempt 1:

```java
    @Bean
    @Scope("prototype")
    public DataSource dataSource() {
        DataSource ds = MyThreadLocalAwesomeClass.getTemplate().getDataSource();
        return ds;
    }

    @Bean
    @Scope("prototype")    
    public DataSourceTransactionManager dataSourceTransactionManager() {
        return new DataSourceTransactionManager(dataSource);
    }
```

Typical strategies for multi tenant Datasource is to write a request interceptor and intercept each request to capture the tenant ID then identify the appropriate Data source params and set the same in a ThreadLocal variable. Care should be taken to remove the same once the request is served by the server. Here the `MyThreadLocalAwesomeClass` contains the `JdbcTemplate` wrapping the datasource which can be used.

So what happened when i ran the code? App was not able to load because the DataSource was null during the time of app initialisation and Spring complained. This lead to me to searching through forums, stackoverflow etc. Found a spring forum post dating back to 2008 which suggested to implement a proxy, it was one liner suggestion with no code examples. 

Had the :lightbulb: moment. 

### Attempt 2: 

[Proxy Pattern](https://en.wikipedia.org/wiki/Proxy_pattern) is a good fit to this problem. Proxies are already used extensively where lazy realisation of an object is applicable e.g. Hibernate lazy loading uses proxies. Wrote a DataSourceProxy as shown below:

```java
public class DataSourceProxy implements DataSource {

    private DataSource delegate() {
        JdbcTemplate template = MyThreadLocalAwesomeClass.getTemplate();
        if(template == null) {
            throw new IllegalStateException("Datasource cannot be found in MyThreadLocalAwesomeClass");
        } else {
            DataSource ds = template.getDataSource();
            return ds;
        }
    }

    @Override
    public Connection getConnection() throws SQLException {
        return delegate().getConnection();
    }

    @Override
    public Connection getConnection(String username, String password) throws SQLException {
        return delegate().getConnection(username, password);
    }

    @Override
    public <T> T unwrap(Class<T> iface) throws SQLException {
        return delegate().unwrap(iface);
    }

    @Override
    public boolean isWrapperFor(Class<?> iface) throws SQLException {
        return delegate().isWrapperFor(iface);
    }

    @Override
    public PrintWriter getLogWriter() throws SQLException {
        return delegate().getLogWriter();
    }

    @Override
    public void setLogWriter(PrintWriter out) throws SQLException {
        delegate().setLogWriter(out);
    }

    @Override
    public void setLoginTimeout(int seconds) throws SQLException {
        delegate().setLoginTimeout(seconds);
    }

    @Override
    public int getLoginTimeout() throws SQLException {
        return delegate().getLoginTimeout();
    }

    @Override
    public Logger getParentLogger() throws SQLFeatureNotSupportedException {
        return delegate().getParentLogger();
    }
}
```

The trick here is to send the proxy object similar to DataSource when Spring asks for the same during initialisation and while the actual methods are being invoked for e.g. `getConnection()` then lazily find the actual datasource on demand from the threadlocal which by now should have a value and execute the incoming method on the actual datasource instance. The implementation in Spring configuration now changes like below:

```java
    @Bean
    public DataSource dataSource() {
        return new DataSourceProxy();
    }


    @Bean
    public DataSourceTransactionManager dataSourceTransactionManager(DataSource dataSource) {
        return new DataSourceTransactionManager(dataSource);
    }

```
This works well without any issues and we were able to use `@Transactional` on our service layer methods. Though the solution was mentioned there were no code examples I could find in the internet. That is the reason I am writing this post. 

#### Caveats:

The solution provided `DataSourceProxy` is dependent on the underlying `DataSource` implementation/extensions. In our case we found an issue where the DB connection is closed immediately after using it in a transactional context. On debugging through Spring JDBC code the reason was weird. We are using `SingleConnectionDataSource` per tenant and caching the same for performance reasons.  Spring JDBC defines an extension to `javax.sql.DataSource` named `SmartDataSource` as shown below:
```java
public interface SmartDataSource extends DataSource {
	boolean shouldClose(Connection con);
}
```

The DataSource is smart enough to say whether a given connection should be closed or not. Spring JDBC checks whether a given DataSource is a SmartDataSource and if true checks whether the connection can be closed post operations. If the check is false or the DataSource is not smart enough then it closes the connection. Since our `DataSourceProxy` implemented `javax.sql.DataSource` Spring JDBC deemed it as dumb and closed the connection immediately. 

This prompted us to change the implementation of `DataSourceProxy` as follows:

```java

public class DataSourceProxy implements SmartDataSource {

    private SmartDataSource delegate() {
        JdbcTemplate template = MyThreadLocalAwesomeClass.getTemplate()
        if(template == null) {
            throw new IllegalStateException("Datasource cannot be found in MyThreadLocalAwesomeClass!");
        } else {
            DataSource ds = template.getDataSource();
            if(!(ds instanceof SmartDataSource)) {
                throw new IllegalStateException("Datasource should be of type SmartDataSource! Other types not supported by this proxy!");
            }
            return (SmartDataSource) ds;
        }
    }

    .
    .
    .
  
    @Override
    public boolean shouldClose(Connection con) {
        return delegate().shouldClose(con);
    }
}
```
We also check if the underlying DataSource is a SmartDataSource and delegate the `shouldClose()` method call to underlying SmartDataSource in addition to the standard DataSource methods. Since this is dependent on the underlying DataSource type the solution is not universal but can be modified to suit the needs. Since we have a `DataSource` bean now we can even ponder about using ORMs like Hibernate (meh!) which was not possible before due to restrictions. 

Thanks for reading till the end!