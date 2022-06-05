---
title: "再学 MyBatis"
tags: ["MyBatis"]
date: "2022-05-01T00:00:00+08:00"
toc: true
---
## 介绍

使用原生 JDBC 开发存在许多问题，比如：数据库连接池管理、SQL 语句的硬编码、结果的遍历...MyBatis 又有什么好呢？主要精力放在 SQL 上，结果集自动映射成 Java 对象...

**[MyBatis 3.4 参考文档中文版](https://www.bookstack.cn/read/MyBatis-zh-3.4/typeHandlers)**

## 快速入门
1. 编写 MyBatis 配置文件
2. 通过配置文件创建 SqlSessionFactory 会话工厂
3. 通过会话工厂创建 SqlSession[^1] 会话工厂
4. 通过条用 SqlSession 的方法去操作，如果需要提交事务，需要执行 SqlSession 的 `commit()` 方法
5. 释放资源，关闭 SqlSession

![](https://raw.githubusercontent.com/Coder-itCheng/blog-images/master/blog/202206021347174.png "MyBatis工作流程")

**代理对象**<br>只需要我们编写 Mapper 接口和 映射文件，并且遵循如下的规范：

|            属性            |            对应            |
| :------------------------: | :------------------------: |
|         namespace          |         类的全路径         |
|      statement 的 id       |        类中的方法名        |
| statement 的 parameterType | 类中的方法输入参数类型一致 |
|  statement 的 resultType   |   类中方法返回值类型一致   |

---

[^1]: 实现对象是线程不安全的，建议 SqlSession 应用场合在方法体内  
