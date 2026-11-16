# 权限控制模型：RBAC、ABAC 与其他方案

> 认证回答“你是谁”，授权回答“你能对什么资源做什么操作”。登录成功只代表身份已确认，不代表可以访问所有接口或数据。本文从模型、决策流程和 NestJS 落地三个层次整理权限控制。

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

## 五、设计授权时的安全底线

- **默认拒绝、显式允许**：未声明的接口和未知权限不能因为缺少配置而放行。
- **最小权限**：用户、服务账号和数据库账号只获得完成任务所需权限。
- **服务端决策**：前端按钮隐藏只是体验优化，不能作为安全边界。
- **权限与身份分离**：角色、权限和资源关系变化后，要考虑 JWT 旧声明、缓存和会话撤销。
- **对象级授权**：每个读取、修改、删除路径都验证租户、所有者或关系，重点防止 IDOR / BOLA。
- **同时覆盖水平和垂直越权**：用户 A 读取用户 B 的私有记录属于水平越权，普通用户调用管理员接口属于垂直越权；列表、导出、批量操作、字段更新和下载路径也要校验。
- **可审计可测试**：记录关键授权结果和策略版本，并覆盖允许、拒绝、跨租户、失效角色和并发修改场景。

## 六、一分钟复述

> RBAC 是用户通过角色获得权限，简单、易审计，适合稳定的岗位和菜单权限；ABAC 根据主体、资源、操作和环境属性计算策略，能表达租户、资源归属和时间等细粒度规则，但治理成本更高。ACL 适合对象共享，MAC 适合强制安全分级，ReBAC 适合协作关系。NestJS 里通常先用认证 Guard 建立 `request.user`，再用授权 Guard 检查角色或权限，最后在 Service 和查询条件中做资源归属与租户隔离。实际项目往往是 RBAC 加权限，再组合 ABAC 或 ReBAC，而不是把所有规则都塞进角色。

## 七、建议练习

以下是学习练习，尚未接入本站运行时：

1. 为文章系统定义 `article:read`、`article:update`、`article:publish`，实现角色到权限的映射。
2. 用两个用户和两个租户验证：同租户编辑、跨租户读取、仅有读取权限的用户分别得到什么结果。
3. 将“作者可编辑草稿、编辑可发布、归档后不可修改”抽成资源策略，并测试并发更新。
4. 检查 JWT 中权限变化后的旧 Token、缓存失效和退出撤销行为。

学习顺序建议：[认证与 JWT](./s_nestjs_12-authentication_jwt) → RBAC 与权限映射 → 资源归属、租户边界 → ABAC / ReBAC → 策略治理。Guard 与 Pipe 的执行顺序可回看[中间件与守卫](./s_nestjs_5-middleware_guard)。

> 来源：[NestJS Authorization](https://docs.nestjs.com/security/authorization)、[NIST RBAC](https://csrc.nist.gov/projects/role-based-access-control)、[NIST ABAC](https://csrc.nist.gov/projects/attribute-based-access-control)、[OWASP Authorization Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authorization_Cheat_Sheet.html)
>
> 来源：[OpenFGA Authorization Concepts](https://openfga.dev/docs/authorization-concepts)、[PostgreSQL Row Security Policies](https://www.postgresql.org/docs/current/ddl-rowsecurity.html)
