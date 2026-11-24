# TypeORM、Prisma 与数据库迁移复习

> ORM 帮助组织数据访问，migration 帮助管理数据库演进；两者都不能替代 SQL、索引、事务和数据建模。本篇在[ORM 入门](./s_nestjs_8-orm)基础上，复习工具选型、迁移操作和生产实践。

**示例范围：NestJS + PostgreSQL；TypeORM CLI 按 0.3 系列、Prisma Migrate 按 Prisma ORM 7 系列说明。** 不同主版本的配置与命令可能变化，应以目标项目的锁文件和对应版本文档为准。本仓库是文档站，没有安装这些 ORM，也没有执行下述数据库操作。

## 一、它们分别是什么？

### TypeORM：用实体类和 Repository 组织数据访问

TypeORM 是 TypeScript / JavaScript ORM，常用装饰器定义 Entity、列、关系和索引，再用 Repository、EntityManager 或 QueryBuilder 读写数据。

- 支持 **Data Mapper**：实体与持久化操作分离，通过 Repository 访问，适合 NestJS 的依赖注入与分层。
- 也支持 **Active Record**：实体继承 `BaseEntity`，直接调用持久化方法；团队应明确主要风格，避免职责混乱。
- NestJS 通过 `@nestjs/typeorm` 提供模块注册和 `@InjectRepository()` 注入。
- 动态条件、复杂 SQL 和手写 migration 较直接，但需要理解实体状态、关系加载和实际生成的 SQL。

### Prisma：用 Schema 描述模型并生成查询客户端

Prisma ORM 的常见工作流是：在 `schema.prisma` 描述模型，通过 `prisma generate` 生成类型化 Prisma Client，用 `prisma.user.findMany()` 等 API 查询，并由 Prisma Migrate 管理迁移。

- 查询输入与返回类型会根据 Schema 和 `select` / `include` 推导，能在编译期发现不少字段使用错误。
- 类型安全不代表数据库数据已通过运行时业务校验，也不会自动补齐对象级授权。
- NestJS 中通常封装 `PrismaService` / Module，通过 DI 复用客户端和连接池，而不是每次请求创建一个客户端。
- 复杂查询可以使用参数化原始 SQL；超出 Schema 表达能力的 PostgreSQL 特性，可能需要在迁移 SQL 中维护。

Prisma 的客户端生成器、输出目录、连接配置和驱动适配器会随主版本变化。特别是使用 v7 时，不要直接照搬旧教程中的客户端构造与 datasource URL 配置；应按该版本完成生成器、`prisma.config.ts` 和数据库驱动的配置。

## 二、差异怎么记？

| 维度 | TypeORM | Prisma |
| --- | --- | --- |
| 模型入口 | 常用 TS Entity 类和装饰器 | 常用 Prisma Schema，生成客户端 |
| 日常访问 | Repository / EntityManager / QueryBuilder | 生成的 Prisma Client 查询 API |
| 类型反馈 | Entity 和泛型提供类型，原始查询等边界需额外约束 | Schema 驱动输入与结果推导，但原始 SQL 及运行时数据仍有边界 |
| NestJS 集成 | `@nestjs/typeorm` 提供官方集成模块 | 通常自己封装 Provider / Module |
| 迁移生成 | 比较实体元数据与连接数据库，生成 TS migration | 按 Schema、迁移历史和开发库状态，生成 SQL migration |
| 迁移执行 | `up()`；可提供 `down()` 做逆向变更 | 开发用 `migrate dev`，部署用 `migrate deploy` |
| 复杂数据库能力 | QueryBuilder、原始 SQL、手写迁移 | Client、原始 SQL、自定义迁移 SQL |

**不能仅凭 ORM 判断性能高低。** 关联加载可能涉及 JOIN、额外查询或批量查询，具体取决于 API、版本和加载策略。看实际 SQL、执行计划、查询次数、返回列和连接池，再比较真实负载。

选型可以这样落地：

- 已有 TypeORM、团队熟悉 Entity / Repository、需要较多 QueryBuilder：继续 TypeORM 通常更经济。
- 希望通过统一 Schema 和生成客户端提升查询开发体验：可以考虑 Prisma。
- 大量依赖 PostgreSQL 特性或复杂报表：先做小规模验证，确认 ORM 如何表达、如何迁移、如何观测，而不是先押注“哪个更快”。

同一套表结构应有明确的迁移负责人和迁移工具。不要让 TypeORM、Prisma 和手工改表同时独立管理，形成多个结构来源。

## 三、先分清六件事

| 操作 | 改变什么 | 需要记住的边界 |
| --- | --- | --- |
| Schema / Entity | 代码中的期望模型 | 改代码不等于数据库已改变 |
| Client generate | 生成查询客户端 | 不等于执行迁移或建表 |
| Migration | 有顺序、可审查的结构或数据变更 | 文件、执行记录和真实数据库要保持一致 |
| Introspection / `db pull` | 从现有数据库反向得到模型 | 不自动补齐历史迁移 |
| Seed | 创建初始化或测试数据 | 要区分环境并设计幂等性，不能向生产灌入测试账号 |
| Data backfill | 为历史数据补值或转换 | 大数据量通常独立、分批执行，并记录进度 |

### `synchronize`、`db push` 为什么不能代替正式迁移？

TypeORM `synchronize` 会按模型尝试同步结构；Prisma `db push` 用于快速把 Schema 推到数据库，不建立正式的 migration 文件历史。适合可丢弃的原型库，但在有重要数据、多人协作或生产发布的环境中，不能当作受审查的版本化迁移流程。

生产 TypeORM 通常设置 `synchronize: false`；由独立迁移任务管理变更，而不是每个应用副本启动时都自动改表。迁移文件记录**变更过程**，ORM 模型记录**目标状态**，二者职责不同。

## 四、TypeORM 0.3：常用操作

### 1. CLI 使用独立 DataSource

CLI 不会自动启动 Nest 应用，也不能仅依靠 Nest 的 `autoLoadEntities`。需要提供导出的 DataSource，明确连接、实体和迁移路径。

下面假设练习项目使用 CommonJS，已配置 `User` 实体、`DATABASE_URL` 和 TS 执行依赖。配置文件位于 `src/database/data-source.ts`：

```typescript
import { join } from 'node:path'
import { DataSource } from 'typeorm'
import { User } from '../users/user.entity'

const url = process.env.DATABASE_URL
if (!url) throw new Error('Missing DATABASE_URL')

export default new DataSource({
  type: 'postgres',
  url,
  entities: [User],
  migrations: [join(__dirname, 'migrations', '*.{ts,js}')],
  synchronize: false,
  migrationsRun: false,
})
```

这是目标练习项目的配置示例，不是对本站环境的修改。CLI 与 Nest 运行时应共享必要的连接与命名约定，但迁移文件不能依赖 Nest 启动或调用会持续变化的业务 Service。凭证、TLS 和配置加载由目标项目管理，避免指向错误数据库或在日志中泄露连接串。

### 2. 开发环境生成、检查、执行

以下命令只针对已安装 TypeORM、`ts-node` 和 PostgreSQL 驱动的练习项目；连接应指向本地开发库：

```bash
# 修改 Entity 后，比较模型与当前数据库并生成迁移文件
pnpm exec typeorm-ts-node-commonjs migration:generate ./src/database/migrations/AddDisplayName -d ./src/database/data-source.ts

# 查看迁移状态
pnpm exec typeorm-ts-node-commonjs migration:show -d ./src/database/data-source.ts

# 审查生成的 up/down 后，执行待应用迁移
pnpm exec typeorm-ts-node-commonjs migration:run -d ./src/database/data-source.ts

# 本地验证：回退最近一次已执行的迁移，不是回退全部历史
pnpm exec typeorm-ts-node-commonjs migration:revert -d ./src/database/data-source.ts
```

没有模型差异时，`migration:generate` 不会凭空生成变更。需要手写迁移时可以使用 `pnpm exec typeorm migration:create ./src/database/migrations/AddDisplayName`，再实现 `up()` / `down()`。

原生 ESM 项目使用相应的 `typeorm-ts-node-esm` 入口，并按该项目的模块解析规则调整路径、扩展名和 `__dirname` 替代方案。不要只把上例命令换名，就认为配置已兼容。

### 3. 生产运行编译后的迁移

构建产物必须包含 DataSource、Entity 和所有迁移文件。假设编译结果确实位于 `dist/database/data-source.js`，迁移任务可执行：

```bash
pnpm exec typeorm migration:show -d ./dist/database/data-source.js
pnpm exec typeorm migration:run -d ./dist/database/data-source.js
```

真实 NestJS / monorepo 的输出路径可能是 `dist/src/...` 或其他位置，应在发布前核对；原生 ESM 产物还需满足 Node 的加载规则。迁移工具及驱动也必须存在于迁移执行环境，不能在裁剪生产依赖后才发现 CLI 缺失。

TypeORM 用迁移记录表跟踪执行状态。默认将整批迁移放进一个事务，也支持 `--transaction each` 或 `--transaction none`。单个迁移的 `transaction = false` 只有在 `each` / `none` 模式下才能按设计生效；不能在外层整批事务中再要求某条语句脱离事务。

`up()` 描述正向操作，`down()` 描述逆向操作；**写了 `down()` 不代表能恢复删掉的数据**。变更类型、删列和数据转换都需要单独评估可逆性。

例如，假设表名映射为 `users`，迁移文件可以表达“新增可空昵称列”：

```typescript
import type { MigrationInterface, QueryRunner } from 'typeorm'

export class AddDisplayName1794873600000 implements MigrationInterface {
  async up(queryRunner: QueryRunner): Promise<void> {
    await queryRunner.query('ALTER TABLE "users" ADD COLUMN "display_name" text')
  }

  async down(queryRunner: QueryRunner): Promise<void> {
    // 逆向删除列会丢失其中数据，只能在已确认可接受的环境执行。
    await queryRunner.query('ALTER TABLE "users" DROP COLUMN "display_name"')
  }
}
```

实际使用 CLI 生成的时间戳类名和文件名，且 Entity 的列映射也要同步为预期结构；不能只手写 SQL、不更新模型，导致下一次生成迁移又试图撤销变更。

## 五、Prisma ORM 7：常用操作

以下假设 Prisma CLI、Client、配置和 PostgreSQL 连接都已在练习项目中按 v7 设置好；命令应使用项目锁定的版本，不临时下载另一个最新版。

### 1. 开发：生成 SQL，再审查应用

```bash
# 检查 Schema 是否有效
pnpm exec prisma validate

# 按修改后的 Schema 生成迁移，先不应用新迁移
pnpm exec prisma migrate dev --name add_display_name --create-only

# 审查 prisma/migrations/<timestamp>_add_display_name/migration.sql 后应用
pnpm exec prisma migrate dev

# 显式重新生成客户端，不把迁移和客户端生成混为一谈
pnpm exec prisma generate

# 查看迁移状态
pnpm exec prisma migrate status
```

`migrate dev` 使用**影子数据库（shadow database）**重放历史、检测漂移，并生成或应用迁移；发现冲突时可能要求 reset。`--create-only` 是先生成新迁移供审查，不代表整个命令没有数据库访问或绝不会出现重置提示。

同一项可空昵称变更，在 Prisma 中先为相应模型增加 `displayName String? @map("display_name")`；若模型通过 `@@map("users")` 映射到 `users` 表，生成的 SQL 可以是：

```sql
ALTER TABLE "users" ADD COLUMN "display_name" TEXT;
```

迁移 SQL 应保留在版本库中；自定义 SQL 还需确认与 Schema 一致。后续收紧非空约束应另建迁移，不修改已在共享环境应用的这份文件。

影子库必须与重要数据库隔离，绝不能把生产库配置为 shadow database。它可能需要创建数据库的权限，或预先配置专用影子库；这不是要求把生产迁移账号也授予超级权限。

### 2. 部署：只应用已提交的迁移

```bash
# 在发布系统的独立迁移任务中执行
pnpm exec prisma migrate deploy
```

`migrate deploy` 应用仓库中尚未执行的迁移，通过 `_prisma_migrations` 跟踪状态；它不根据模型临时生成迁移、不使用影子库、不做完整数据库漂移检测，也不生成 Prisma Client。Client 应在构建阶段显式生成，构建和迁移使用同一份 Schema 与迁移历史。

`migrate reset` 会重置数据库并重放迁移，是开发或可丢弃测试库操作，不能作为生产修复办法。

### 3. 已有数据库与失败恢复

- **接管已有库**：先检查实际结构，按对应版本的 baselining 流程建立初始迁移；验证该库确实已具备这些结构后，才用 `migrate resolve --applied <完整迁移目录名>` 标记基线，避免重复创建表。
- **迁移失败**：先确认哪些 SQL 已生效、是否有部分对象或数据残留。修复或撤销部分结果后，按真实状态选择重试或前向修复。
- **`migrate resolve`**：协调迁移历史记录，不会替你执行或回滚 SQL。`--rolled-back` 标记后重试之前，要确认残留操作不会造成重复；`--applied` 需要确认所有预期变更已完成。
- **回滚**：Prisma Migrate 没有与 TypeORM `migration:revert` 对等的自动生产回退命令。通常通过经审查的新迁移向前修复，必要时另行编写、演练补偿 SQL。

不要默认 Prisma 的 PostgreSQL SQL 迁移会自动整份包进事务；需要事务时审查后显式设计 `BEGIN` / `COMMIT`，并确认所有语句都允许在事务中运行。应用的 `$transaction()` 与迁移文件的事务边界是两回事。

## 六、企业实践：先兼容，再收缩

### 示例：给用户增加必填昵称

假设已有很多用户，希望最终把 `users.display_name` 设为非空。不要直接加非空列后立刻发布只认识新结构的应用，应按 **Expand → Migrate → Contract** 拆分：

1. **Expand（扩展）**：先新增可空列，确保旧应用仍可运行；检查 DDL 锁、表规模和超时。
2. **应用过渡**：发布能兼容空值、同时为新增用户写昵称的版本。旧副本、消费者和定时任务尚未退出前，不能假设所有写入都已更新。
3. **Migrate（数据迁移）**：按主键范围或游标分批回填，使用 `WHERE display_name IS NULL` 等条件保证可重试；控制每批事务、并发、WAL 和复制延迟，记录进度及失败样本。
4. **验证**：检查空值、非法值和业务含义；确认所有写入方均已适配，回填不会被旧写入继续破坏。
5. **Contract（收缩）**：在兼容性窗口结束后收紧约束。若是重命名或替换列，应再分阶段停止旧读写，最后才删除旧列。

结构迁移可以很短，但大规模回填往往是独立作业，不宜放在一次长时间持锁的启动事务中。回填要设计幂等、可暂停、可续跑与审计，不能只检查“脚本退出码为 0”。

### PostgreSQL：DDL 有事务，也有锁和成本

- `ALTER TABLE` 的许多操作需要强锁；可以回滚不代表不会阻塞线上读写。结合操作类型、数据库版本和数据规模评估，并为迁移会话设置合适的锁等待与执行超时。
- 大表索引可以考虑 `CREATE INDEX CONCURRENTLY`，但它**不能在事务块内执行**，仍有扫描和等待成本；失败后可能残留无效索引，需要检查并处理。
- 大表非空校验可评估分阶段 `CHECK ... NOT VALID`、`VALIDATE CONSTRAINT`，再设置 `NOT NULL`；`NOT VALID` 不适用于任意约束，唯一约束也要按其专门方式处理。
- 新增唯一约束前先处理重复数据；改列类型、默认值或重命名要检查重写成本、隐式转换、依赖和 ORM 生成 SQL。工具无法可靠判断你是在重命名还是“删除再新增”。

### 发布过程应获得哪些保障？

| 保障 | 实践要点 |
| --- | --- |
| 历史可重放 | 提交模型、迁移文件和锁文件；Prisma 还包括迁移目录中的工具元数据；已应用的共享迁移不随意修改 |
| 验证两个起点 | 在空 PostgreSQL 上完整重放，也从上一发布版本的代表性数据升级；仅验证空库无法发现真实数据问题 |
| 只执行一次 | 使用单独迁移任务并串行部署；Prisma 的 advisory lock 不是发布编排的替代，TypeORM 也不能假设多副本竞跑安全 |
| 失败阻断 | 迁移失败停止后续发布，保留原错误、执行状态和排障记录，不忽略异常继续启动 |
| 权限隔离 | 应用账号只做必要 DML，迁移账号拥有经过限制的 DDL 权限；两者凭证与审计分开管理 |
| 可观测 | 记录发布版本、迁移编号、目标环境标识、耗时和结果，观测锁等待、连接池、错误率及复制延迟，不记录密码或连接串 |
| 恢复已演练 | 备份与 PITR 需要恢复演练和 RPO / RTO 目标；“有备份”不等于能即时回滚，也不保证保留切换后的新写入 |

以上是目标后端项目的发布设计建议，没有修改本站 CI/CD。迁移任务使用本次发布的固定构建产物；应用滚动发布时，新旧代码必须在过渡结构上共存。不可逆结构或数据变更之后，不能只回滚应用镜像就宣布恢复。

## 七、NestJS 项目中还要记住什么？

- **事务边界在业务层**：TypeORM 事务内使用回调提供的 `EntityManager` / Repository；Prisma 使用该事务的 `tx`。误用全局客户端会让部分操作脱离预期事务。
- **事务保持短小**：不在持锁期间等待模型接口、支付或长时间网络调用；跨服务副作用需要幂等和可靠消息等独立设计。
- **客户端和连接池复用**：一般按应用实例复用，计算总连接数时包含副本、后台作业及迁移任务；关闭时按驱动生命周期释放连接。
- **返回 DTO 与持久化模型分开**：不把 Entity / Client 返回值直接作为全部响应字段；查询时尽量只取需要的列。
- **ORM 不包办安全**：依然需要[参数化查询](../10safe/s_safe_6-sql_injection)、[资源与租户授权](./s_nestjs_13-authorization_models)、数据库约束和错误治理。

## 八、一分钟复述

> TypeORM 和 Prisma 都是数据访问工具。TypeORM 主要用实体类、Repository 和 QueryBuilder，贴近 NestJS 的 DI 和面向对象分层；Prisma 通过 Schema 生成类型化客户端，查询的输入和结果推导比较直观。两者没有绝对性能胜负，关键还是生成的 SQL、索引、事务和查询模式。
>
> 数据库结构变更应使用版本化 migration。TypeORM 生成并执行 `up/down`，Prisma 在开发用 `migrate dev`，部署用 `migrate deploy`。生产不能依赖自动同步，也不能把有回滚命令理解成能恢复数据。企业项目更重视新旧应用兼容、分批回填、单独迁移任务、锁与数据验证，以及可演练的恢复方案。

## 九、复习自测与练习

1. Entity / Schema、migration、Client generate 和 seed 分别改变什么？
2. 为什么不能只测空库迁移？生成的“删列再新增”应该如何检查？
3. `migrate resolve --rolled-back` 与真的执行回滚 SQL 有何不同？
4. 两个应用副本同时启动，谁执行迁移？旧副本还在写入时，回填为何可能永远做不完？
5. 在可丢弃的 PostgreSQL 练习库中演练昵称字段的扩展、回填和非空约束；记录实际 SQL、耗时、锁等待和验证结果。

上述数据库命令和练习未在本站执行；页面构建通过只验证文档与导航，不等于迁移已在真实数据上验证。

> 来源：[NestJS Database](https://docs.nestjs.com/techniques/database)、[TypeORM Migrations](https://typeorm.io/docs/advanced-topics/migrations/)、[TypeORM Transactions](https://typeorm.io/docs/advanced-topics/transactions/)
>
> 版本化参考：[TypeORM 0.3.20 Migrations](https://github.com/typeorm/typeorm/blob/0.3.20/docs/migrations.md)（用于核对 0.3 CLI、事务模式与迁移接口，不代表建议固定使用这个旧补丁版本）。
>
> 来源：[Prisma ORM 7：Development and production](https://www.prisma.io/docs/orm/v7/prisma-migrate/workflows/development-and-production)、[Patching and hotfixing](https://www.prisma.io/docs/orm/v7/prisma-migrate/workflows/patching-and-hotfixing)、[Baselining](https://www.prisma.io/docs/orm/v7/prisma-migrate/workflows/baselining)
>
> 来源：[PostgreSQL ALTER TABLE](https://www.postgresql.org/docs/current/sql-altertable.html)、[CREATE INDEX](https://www.postgresql.org/docs/current/sql-createindex.html)、[Backup and Restore](https://www.postgresql.org/docs/current/backup.html)
