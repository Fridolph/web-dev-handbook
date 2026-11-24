# TypeORM & Prisma

> 本篇保留模型与查询入门。系统复习工具差异、迁移命令及生产实践，见[TypeORM、Prisma 与数据库迁移复习](./s_nestjs_14-orm_migrations)。代码是概念片段，客户端构造和连接配置需匹配所用 ORM 的主版本。

## 概念

ORM（Object-Relational Mapping）帮助在关系数据与应用对象之间建立映射，不一定都使用 TypeScript 类定义模型。TypeORM 常用装饰器 Entity，Prisma 则通过 Schema 生成客户端。两者都是 NestJS 项目中常见的选择。

ORM 是实现数据访问的工具，不等同于 DAO / Repository 的职责设计；也不能代替索引、事务和 SQL 知识。概念区分见[MVC 与 DTO / DAO 分层](./s_nestjs_11-layering_dto_dao)。

## TypeORM

TypeORM 同时支持 Active Record 与 Data Mapper，可通过 `@nestjs/typeorm` 集成 NestJS。下面的普通 Entity 配合 Repository 使用，属于 Data Mapper 风格；Active Record 通常由实体继承 `BaseEntity` 并直接调用持久化方法。

```typescript
import { Entity, Column, PrimaryGeneratedColumn, OneToMany } from 'typeorm'

@Entity()
export class User {
  @PrimaryGeneratedColumn('uuid')
  id: string

  @Column({ unique: true })
  email: string

  @Column()
  password: string

  @Column({ default: true })
  isActive: boolean

  @OneToMany(() => Post, post => post.author)
  posts: Post[]
}
```

Repository 模式：

```typescript
@Injectable()
export class UsersService {
  constructor(
    @InjectRepository(User)
    private usersRepo: Repository<User>,
  ) {}

  async findByEmail(email: string) {
    return this.usersRepo.findOne({ where: { email } })
  }

  async findWithPosts(id: string) {
    return this.usersRepo.findOne({
      where: { id },
      relations: ['posts'],
    })
  }
}
```

## Prisma

Schema-first 方式，更直观：

```prisma
// prisma/schema.prisma
model User {
  id        String   @id @default(uuid())
  email     String   @unique
  password  String
  isActive  Boolean  @default(true)
  posts     Post[]
  createdAt DateTime @default(now())
}

model Post {
  id        String   @id @default(uuid())
  title     String
  content   String?
  author    User     @relation(fields: [authorId], references: [id])
  authorId  String
}
```

使用 Prisma Client：

```typescript
@Injectable()
export class PrismaService extends PrismaClient implements OnModuleInit {
  async onModuleInit() {
    await this.$connect()
  }
}

@Injectable()
export class UsersService {
  constructor(private prisma: PrismaService) {}

  async findByEmail(email: string) {
    return this.prisma.user.findUnique({ where: { email } })
  }

  async findWithPosts(id: string) {
    return this.prisma.user.findUnique({
      where: { id },
      include: { posts: true },
    })
  }
}
```

## TypeORM vs Prisma

| 维度 | TypeORM | Prisma |
|------|---------|--------|
| 设计方式 | Code First（装饰器） | Schema First（.prisma 文件） |
| TypeScript 类型 | Entity 与泛型提供类型，部分边界需显式约束 | Schema 生成客户端，推导查询输入与返回值 |
| 迁移（Migration） | 根据实体与数据库差异生成迁移，再执行 | 开发生成并应用，部署执行已提交的迁移 |
| 关联查询 | relations 选项 | include 语法 |
| 学习曲线 | 较高（概念多） | 中等 |
| 查询性能 | 都需要关注 SQL、索引、关联加载与 N+1 | 都需要关注 SQL、索引、关联加载与 N+1 |
| 社区 | 成熟 | 快速增长 |
| NestJS 集成 | @nestjs/typeorm | 需手动封装 Module |

## 迁移学习入口

TypeORM 0.3 CLI 使用迁移输出路径及 `-d` 指定 DataSource，不能照搬旧教程中的 `migration:generate -n ...`。Prisma 开发使用 `migrate dev`，部署使用 `migrate deploy`，客户端生成是另一个操作。

完整命令、版本前提、事务与回滚边界，统一维护在[ORM 对比与数据库迁移](./s_nestjs_14-orm_migrations)。生产变更还应检查锁、历史数据、新旧应用兼容、单独迁移任务及恢复演练，不能只归结为“备份后执行”。

## 面试常问

- TypeORM 和 Prisma 的核心设计差异？各适合什么场景？
- 如何处理 N+1 查询问题？
- 数据库迁移在生产环境的安全实践？

## 参考

- [TypeORM 文档](https://typeorm.io/)
- [Prisma 文档](https://www.prisma.io/docs)
- [NestJS: Database](https://docs.nestjs.com/techniques/database)
