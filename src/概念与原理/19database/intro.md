# 数据库概念

> 以 PostgreSQL 为主线，逐步整理数据建模、事务、MVCC、索引和查询优化等基础概念。概念文章将随学习进度补充。

目前可先阅读[PostgreSQL 与 MySQL 对比及选型](/学习自测/19database/q_database_1-postgresql_vs_mysql)，建立整体认识。

## 数据访问与凭证安全

以下原理统一维护在 Web 安全模块，适用于 PostgreSQL、MySQL 等数据库与上层应用：

- [SQL 注入与参数化查询](../10safe/s_safe_6-sql_injection)：查询结构与数据分离，理解 ORM 和动态 SQL 的边界。
- [彩虹表与离线密码破解](../10safe/s_safe_7-rainbow_tables)：理解密码验证数据泄露后的风险。
- [密码哈希、加盐与存储](../10safe/s_safe_8-password_hashing)：学习应用用户密码的存储、验证与升级。

应用用户密码哈希与数据库连接账号的认证是不同层次；例如 PostgreSQL 的 SCRAM-SHA-256 服务于数据库连接认证，不等于业务用户密码存储方案。

相关入口：[数据库面试题](/学习自测/19database/intro) · [数据库实践](/编码与实践/19database/intro)
