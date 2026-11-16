# 密码哈希、加盐与存储

> 登录密码通常只需验证，不需要恢复原文。准确说法是**加盐的密码哈希或密码派生**，不是可解密的“密码加密”。本文讨论应用用户密码，不修改本站认证实现或数据库。

## 一、编码、加密、哈希分别是什么？

| 机制与用途 | 能否还原 | 用于登录密码存储 |
| --- | --- | --- |
| Base64 编码：格式转换 | 无需秘密即可还原 | 不适合 |
| AES 加密：保存需恢复的数据 | 有密钥就能解密 | 通常无需保留解密能力 |
| SHA-256 通用哈希：完整性等 | 不能解密，但可猜测验证 | 单独使用过快，不适合 |
| Argon2id、scrypt：密码哈希 / KDF | 可验证猜测，增加成本 | 适合，需合理配置 |

“哈希不可逆”不代表弱密码无法猜中。加密保存密码又保留解密能力时，密钥泄露可能暴露大量原密码。

## 二、盐与成本参数

```text
存储记录 ≈ 算法版本 + 成本参数 + 随机盐 + 派生结果
```

盐使用密码学安全随机数，通常由成熟库自动生成，与验证值一起保存。每次创建或重设密码时生成新盐，让相同密码也有不同记录。

成本参数控制内存、迭代等资源消耗，提高每次猜测成本，也增加正常登录负载。需要在实际部署环境下结合峰值并发与延迟压测，而不是盲目越大越好。

盐降低预计算复用，专用算法提高猜测成本，二者不能互相替代。不要自行设计“MD5 多算几次”或“SHA-256 + 固定盐”。原理见[彩虹表](./s_safe_7-rainbow_tables)。

## 三、算法怎么选？

| 算法 | 常见定位 | 重要边界 |
| --- | --- | --- |
| Argon2id | 新项目通常优先考虑的内存困难方案 | 配置内存、时间成本和并行度，使用维护良好的实现 |
| scrypt | Argon2id 不可用时的成熟选择 | 合理配置 N、r、p 和内存上限 |
| bcrypt | 维护既有系统时常见 | 多数实现存在 72 字节输入限制，不能静默截断 |
| PBKDF2 | 部分合规或平台约束下使用 | 明确 PRF、迭代次数，随建议与硬件能力调整 |

OWASP 提供多组参考参数，例如 Argon2id 的一个最低参考组合是内存 19 MiB、迭代 2、并行度 1。建议会随时间演进，应查阅原文并压测，不把参考下限当成所有系统的最优值。

异步计算不会让成本消失。Node.js 的某些实现使用 libuv 线程池或工作线程，并发过多时仍争用 CPU、内存和线程；认证入口应有限流、并发边界和资源监控。

## 四、注册、登录与升级流程

### 注册 / 重设

```text
HTTPS 提交 → 校验输入、权限和限流 → 生成随机盐和验证值
         → 保存算法、成本、盐、结果 → 返回不含密码验证数据的响应
```

设置长度边界时考虑算法、体验和资源消耗，不能静默截断、擅自 trim 或改变大小写。重置还需检查凭证有效期、一次性使用，并按策略撤销旧会话。

### 登录验证

```text
定位用户 → 读取原记录的算法、参数和盐 → 处理本次密码并验证
       → 检查账号状态 → 创建会话或签发令牌
```

验证时不能生成新盐后直接比较旧哈希。优先用库的 `verify()` 按存储格式读取参数并校验。很多库会把算法、成本、盐、结果编码为一个字符串，数据库不一定需要拆开字段，但必须预留足够长度，避免截断。

### 升级旧记录

登录成功后，依据版本或 `needsRehash` 用用户刚输入的原密码生成新记录；不能直接再哈希一次旧值，就视为标准算法迁移。并发升级应避免覆盖刚发生的密码重置，例如使用旧验证值或凭证版本做条件更新。

拿不到原密码的长期未登录用户，可以走适当的重置流程。Pepper 的管理和轮换也需要单独设计，它不是可公开的盐。

## 五、Node.js scrypt 学习示例

以下使用内置异步 scrypt，不新增依赖。参数 `N=2^15, r=8, p=3` 是 OWASP 列出的一组参考，每次需数十 MiB 计算内存，尚不代表真实服务的容量结论。

保存为 `password-demo.mjs`，执行 `node password-demo.mjs`。记录格式为教学自定义；生产项目优先采用成熟库和稳定格式，补齐版本迁移与持久化边界。

```js
import { randomBytes, scrypt, timingSafeEqual } from 'node:crypto'
import { promisify } from 'node:util'
import assert from 'node:assert/strict'

const derive = promisify(scrypt)
const options = { N: 32768, r: 8, p: 3, maxmem: 64 * 1024 * 1024 }
const keyLength = 32

function checkPasswordInput(password) {
  // 仅为本例资源边界，不代表推荐的产品密码策略。
  if (typeof password !== 'string' ||
      Buffer.byteLength(password, 'utf8') > 1024) {
    throw new Error('Invalid password input')
  }
}

async function hashPassword(password) {
  checkPasswordInput(password)
  const salt = randomBytes(16)
  const key = await derive(password, salt, keyLength, options)
  return {
    algorithm: 'scrypt', version: 1,
    N: options.N, r: options.r, p: options.p,
    salt: salt.toString('hex'), key: key.toString('hex'),
  }
}

async function verifyPassword(password, record) {
  checkPasswordInput(password)
  // 只接受支持的固定配置，防止任意成本参数耗尽资源。
  if (!record || record.algorithm !== 'scrypt' || record.version !== 1 ||
      record.N !== options.N || record.r !== options.r || record.p !== options.p ||
      typeof record.salt !== 'string' || !/^[0-9a-f]{32}$/.test(record.salt) ||
      typeof record.key !== 'string' || !/^[0-9a-f]{64}$/.test(record.key)) {
    throw new Error('Unsupported or malformed password record')
  }
  const expected = Buffer.from(record.key, 'hex')
  const actual = await derive(
    password, Buffer.from(record.salt, 'hex'), keyLength, options,
  )
  // 已检查格式，两个 Buffer 长度相同。
  return timingSafeEqual(actual, expected)
}

const password = '仅用于本地学习的密码示例'
const first = await hashPassword(password)
const second = await hashPassword(password)
assert.notEqual(first.salt, second.salt)
assert.notEqual(first.key, second.key)
assert.equal(await verifyPassword(password, first), true)
assert.equal(await verifyPassword('错误的密码', first), false)
console.log('随机盐、相同密码不同记录、正确及错误密码校验通过')
```

`timingSafeEqual()` 只帮助安全比较等长字节序列，不自动让整个登录接口成为恒定时间。账号查询、错误分支等仍可能产生差异；不存在账号的验证路径、统一错误响应需另行设计。

算法、成本和存储格式不能无边界信任。多版本验证器需要限制可接受的算法、格式和计算成本；示例直接拒绝旧配置，实际系统要显式管理兼容版本。

## 六、NestJS 与数据库如何分工？

- Controller / DTO：检查格式与长度，避免密码进入日志。
- 密码 Provider / 认证 Service：封装 hash、verify、参数升级，避免业务各处散落算法。
- Repository / ORM：保存验证数据，使用[参数化查询](./s_safe_6-sql_injection)。
- 响应映射：明确字段允许名单，不返回哈希、盐和重置凭证；哈希不是普通公开业务字段。
- 会话系统：验证成功后创建会话或[签发 JWT](../18nestjs/s_nestjs_12-authentication_jwt)。JWT 签名密钥与密码盐是不同概念。

## 七、常见误区

| 说法 | 正确认识 |
| --- | --- |
| 加盐就是把盐藏起来 | 盐可与哈希一起存，Pepper 才是另管的秘密 |
| HTTPS 加密了就不用哈希 | TLS 保护传输，密码哈希降低存储泄露后的利用速度 |
| 前端哈希后服务端不用再处理 | 可重复使用的客户端哈希可能本身成为凭证，不替代 TLS 与服务端安全存储 |
| 慢哈希不怕破解 | 只是提高成本，弱密码与账号保护仍要关注 |
| 限流能防离线破解 | 它限制在线接口，不限制攻击者自己的计算 |
| 超长密码截断即可 | 可能让不同密码产生相同有效输入，应明确边界或改用合适方案 |

## 八、一分钟复述

> 登录密码通常不需要还原，应使用带独立随机盐和成本参数的密码哈希。盐可以公开，主要降低彩虹表和批量预计算复用；Argon2id、scrypt 等算法提高每次猜测的时间与内存成本。登录时按原记录的参数和盐验证，参数过旧可以在成功登录后升级。还要结合 HTTPS、限流、日志过滤和会话管理，加盐不能解决全部安全问题。

> 来源：[OWASP Password Storage](https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html)、[OWASP Authentication](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)、[Node.js Crypto](https://nodejs.org/docs/latest-v24.x/api/crypto.html)、[RFC 9106：Argon2](https://www.rfc-editor.org/rfc/rfc9106)
