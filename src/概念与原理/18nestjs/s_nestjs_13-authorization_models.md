# 权限控制：RBAC、ABAC 与 CASL 实践

> 认证回答“你是谁”，授权回答“你能对什么资源做什么操作”。登录成功只代表身份已确认，不代表可以访问所有接口或数据。本文先梳理权限模型与授权分层，再用 CASL 串起 NestJS 的接口、对象、字段和数据权限。

## 一、先建立授权的共同语言

一次授权判断至少包含四个要素：

```text
主体 Subject + 操作 Action + 资源 Resource + 上下文 Context
                         ↓
                  Policy / Policy Decision
                         ↓
                    Allow 或 Deny
```

- **主体**：用户、服务账号或其他调用方；认证后通常放在 `request.user`。
- **操作**：读取、创建、修改、删除、发布、导出等，不要只用一个笼统的“访问”。
- **资源**：接口、订单、简历、租户、文件等，既包括接口级资源，也包括具体数据对象。
- **上下文**：租户、资源所有者、时间、来源网络、设备状态和数据状态等。

权限模型决定“如何计算允许”。JWT 是令牌格式，Session 是会话机制，OAuth 是委托授权框架，分别为凭证、会话和委托提供机制，不能代替具体资源的业务授权。OAuth 的 `scope` 也不能单独证明调用方可以访问某个订单或文档。

这些术语不是完全同层的互斥选项：DAC、MAC 强调谁控制授权，ACL 强调授权如何记录，RBAC、ABAC、ReBAC 强调决策依据。同一个系统可以同时使用多种机制。

## 二、主流权限模型对比

### 1. ACL：把权限挂在资源上

Access Control List（访问控制列表）为每个资源记录主体或主体组可以执行的操作：

```text
document-1:
  user-a: read, write
  user-b: read
```

它适合文件、对象共享等“谁能访问这一项”的场景，判断直观；但资源和主体很多时，列表维护、继承和批量变更会变复杂。ACL 是一种授权数据组织方式，RBAC 可以作为 ACL 中主体组的来源。

### 2. DAC：资源所有者可以授权

Discretionary Access Control（自主访问控制）由资源所有者决定谁可以访问，例如文档作者分享给其他用户。它灵活，但若所有者权限过大，可能出现权限扩散；企业系统通常还要叠加组织边界、审计和管理员策略。

### 3. MAC：按安全标签强制控制

Mandatory Access Control（强制访问控制）由系统管理员定义安全级别或标签，主体和资源都带标签，用户不能自行放宽规则。典型思路是“机密级主体不能读取更高密级资源”。它适合安全等级明确的系统，但业务配置和迁移成本较高，普通 SaaS CRUD 很少单独采用。

### 4. RBAC：用户通过角色获得权限

Role-Based Access Control（基于角色的访问控制）将关系拆成：

```text
User → Role → Permission(Action + Resource)
```

例如 `editor` 拥有 `article:read` 和 `article:update`，用户只需绑定角色。RBAC 的优点是易理解、易审计、适合组织岗位稳定的后台系统。常见扩展包括角色层级，以及带约束的职责分离：静态约束可禁止同时授予冲突角色；“提交人不能审批自己的申请”还需要检查具体对象与操作历史，不能仅凭角色名保证。

关系型数据库中常见的概念模型是 `users`、`roles`、`permissions`、`user_roles`、`role_permissions` 五张表。这里是设计示意，不是本项目的数据库迁移。多租户系统还应把角色分配绑定到租户或项目，例如 `user_roles(user_id, role_id, tenant_id)`；某用户在租户 A 是管理员，不代表在租户 B 也有管理权限。

它的边界是：角色数量可能膨胀，难以表达“只能修改自己创建的文章”“工作时间内才允许导出”“同一租户内才可读取”等对象和上下文条件。把所有条件硬编码成角色，最终会形成 `tenant_a_editor_working_hours` 一类难维护的角色。

### 5. ABAC：根据属性和策略计算

Attribute-Based Access Control（基于属性的访问控制）根据主体、资源、操作和环境属性判断：

```text
allow if user.tenantId == document.tenantId
        && user.department == document.department
        && action in ['read', 'update']
        && document.status != 'archived'
```

以上是策略伪代码，前提是所需属性均已验证且完整；缺失属性、查询失败或策略计算错误时应拒绝，不能让两个缺失的 `tenantId` 因为“相等”而放行。

ABAC 能表达租户隔离、资源归属、数据状态、时间和风险等级等细粒度规则，适合规则变化多的系统；代价是策略语言、属性来源、冲突处理、调试和审计都更复杂。属性必须来自可信服务端或数据库，不能直接相信客户端提交的 `role`、`ownerId` 或 `isAdmin`。

### 6. ReBAC：根据关系图判断

Relationship-Based Access Control（基于关系的访问控制）根据主体与资源的关系授权，例如“项目成员可以读项目内文档”“文档作者可以编辑”。它适合协作软件、组织层级和共享关系复杂的产品。关系可用关系型表存储，复杂场景再考虑图查询或专门关系服务，不要求必须使用图数据库；还要关注关系递归、继承和撤销后的一致性。

### 7. PBAC 与 Capability

- **PBAC（Policy-Based Access Control，本文指基于策略的访问控制）**强调把规则集中为可审计的 Policy，由策略引擎根据输入作出决定；策略可以结合角色、属性和关系。这个缩写在不同资料中也有其他含义，交流时应说明全称。
- **Capability（能力令牌）**把“对某个资源执行某个动作的能力”作为不可伪造的令牌传递。它适合临时分享、对象级委托等场景，但令牌范围、过期、撤销和转发风险需要明确设计。

## 三、如何选择和组合？

| 模型 | 判断重点 | 优势 | 常见边界 |
| --- | --- | --- | --- |
| ACL / DAC | 资源与主体的直接授权 | 对象共享直观 | 大规模变更和继承复杂 |
| MAC | 安全标签与强制规则 | 约束强、适合分级数据 | 灵活业务配置成本高 |
| RBAC | 用户拥有哪个角色 | 简单、可审计、易落地 | 对资源归属和上下文表达不足 |
| ABAC | 属性与策略条件 | 细粒度、适合多租户 | 策略和属性治理复杂 |
| ReBAC | 主体与资源的关系 | 适合协作和组织关系 | 关系存储、查询和一致性复杂 |

多数业务不需要一次选一个“纯模型”。RBAC 和 ABAC 也不只是从低级到高级的替代关系，角色本身可以是 ABAC 的一个主体属性。常见演进是：

1. 用 RBAC 表达菜单和接口的基础权限。
2. 用权限（`article:update`）替代散落的角色字符串，便于角色组合。
3. 在 Service 或策略层加入资源归属、租户和数据状态等 ABAC / ReBAC 条件。
4. 当策略跨服务、规则频繁变化或需要统一审计时，再评估 CASL、Casbin、OPA 等策略库或服务。

## 四、NestJS 中的授权分层

```text
Middleware
  ↓
认证 Guard：验证 Session / JWT，建立 request.user
  ↓
授权 Guard：检查角色、权限或策略
  ↓
Controller / Service：检查具体资源归属与状态
  ↓
Repository：用当前身份和租户条件限制查询、更新或删除
```

### 1. 用元数据声明粗粒度权限

NestJS 提供 `SetMetadata`、`Reflector` 和 Guard 来读取路由元数据。下面是学习片段，真正项目还需要定义权限常量、缺省行为和审计策略：

```typescript
import { Injectable, SetMetadata } from '@nestjs/common'
import type { CanActivate, ExecutionContext } from '@nestjs/common'
import { Reflector } from '@nestjs/core'

export const PERMISSIONS_KEY = 'permissions'
export const Permissions = (...permissions: string[]) =>
  SetMetadata(PERMISSIONS_KEY, permissions)

@Injectable()
export class PermissionsGuard implements CanActivate {
  constructor(private readonly reflector: Reflector) {}

  canActivate(context: ExecutionContext) {
    const required = this.reflector.getAllAndOverride<string[]>(
      PERMISSIONS_KEY,
      [context.getHandler(), context.getClass()],
    )

    // 默认拒绝：公开接口应使用单独的 @Public 元数据显式放行。
    if (!required || required.length === 0) return false

    const request = context.switchToHttp().getRequest()
    const granted = new Set(request.user?.permissions ?? [])
    return required.every(permission => granted.has(permission))
  }
}

```

在已配置的 Controller 中，可以用 `@Permissions('article:update')` 声明要求，再由 `@UseGuards(JwtAuthGuard, PermissionsGuard)` 保证先认证、后检查权限。示例假设可信的 `request.user.permissions` 是字符串数组；实际还需验证身份上下文的结构。它仅在已绑定守卫的路由上生效；全站默认保护还需注册全局守卫，并显式设计公开接口。

`request.user.permissions` 必须来自服务端验证后的身份或权限查询，并限制为当前租户或项目内的权限，不能合并所有租户的权限，更不能由请求体直接注入。角色守卫只是 RBAC 的一种实现，不能自动完成对象级授权。

这里使用 `getAllAndOverride` 表示“方法级元数据覆盖 Controller 级元数据”；如果业务需要把两级权限合并，应改用 `getAllAndMerge`，并明确多个权限是“全部满足”还是“满足其一”。公开接口也要显式声明，避免因漏写元数据而意外放行。

### 2. 在业务层检查资源归属

“用户拥有 `article:update`”不等于“可以修改任意文章”。Service 应使用可信的当前用户和租户条件查询资源，并在修改时再次带上条件：

```typescript
const article = await repository.findOne({
  where: { id: articleId, tenantId: currentUser.tenantId },
})

if (!article || article.ownerId !== currentUser.id) {
  throw new ForbiddenException('无权修改该资源')
}
```

更稳妥的更新可以把 `id`、`tenantId` 和归属条件放进同一条 `UPDATE ... WHERE`，检查受影响行数，避免先检查后更新之间的竞态。数据库约束或 PostgreSQL Row-Level Security 可以作为纵深防御，但不能替代应用层对业务状态和错误响应的设计。

如果“只能修改草稿”也是条件，应同时约束状态；跨多表策略可能还需要事务或锁。RLS 也并非开箱即保护所有查询：需正确配置策略和连接身份，注意表所有者、超级用户及 `BYPASSRLS` 角色的绕过规则。

### 3. 策略决策与执行分开

规模扩大后，可以把“是否允许”的纯判断抽成 `AuthorizationService` 或 Policy Handler，再由 Guard / Service 调用。策略输入应记录主体、动作、资源摘要、租户和决策原因；日志不要包含密码、完整令牌等敏感内容。策略拒绝默认返回 403，认证失败通常返回 401；对不应暴露资源存在性的接口，也可以统一返回 404。

常见术语中，**PDP（策略决策点）**负责计算允许或拒绝，**PEP（策略执行点）**负责阻断或放行，策略管理和可信属性获取则分别常称为 PAP、PIP。它们是职责划分，可以先放在同一个进程中，不要求拆成多个服务。

多条策略同时适用时，要明确组合规则。例如“权限允许 AND 同租户 AND 对象策略允许”；如采用“显式拒绝优先”，任何强制拒绝都应生效，不能简单把多个结果做 OR。策略服务超时或属性读取失败时默认拒绝。权限缓存键应包含主体、租户、资源范围和策略版本等必要维度，并设计撤销与失效机制。

## 五、CASL：把权限模型变成可执行规则

### 1. CASL 与 RBAC / ABAC 是什么关系？

CASL 是 JavaScript / TypeScript 授权库，核心包是 `@casl/ability`。**RBAC、ABAC 描述授权依据，CASL 帮我们声明和计算这些规则**，它不是新的认证方式，也不负责登录、签发 JWT 或自动创建角色权限表。

例如：根据用户的 `editor` 角色赋予编辑能力，属于 RBAC；再要求文章与用户同租户、文章处于草稿状态，属于 ABAC。两者可以在同一个 Ability 中表达。复杂协作关系仍需要业务查询或关系服务提供可信属性，CASL 不会自动遍历组织关系图。

在前文的职责划分中，CASL 承担进程内的策略决策；Guard、Service 和查询代码负责执行结果。它不要求额外搭建授权服务器，也不自带完整的策略管理后台。

### 2. 一条规则如何组成？

```typescript
can('update', 'Article', ['title', 'content'], {
  tenantId: actor.tenantId,
  ownerId: actor.id,
  status: 'draft',
})
```

| 部分 | 示例 | 含义 |
| --- | --- | --- |
| Action | `update` | 做什么操作；`publish`、`export` 等也可自行定义 |
| Subject | `Article` | 对什么类型的资源操作，不是当前登录用户 |
| Fields | `title`、`content` | 可以操作哪些字段，可省略 |
| Conditions | 租户、作者、状态 | 哪些具体资源符合条件，可省略 |

`AbilityBuilder` 用来收集 `can` / `cannot` 规则，`build()` 生成 Ability，随后通过 `ability.can(action, resource, field?)` 判断。没有匹配的允许规则时返回 `false`。`manage` 和 `all` 默认分别表示任意操作和任意资源；授予 `manage all` 范围很大，不能当作普通租户管理员的默认配置。

`createMongoAbility` 使用 Mongo 风格的条件操作符，例如 `$in`、`$ne`，**不要求数据库使用 MongoDB**。在 PostgreSQL + TypeORM 项目中，同样可以用它检查内存中的对象；但这不代表条件会自动变成 SQL。

### 3. 规则匹配原理与三个关键边界

CASL 根据操作和资源类型建立规则索引，检查时再匹配字段与条件，而不是每次从数据库加载角色权限。条件判断基于传入对象，属性获取和数据库访问由应用负责。

- **后声明且匹配的规则优先**。`cannot` 是拒绝规则，不是全局、无条件的“拒绝优先”。如果最后再写一个宽泛的 `can('manage', 'all')`，可能覆盖前面的限制。需要保留的限制应最后声明，并用测试固定顺序；租户隔离等强制边界还应落实在查询中。
- **类型检查不等于对象检查**。`ability.can('update', 'Article')` 可能因为存在带条件的允许规则而返回 `true`，此时尚未验证某篇文章的作者、租户和状态。对象权限必须用实际可信记录进行判断。
- **对象权限不等于字段权限**。允许更新文章，不代表可以改 `ownerId`、`tenantId` 或任意字段。需要指定字段检查，并控制实际写入和返回的字段。CASL 不会自动删除请求体或响应中的敏感字段。

普通 JSON 对象没有可靠的类名，可用 `subject('Article', article)` 显式标记资源类型；该名称必须与规则一致。不要把客户端提交的“文章对象”直接作为授权依据，应使用数据库记录或服务端构造的可信对象。

## 六、NestJS 集成：从 Ability 到实际写入

> 版本范围：以下核心示例采用 **`@casl/ability` 7.x** 的 `createMongoAbility` / `MongoAbility`，使用字符串资源名与 `subject()` 标记。旧课程中的 `PureAbility` 等类型写法可能不同。以下是业务项目的学习片段，本站未安装 CASL / NestJS，也未接入数据库；代码中的实体、认证 Guard 和 DTO 需要在实际项目中补齐。

### 1. AbilityFactory：根据可信身份构建规则

```typescript
// casl-ability.factory.ts
import { Injectable } from '@nestjs/common'
import { AbilityBuilder, createMongoAbility } from '@casl/ability'
import type { ForcedSubject, MongoAbility } from '@casl/ability'

export interface Actor {
  id: string
  tenantId: string
  // 由服务端解析的当前租户角色，不是所有租户的角色合集。
  roles: readonly string[]
}

export interface ArticleResource {
  id: string
  tenantId: string
  ownerId: string
  status: 'draft' | 'published' | 'archived'
  title: string
  content: string
  version: number
}

type Action = 'read' | 'update'
type ArticleSubject = ArticleResource & ForcedSubject<'Article'>
export type AppAbility = MongoAbility<[Action, 'Article' | ArticleSubject]>

@Injectable()
export class CaslAbilityFactory {
  createForUser(actor: Actor): AppAbility {
    const { can, cannot, build } =
      new AbilityBuilder<AppAbility>(createMongoAbility)

    // 认证层还应验证角色的结构和有效性；缺少关键身份属性时拒绝授权。
    if (!actor.id || !actor.tenantId) return build()

    can('read', 'Article', { tenantId: actor.tenantId })

    if (actor.roles.includes('editor')) {
      can('update', 'Article', ['title', 'content'], {
        tenantId: actor.tenantId,
        ownerId: actor.id,
        status: 'draft',
      })
    }

    // 放在允许规则后面，明确保留归档不可编辑的约束。
    cannot('update', 'Article', { status: 'archived' })
    return build()
  }
}
```

这是一组明确的业务策略：当前租户成员可读本租户文章；`editor` 只能修改自己的草稿，且仅能修改标题和正文。真实系统是否允许读取其他成员草稿，应按产品需求再细分。

Factory 本身可以是无状态的默认单例，**每次根据当前请求身份创建 Ability**；不要在单例上保存“当前用户 Ability”，否则可能串用其他用户的权限。若请求内复用 Ability，应绑定到本次请求；跨请求缓存则需要主体、租户和策略版本等完整键及失效机制。

### 2. Policy Decorator + Guard：声明接口所需能力

```typescript
// policies.guard.ts
import { Injectable, SetMetadata } from '@nestjs/common'
import type { CanActivate, ExecutionContext } from '@nestjs/common'
import { Reflector } from '@nestjs/core'
import { CaslAbilityFactory } from './casl-ability.factory'
import type { Actor, AppAbility } from './casl-ability.factory'

type PolicyHandler = (ability: AppAbility) => boolean
const POLICIES_KEY = 'casl:policies'
export const CheckPolicies = (...handlers: PolicyHandler[]) =>
  SetMetadata(POLICIES_KEY, handlers)

@Injectable()
export class PoliciesGuard implements CanActivate {
  constructor(
    private readonly reflector: Reflector,
    private readonly abilityFactory: CaslAbilityFactory,
  ) {}

  canActivate(context: ExecutionContext): boolean {
    const handlers = this.reflector.getAllAndOverride<PolicyHandler[]>(
      POLICIES_KEY,
      [context.getHandler(), context.getClass()],
    )
    const request = context.switchToHttp()
      .getRequest<{ user?: Actor }>()

    // 先认证；此处按默认拒绝设计，漏配策略不放行。
    if (!request.user || !handlers?.length) return false

    const ability = this.abilityFactory.createForUser(request.user)
    return handlers.every(handler => handler(ability))
  }
}
```

在业务模块的 `providers` 中注册 `CaslAbilityFactory`、`PoliciesGuard`；跨模块复用时可封装 `CaslModule`，导出所需 Provider，再由业务模块导入。Nest 的 `@Injectable()` 不能代替模块注册。

Controller 使用示意如下；`JwtAuthGuard` 是项目已有的认证 Guard，`UpdateArticleDto` 应通过全局 ValidationPipe 做运行时校验：

```typescript
@UseGuards(JwtAuthGuard, PoliciesGuard)
@CheckPolicies(ability => ability.can('update', 'Article'))
@Patch(':id')
update(
  @Req() request: { user: Actor },
  @Param('id') id: string,
  @Body() dto: UpdateArticleDto,
) {
  return this.articlesService.update(request.user, id, dto)
}
```

这里 Guard 只判断“是否可能具有文章更新能力”，不能拿它证明某条记录可更新。认证 Guard 先建立可信身份，对象与字段检查放在 Service 中执行。Guard 在 Pipe 之前执行，也不能把尚未校验的请求体当作已验证数据。

此示例只保护显式绑定 Guard 的路由；全站保护需配置全局守卫，并设计 `@Public`。PolicyHandler 暂限定同步布尔函数；如果以后支持异步策略，需显式 `await` 后组合结果，不能直接把 Promise 放进 `every()`，否则可能被当作真值放行。

### 3. Service：检查对象和字段，再执行条件更新

以下是 TypeORM Service 的核心方法片段，假设 `this.articles` 是 `Repository<ArticleEntity>`，实体字段与 `ArticleResource` 一致，`this.abilityFactory` 已注入。方法涉及的异常来自 `@nestjs/common`，`subject` 来自 `@casl/ability`。

```typescript
async update(
  actor: Actor,
  id: string,
  dto: { title?: string; content?: string },
) {
  const ability = this.abilityFactory.createForUser(actor)
  const article = await this.articles.findOneBy({
    id,
    tenantId: actor.tenantId,
  })
  if (!article) throw new NotFoundException('资源不存在')

  const resource = subject('Article', article)
  if (!ability.can('update', resource)) {
    throw new ForbiddenException('无权修改该文章')
  }

  // 演示仅允许平面字段；不把整个 dto 传给 ORM。
  const patch: { title?: string; content?: string } = {}
  for (const key of Object.keys(dto)) {
    if (key !== 'title' && key !== 'content') {
      throw new BadRequestException('包含不支持的字段')
    }
    if (!ability.can('update', resource, key)) {
      throw new ForbiddenException('无权修改该字段')
    }
    const value = dto[key]
    if (value !== undefined) patch[key] = value
  }
  if (!Object.keys(patch).length) {
    throw new BadRequestException('没有可更新的字段')
  }

  // 将本例的归属、状态、版本限制落实到同一条 UPDATE 的 WHERE。
  const result = await this.articles.update(
    {
      id,
      tenantId: actor.tenantId,
      ownerId: actor.id,
      status: 'draft',
      version: article.version,
    },
    { ...patch, version: article.version + 1 },
  )
  if (result.affected !== 1) {
    throw new ConflictException('资源状态已变化，请刷新后重试')
  }
  return { id, version: article.version + 1 }
}
```

DTO 应使用 class 和校验装饰器；推荐配置 `whitelist: true`、`forbidNonWhitelisted: true`，拒绝未声明字段。**DTO 校验解决数据是否合法，CASL 解决当前主体能否操作该字段**，两者不能互相替代。嵌套对象、关联写入和批量更新还要逐层限制，不能只检查顶层键。

CASL 检查与写入是两个步骤，单独检查可能产生 TOCTOU（检查与使用之间的状态变化）。以上通过条件更新约束本例的资源属性，并要求其他写入路径遵守同一版本协议；它不会自动解决跨表成员关系变化或权限即时撤销。复杂策略需要事务、锁或其他一致性方案。

这里的条件是**针对本例手工映射**，并非任意 CASL 规则的 SQL 翻译器。以后允许编辑他人文章或增加管理员规则时，Ability 和更新条件必须一起调整并回归，避免决策与实际查询不一致。

## 七、数据权限：列表、分页与 ORM 集成

### 1. TypeORM：不能先分页，再用 CASL 过滤

对详情检查对象还不够，列表、统计和导出必须在数据库查询阶段限制范围。先查出所有数据再 `.filter(ability.can)` 会浪费资源；先分页再过滤还会造成数量错误、空页，甚至通过总数泄露其他租户的数据。

本例的读取策略是“只读当前租户文章”，可明确映射为：

```typescript
const [items, total] = await repository.findAndCount({
  where: { tenantId: actor.tenantId },
  select: { id: true, title: true, status: true },
  order: { id: 'ASC' },
  skip: offset,
  take: limit,
})
```

详情、列表、总数、聚合、批量操作都应使用一致的范围。字段权限也不会自动影响 `SELECT` 或序列化，响应仍需 DTO / 显式投影。不要通过拼接 SQL 或直接照搬 `ability.rules` 构造查询；规则中可能包含允许、拒绝、优先级和嵌套条件，需要完整且经过验证的转换语义。

### 2. Prisma：使用对应版本的查询适配器

`@casl/prisma` 提供 `createPrismaAbility` 和 `accessibleBy`，用 Prisma `WhereInput` 描述条件，并组合成数据库过滤条件。它不是把前面的 Mongo 风格 Ability 原样传给 Prisma，条件语法和类型定义需要一起更换。

**以下片段按 `@casl/ability` 7.x + `@casl/prisma` 2.x 编写**，假设项目已定义对应的 `PrismaAbility`，并在实际使用的 Prisma Client 上配置 `createCaslExtension()`。使用 Prisma 7 自定义生成路径时，应按适配器文档从 `@casl/prisma/runtime` 封装项目类型，不直接复制旧教程的 `@prisma/client` 类型导入。

```typescript
// prismaAbility 由 createPrismaAbility 创建，不能传前面的 MongoAbility。
const rows = await prisma.article.findMany({
  where: {
    AND: [
      accessibleBy(prismaAbility, 'read').ofType('Article'),
      { tenantId: actor.tenantId },
      { status: 'published' },
    ],
  },
  select: { id: true, title: true },
  orderBy: { id: 'asc' },
  take: 20,
})
```

权限条件与业务筛选使用 `AND` 合并，不能把授权约束展开后又覆盖。在 2.x 中，无允许规则时会生成特殊空条件，需要配套 Client 扩展处理为“无可访问记录”；查询适配错误必须向上报错，**绝不能捕获后退回无过滤查询**。旧版 1.x 常见 `accessibleBy(ability).Article` 并抛 `ForbiddenError`，不要把两套 API 和错误语义混用。

适配器也有操作符和运行时解释限制，复杂关系、JSON / 数组条件需对照对应版本验证。它负责查询条件，不会自动完成字段裁剪、关联写入授权或创建资源的安全校验。创建时应在服务端设置租户和作者，再根据可信候选对象检查创建权限；不能让客户端自行选择受保护的归属字段。

## 八、工程实践与验证清单

1. **规则有归属和版本**：集中维护 Action、Subject、角色映射及条件，按业务模块组织策略；策略变更与相应测试一起评审，不把任意客户端输入作为规则加载。
2. **前后端共享语义，服务端最终执行**：前端可通过 `@casl/vue` 控制按钮体验，但后端必须重新校验；下发规则应经过筛选，避免暴露内部策略和敏感属性。
3. **可信属性完整**：角色限制在当前租户；需要的状态、归属和关系来自服务端。查询投影漏掉授权属性时应补齐或拒绝，不能假定“字段缺失就安全”。
4. **缓存和撤销有边界**：角色调整、成员移除、租户切换后要更新 Ability / 缓存；高风险操作不能长期依赖 JWT 中的旧权限声明。时间条件若在构建时计算，也要考虑长期缓存造成的过期。
5. **错误映射明确**：认证失败通常 401，权限拒绝通常 403；CASL 的 `ForbiddenError` 是普通库错误，使用时需转换为 Nest 的 `ForbiddenException` 或由 Filter 映射，否则可能成为 500。对隐藏存在性的资源统一 404。
6. **测试决策与执行的一致性**：除了 `ability.can()` 单元测试，还要测 Guard 顺序、Service、真实 ORM 查询、批量与导出路径；查询编译不支持时应失败关闭。

可从这组用例开始：

| 场景 | 预期 |
| --- | --- |
| 本租户 editor 修改自己的草稿标题 | 允许；条件更新命中一行 |
| 同租户普通成员修改文章 | 拒绝，即使登录有效 |
| editor 修改他人草稿或跨租户文章 | 拒绝；不返回受保护数据 |
| editor 修改自己的已发布 / 归档文章 | 拒绝 |
| 修改 `tenantId`、`ownerId` 或其他未授权字段 | 拒绝且数据库不变 |
| 仅做类型检查，再传入他人文章 | 类型检查可能通过，对象检查仍应拒绝 |
| 检查后文章状态或版本被另一请求修改 | 条件更新失败，不覆盖新状态 |
| 漏配策略、身份缺失、策略查询失败 | 拒绝访问，不执行无过滤查询 |
| 调换宽泛允许与拒绝规则的声明顺序 | 用测试暴露结果变化，防止误以为拒绝恒优先 |

练习时建议在独立 NestJS 项目完成“认证 → Guard → 对象 / 字段检查 → 条件写入 → 数据查询”闭环，并对照[ORM 与数据库迁移](./s_nestjs_14-orm_migrations)理解事务和并发边界。本站不会因为收录了示例就自动具备这些权限能力。

## 九、设计授权时的安全底线

- **默认拒绝、显式允许**：未声明的接口和未知权限不能因为缺少配置而放行。
- **最小权限**：用户、服务账号和数据库账号只获得完成任务所需权限。
- **服务端决策**：前端按钮隐藏只是体验优化，不能作为安全边界。
- **权限与身份分离**：角色、权限和资源关系变化后，要考虑 JWT 旧声明、缓存和会话撤销。
- **对象级授权**：每个读取、修改、删除路径都验证租户、所有者或关系，重点防止 IDOR / BOLA。
- **同时覆盖水平和垂直越权**：用户 A 读取用户 B 的私有记录属于水平越权，普通用户调用管理员接口属于垂直越权；列表、导出、批量操作、字段更新和下载路径也要校验。
- **可审计可测试**：记录关键授权结果和策略版本，并覆盖允许、拒绝、跨租户、失效角色和并发修改场景。

## 十、一分钟复述

> RBAC 是用户通过角色获得权限，简单、易审计，适合稳定的岗位和菜单权限；ABAC 根据主体、资源、操作和环境属性计算策略，能表达租户、资源归属和时间等细粒度规则，但治理成本更高。ACL 适合对象共享，MAC 适合强制安全分级，ReBAC 适合协作关系。NestJS 里通常先用认证 Guard 建立 `request.user`，再用授权 Guard 检查角色或权限，最后在 Service 和查询条件中做资源归属与租户隔离。实际项目往往是 RBAC 加权限，再组合 ABAC 或 ReBAC，而不是把所有规则都塞进角色。

### CASL 集成怎么口头解释？

> CASL 是授权库，RBAC、ABAC 是权限模型。我会先根据当前用户在当前租户的角色和属性构建 Ability，用动作、资源、条件和字段表达规则。NestJS 里先认证，再通过 Guard 检查接口能力；到 Service 时还要传入真实资源，检查归属、状态和字段权限。列表要把权限落实到数据库查询，更新则结合条件和版本避免检查后状态变化。CASL 不替代认证，也不会自动保护 ORM 查询和响应字段；规则顺序、权限撤销和前后端一致性都需要测试。

## 十一、建议练习

以下是学习练习，尚未接入本站运行时：

1. 为文章系统定义 `article:read`、`article:update`、`article:publish`，实现角色到权限的映射。
2. 用两个用户和两个租户验证：同租户编辑、跨租户读取、仅有读取权限的用户分别得到什么结果。
3. 将“作者可编辑草稿、编辑可发布、归档后不可修改”抽成资源策略，并测试并发更新。
4. 检查 JWT 中权限变化后的旧 Token、缓存失效和退出撤销行为。
5. 用 CASL 实现本文文章编辑策略，再验证列表过滤、字段保护和并发条件更新。

学习顺序建议：[认证与 JWT](./s_nestjs_12-authentication_jwt) → RBAC 与权限映射 → 资源归属、租户边界 → ABAC / ReBAC → CASL 对象与字段授权 → 查询约束与策略治理。Guard 与 Pipe 的执行顺序可回看[中间件与守卫](./s_nestjs_5-middleware_guard)。

> 来源：[NestJS Authorization](https://docs.nestjs.com/security/authorization)、[NIST RBAC](https://csrc.nist.gov/projects/role-based-access-control)、[NIST ABAC](https://csrc.nist.gov/projects/attribute-based-access-control)、[OWASP Authorization Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authorization_Cheat_Sheet.html)
>
> 来源：[OpenFGA Authorization Concepts](https://openfga.dev/docs/authorization-concepts)、[PostgreSQL Row Security Policies](https://www.postgresql.org/docs/current/ddl-rowsecurity.html)

> 来源：[CASL 7 文档](https://casl.js.org/v7/en/guide/intro)、[CASL 7.1.3 规则索引与匹配实现](https://github.com/stalniy/casl/tree/%40casl/ability%407.1.3/packages/casl-ability/src)、[CASL Prisma 2.2.0 集成说明](https://github.com/stalniy/casl/blob/%40casl/prisma%402.2.0/packages/casl-prisma/README.md)
