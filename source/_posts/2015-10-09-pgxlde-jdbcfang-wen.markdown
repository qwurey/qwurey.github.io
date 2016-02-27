---
layout: post
title: "PGXL的JDBC访问"
date: 2015-10-09 21:34:17 +0800
comments: true
categories: pg
tags: [pgxl, postgres-xl]
styles: [data-table]
---

> “任何傻瓜都能写出计算机可以理解的代码。好的程序员能写出人能读懂的代码” —— Martin Fowler


#### **PGXL集群的配置**
<br>
见前面的博文[《Postgres-XL集群的搭建》](<http://blog.csdn.net/yeruby/article/details/48996027>)。


#### **JDBC连接PGXL集群**
<br>
PGXL底层为PostgreSQL，这意味着它支持所有支持PostgresSQL类型的驱动，包括： JDBC, ODBC, OLE DB, Python, Ruby, perl DBI, Tcl, and Erlang.

这里在cnode5上连接cnode5的Coordinator进行操作：

建立Java Project，导入相应的PostgreSQL的jdbc驱动；

编写代码，注意以下变量含义：

```
static String url = "jdbc:postgresql://172.16.15.135:20004/test";
static String usr = "postgres";
static String psd = "333333";
```

Coordinator的port为20004，数据库test，user为postgres；

完整code：

{% highlight java %}

package indi.qw.pgxljdbc.demo;

import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.ResultSet;
import java.sql.Statement;

public class PGXLJDBCDemo {
	static String url = "jdbc:postgresql://172.16.15.135:20004/test";
    static String usr = "postgres";
    static String psd = "333333";

	public static void main(String[] args) {
		// TODO Auto-generated method stub
		Connection conn = null;
        try {
            Class.forName("org.postgresql.Driver");
            conn = DriverManager.getConnection(url, usr, psd);
            Statement st = conn.createStatement();
            ResultSet rs = st.executeQuery("SELECT * FROM TEST");
            while (rs.next()) {
               System.out.print(rs.getString(1));
               System.out.print("\n");
            }
            rs.close();
            st.close();
            conn.close();
        } catch (Exception e) {
            e.printStackTrace();
        }
	}

}

{% endhighlight %}


