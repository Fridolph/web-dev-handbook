# SQL 注入与参数化查询

> SQL 注入的核心不是“输入出现特殊字符”，而是应用把不可信数据拼进 SQL，让输入获得改变查询语义的机会。以下片段用于解释原理，不代表本站已接入数据库。

## 一、SQL 注入怎么发生？

```ts
// 错误示范：不要在真实服务中拼接用户输入。
const sql = `SELECT id, email FROM users WHERE email = '${email}'`
```

应用期望 `email` 只是字符串值，但先拼接 SQL 再交给数据库解析时，输入可能改变引号、查询条件等结构，让数据库执行偏离原意的查询。

影响取决于注入位置、驱动行为和账号权限，可能包括数据泄露、越权、篡改或删除；不要求一定能执行多条 SQL。攻击也可能通过布尔、时间差异等间接判断结果，隐藏错误信息并不等于消除漏洞。

输入可能来自 URL、请求头、表单、导入文件，也可能先存库、后被拼进另一条 SQL。数据曾经保存在数据库中，不代表再次进入 SQL 结构时就可信。

## 二、首选参数化查询

以 PostgreSQL 的 `pg` 驱动为例，假设 `pool` 是应用已有的连接池：

```ts
const result = await pool.query(
  'SELECT id, email FROM users WHERE email = $1',
  [email],
)
```

SQL 结构与参数值分开传给驱动，输入作为值处理，不能通过包含引号等字符改写结构。不要自己替换占位符，也不要先把用户数据插值进 SQL 再称为“参数化”。

参数化不要求必须缓存一个命名的服务端预编译语句，正确调用驱动参数接口才是重点。不同驱动占位符不同，例如部分 MySQL 驱动使用 `?`，应按对应 API 使用。

### 为什么过滤、转义不够？

- 禁止引号、空格和关键词会误伤合法数据，也不覆盖所有上下文。
- 手写转义依赖数据库、编码与语句位置，容易遗漏；HTML 转义不解决 SQL 注入。
- DTO 校验应约束业务类型、长度与格式，但合法字符串也应以参数传递。
- HTTPS、WAF、隐藏报错可以承担其他防护职责，无法修复 SQL 拼接本身。

## 三、动态列名与排序怎么办？

值参数通常不能替代标识符或关键字。`ORDER BY $1` 不会把字符串参数自动当成排序列名，而可能作为常量表达式处理。

优先采用服务端允许名单：

```ts
const sortColumns = new Map([
  ['createdAt', 'created_at'],
  ['email', 'email'],
])
const column = sortColumns.get(sortBy)
if (!column) throw new Error('Unsupported sort field')
if (direction !== 'asc' && direction !== 'desc') {
  throw new Error('Unsupported sort direction')
}
const order = direction === 'desc' ? 'DESC' : 'ASC'

// 插值部分全来自服务端固定映射；用户 ID 仍是值参数。
const result = await pool.query(
  `SELECT id, email FROM users WHERE owner_id = $1 ORDER BY ${column} ${order}`,
  [currentUserId],
)
```

接口层可将无效选项转换为 400 响应。确需动态标识符时，可借助成熟库的标识符引用工具，但仍需业务允许名单；正确引用不代表允许用户任意选表。

### LIKE 的边界

```ts
await pool.query(
  'SELECT id, email FROM users WHERE email LIKE $1',
  [`%${keyword}%`],
)
```

这里构造的是参数值，仍然属于参数化。但输入中的 `%`、`_` 仍有通配符含义；若需求是字面子串匹配，应按数据库规则转义并约定 `ESCAPE`，同时限制长度、分页和成本。参数化不保证业务查询语义或性能符合预期。

## 四、ORM 能自动防所有注入吗？

结构化 API 通常帮助绑定参数，但原始 SQL 和动态表达式仍需审查：

```ts
// TypeORM：userRepository 已由应用注入。
await userRepository.createQueryBuilder('user')
  .where('user.email = :email', { email })
  .getOne()

// Prisma 官方标签模板 API：prisma 已创建。
await prisma.$queryRaw`SELECT id, email FROM users WHERE email = ${email}`
```

Prisma 标签模板由库处理值参数，不等于普通 JavaScript 模板字符串拼接。将拼接字符串传给 `$queryRawUnsafe`，或把不可信数据作为 `Prisma.raw` 片段，可能重新引入风险；标签模板的值插值也不能直接代替表名、列名。

存储过程内部若继续拼接输入执行动态 SQL，也不天然安全。

## 五、参数化之外的防线

| 措施 | 负责什么 |
| --- | --- |
| 参数化、固定模板、动态字段允许名单 | 避免输入改变 SQL 结构 |
| 认证、对象级授权、租户隔离 | 避免“没有注入却访问了别人的记录” |
| 数据库账号最小权限 | 限制漏洞或误操作的影响，业务账号不拥有不必要的管理能力 |
| 长度、分页、超时、并发限制 | 约束成本，避免资源耗尽 |
| 错误与日志治理 | 不暴露 SQL 内部细节，不记录密码、完整令牌等敏感参数 |

事务不是注入防护；密码哈希降低泄露后被猜中的速度，也不能阻止数据被漏洞读取。不同机制保护不同边界。

## 六、后续验证任务

以下需在本地练习库执行，尚非本站完成的数据库集成测试：

1. 用 `o'connor@example.com` 等合法含引号输入验证查询正确性。
2. 将包含 SQL 符号的输入作为参数，确认只参与值匹配。
3. 测试非法排序字段、方向、分页边界和查询成本。
4. 用用户 A 的有效身份请求用户 B 的记录，检查对象级授权。
5. 检查原始查询入口和“先保存、后拼接”的数据流。

## 七、一分钟复述

> SQL 注入是把不可信输入拼进 SQL，导致数据改变查询结构。首选防护是参数化查询，让 SQL 与值分开传递；表名、列名和排序方向等不能用值参数替代的部分，使用服务端允许名单。ORM 可以帮助绑定参数，但原始 SQL 仍可能有风险。此外还要做对象级授权、数据库最小权限和成本限制，因为防注入不等于防所有越权与滥用。

> 来源：[OWASP SQL Injection Prevention](https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html)、[node-postgres Queries](https://node-postgres.com/features/queries)、[TypeORM QueryBuilder](https://typeorm.io/docs/query-builder/select-query-builder/)、[Prisma Raw Queries](https://www.prisma.io/docs/orm/prisma-client/using-raw-sql/raw-queries)
