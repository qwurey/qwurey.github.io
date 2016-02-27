---
layout: post
title: "PostgreSQL的JDBC访问"
date: 2015-10-09 20:45:33 +0800
comments: true
categories: pg
tags: [pg]
styles: [data-table]
---



> “过去的代码都是未经测试的代码” —— 无名


#### **JDBC的版本选择**
<br>
官网有给出具体的jdk与postgresql driver的版本对应关系：https://jdbc.postgresql.org/download.html

这里使用jdk1.7，对应：JDBC41 Postgresql Driver, Version 9.4-1203

#### **连接数据库**
<br>
导入jar包


在shell终端进入psql：

```
psql -U urey
```

新建数据库testdb并切换到该数据库下：
```
CREATE DATABASE testdb;
\c testdb
```
新建表test并插入一条数据：

```
CREATE TABLE test(id int);
INSERT INTO test VALUES(1);
```

查看刚插入的数据：

```
SELECT * FROM test;
```

编辑java代码连接：

```java

package indi.qw.pgjdbc.demo;

import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.ResultSet;
import java.sql.Statement;

public class PGJDBCDemo {
	static String url = "jdbc:postgresql://127.0.0.1:5432/testdb";
    static String usr = "urey";
    static String psd = "urey";
    
    public static void main(String args[]) {
        Connection conn = null;
        try {
            Class.forName("org.postgresql.Driver");
            conn = DriverManager.getConnection(url, usr, psd);
            Statement st = conn.createStatement();
            ResultSet rs = st.executeQuery("SELECT * FROM TEST");
            while (rs.next()) {
               System.out.print(rs.getString(1));
            }
            rs.close();
            st.close();
            conn.close();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}

```

OK,测试成功。
