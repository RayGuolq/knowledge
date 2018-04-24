# JDBC-Statement,PrepareStatement,CallableStatement的联系和区别

JDBC核心API提供了三种向数据库发送SQL语句的类：

- Statement：使用createStatement()创建，用于执行不带参数的简单 SQL 语句。
- PreparedStatement：经过预编译并存储在PreparedStatement对象中的SQL语句，使用prepareStatement()方法创建，**继承 Statement**；用于执行带或不带 IN参数的预编译 SQL 语句。
- CallableStatement：用于执行SQL存储过程，使用prepareCall()方法创建，**继承 PreparedStatement**；用于执行执行对数据库已存储过程的调用。

> 特点 
> Statement 接口提供了执行语句和获取结果的基本方法。  
> PreparedStatement 接口添加了处理 IN 参数的方法。  
> 而CallableStatement 添加了处理 OUT 参数的方法。  


## 联系
- CallableStatement继承自PreparedSatement，PreparedStatement继承自Statement。

## 区别
- Statement每次执行sql语句，数据库都要执行sql语句的编译。当仅执行一次查询并返回结果的情况时，执行效率高于PreparedStatement。
- PreparedStatement是预编译的，使用PreparedStatement有几个好处
  1. 在执行可变参数的一条SQL时，PreparedStatement比Statement的效率高，因为DBMS预编译一条SQL当然会比多次编译一条SQL的效率要高。  
  2. 安全性好，有效防止Sql注入等问题。
  3. 对于多次重复执行的语句，使用PreparedStament效率会更高一点，并且在这种情况下也比较适合使用batch。
  4. 代码的可读性和可维护性。
- CallableStatement接口扩展PreparedStatement，用来调用存储过程,它提供了对输出和输入/输出参数的支持。CallableStatement 接口还具有对 PreparedStatement 接口提供的输入参数的支持。

## 详细介绍下 PreparedStatement(使用的比较多)
- PreparedStatement可以写动态参数化的查询

用PreparedStatement你可以写带参数的sql查询语句，通过使用相同的sql语句和不同的参数值来做查询比创建一个不同的查询语句要好，下面是一个参数化查询： 

```
SELECT interest_rate FROM loan WHERE loan_type=?
```

现在你可以使用任何一种loan类型如：”personal loan”,”home loan” 或者”gold loan”来查询，这个例子叫做参数化查询，因为它可以用不同的参数调用它，这里的”?”就是参数的占位符。 

- PreparedStatement比 Statement 更快

使用 PreparedStatement 最重要的一点好处是它拥有更佳的性能优势，SQL语句会预编译在数据库系统中。执行计划同样会被缓存起来，它允许数据库做参数化查询。使用预处理语句比普通的查询更快，因为它做的工作更少（数据库对SQL语句的分析，编译，优化已经在第一次查询前完成了）。为了减少数据库的负载，生产环境中德JDBC代码你应该总是使用PreparedStatement 。值得注意的一点是：为了获得性能上的优势，应该使用参数化sql查询而不是字符串追加的方式。下面两个SELECT 查询，第一个SELECT查询就没有任何性能优势。

SQL Query 1:字符串追加形式的PreparedStatemen
```
String loanType = getLoanType();
PreparedStatement prestmt = conn.prepareStatement("select banks from loan where loan_type=" + loanType);
```

SQL Query 2：使用参数化查询的PreparedStatement
```
PreparedStatement prestmt = conn.prepareStatement("select banks from loan where loan_type=?");
prestmt.setString(1,loanType);
```

第二个查询就是正确使用PreparedStatement的查询，它比SQL1能获得更好的性能。 
 
- PreparedStatement可以防止SQL注入式攻击

如果你是做Java web应用开发的，那么必须熟悉那声名狼藉的SQL注入式攻击。去年Sony就遭受了SQL注入攻击，被盗用了一些Sony play station（PS机）用户的数据。在SQL注入攻击里，恶意用户通过SQL元数据绑定输入，比如：某个网站的登录验证SQL查询代码为： 

```
strSQL = "SELECT * FROM users WHERE name = '" + userName + "' and pw = '"+ passWord +"';"
```

恶意填入：
```
userName = "1' OR '1'='1";
passWord = "1' OR '1'='1";
```

那么最终SQL语句变成了：
```
strSQL = "SELECT * FROM users WHERE name = '1' OR '1'='1' and pw = '1' OR '1'='1';"
```

 因为WHERE条件恒为真，这就相当于执行：
```
strSQL = "SELECT * FROM users;"
```
 因此可以达到无账号密码亦可登录网站。如果恶意用户要是更坏一点，用户填入：
```
passWord = "1' OR '1'='1";DROP TABLE USERS;
```

SQL语句变成了：
```
strSQL = "SELECT * FROM users WHERE name = 'any_value' and pw = ''; DROP TABLE users"
``` 

这样一来，虽然没有登录，但是数据表都被删除了。 

然而使用PreparedStatement的参数化的查询可以阻止大部分的SQL注入。在使用参数化查询的情况下，数据库系统（eg:MySQL）不会将参数的内容视为SQL指令的一部分来处理，而是在数据库完成SQL指令的编译后，才套用参数运行，因此就算参数中含有破坏性的指令，也不会被数据库所运行。

补充：避免SQL注入的第二种方式：
在组合SQL字符串的时候，先对所传入的参数做字符取代（将单引号字符取代为连续2个单引号字符，因为连续2个单引号字符在SQL数据库中会视为字符中的一个单引号字符，譬如：
```
strSQL = "SELECT * FROM users WHERE name = '" + userName + "';"
```

 传入字符串：
```
userName  = " 1' OR 1=1 "
```

 把userName做字符替换后变成：
```
 userName = " 1'' OR 1=1"
```

最后生成的SQL查询语句为：
```
strSQL = "SELECT * FROM users WHERE name = '1'' OR 1=1'
``` 

比起凌乱的字符串追加似的查询，PreparedStatement查询可读性更好、更安全。 



