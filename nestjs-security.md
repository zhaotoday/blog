# NestJS 安全知识文档

> 一份系统梳理 NestJS 应用安全的实战文档，涵盖常见 Web 安全威胁、防护手段，以及安全相关模块/包的使用方式。

## 目录

1. [安全总览](#1-安全总览)
2. [Helmet：HTTP 安全响应头](#2-helmethttp-安全响应头)
3. [CORS：跨域资源共享](#3-cors跨域资源共享)
4. [CSRF：跨站请求伪造防护](#4-csrf跨站请求伪造防护)
5. [Rate Limiting：限流防护](#5-rate-limiting限流防护)
6. [认证 Authentication](#6-认证-authentication)
7. [授权 Authorization](#7-授权-authorization)
8. [密码与数据加密](#8-密码与数据加密)
9. [输入校验与数据清洗](#9-输入校验与数据清洗)
10. [配置与密钥管理](#10-配置与密钥管理)
11. [数据库与注入防护](#11-数据库与注入防护)
12. [日志、监控与错误处理](#12-日志监控与错误处理)
13. [依赖与供应链安全](#13-依赖与供应链安全)
14. [常用安全包速查表](#14-常用安全包速查表)
15. [生产环境安全检查清单](#15-生产环境安全检查清单)

---

## 1. 安全总览

NestJS 官方安全文档主要涵盖：**Authentication（认证）**、**Authorization（授权）**、**Helmet**、**CORS**、**CSRF Protection**、**Rate Limiting**。除此之外，一个安全的 NestJS 应用还需要关注输入校验、加密、密钥管理、注入防护、依赖安全等方面。

安全防护应遵循的核心原则：

- **纵深防御（Defense in Depth）**：多层防护，不依赖单一手段。
- **最小权限（Least Privilege）**：用户、服务、数据库账号只授予必要权限。
- **默认拒绝（Secure by Default）**：白名单优于黑名单。
- **不信任任何输入（Never Trust Input）**：所有外部输入都需校验和清洗。
- **敏感信息不落地**：密钥、密码不写入代码库，日志中脱敏。

对应到 OWASP Top 10 的主要防护点：

| OWASP 风险 | NestJS 对应防护 |
| --- | --- |
| 失效的访问控制 | Guards + RBAC/CASL 授权 |
| 加密机制失效 | bcrypt/argon2、HTTPS、密钥管理 |
| 注入 | ORM 参数化查询、class-validator 校验 |
| 不安全设计 | 限流、CSRF、纵深防御 |
| 安全配置错误 | Helmet、CORS、关闭调试信息 |
| 易受攻击的组件 | npm audit、依赖升级 |
| 认证失败 | Passport/JWT、强密码策略、限流 |
| 数据完整性失效 | 签名校验、依赖锁定 |
| 日志监控不足 | 结构化日志、审计、告警 |
| SSRF | 出站请求校验、白名单 |

---

## 2. Helmet：HTTP 安全响应头

Helmet 通过设置一系列 HTTP 响应头（如 `Content-Security-Policy`、`X-Frame-Options`、`Strict-Transport-Security` 等）来防范常见攻击（点击劫持、XSS、MIME 嗅探等）。

**安装：**

```bash
npm i helmet
```

**Express 平台（默认）：**

```ts
// main.ts
import helmet from 'helmet';
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.use(helmet());
  await app.listen(3000);
}
bootstrap();
```

> 注意：`helmet` 必须在其他路由/中间件注册**之前**调用。

**Fastify 平台：**

```bash
npm i @fastify/helmet
```

```ts
import fastifyHelmet from '@fastify/helmet';
// app 为 NestFastifyApplication
await app.register(fastifyHelmet);
```

**自定义 CSP（内容安全策略）示例：**

```ts
app.use(
  helmet({
    contentSecurityPolicy: {
      directives: {
        defaultSrc: [`'self'`],
        scriptSrc: [`'self'`, `'unsafe-inline'`],
        imgSrc: [`'self'`, 'data:', 'https:'],
      },
    },
  }),
);
```

---

## 3. CORS：跨域资源共享

CORS 控制哪些外部源（Origin）可以访问你的 API。NestJS 内置支持。

**快速开启：**

```ts
const app = await NestFactory.create(AppModule);
app.enableCors();
```

**生产环境应使用白名单，避免 `origin: '*'`：**

```ts
app.enableCors({
  origin: ['https://app.example.com', 'https://admin.example.com'],
  methods: ['GET', 'POST', 'PUT', 'DELETE', 'PATCH'],
  credentials: true, // 允许携带 Cookie
  allowedHeaders: ['Content-Type', 'Authorization'],
});
```

**动态校验 Origin：**

```ts
app.enableCors({
  origin: (origin, callback) => {
    const whitelist = ['https://app.example.com'];
    if (!origin || whitelist.includes(origin)) {
      callback(null, true);
    } else {
      callback(new Error('Not allowed by CORS'));
    }
  },
  credentials: true,
});
```

> 安全提醒：当 `credentials: true` 时，**不能**同时使用 `origin: '*'`，否则浏览器会拒绝。

---

## 4. CSRF：跨站请求伪造防护

CSRF 攻击利用用户已登录的身份（Cookie）在其不知情的情况下发起请求。**若你的应用使用 Cookie/Session 认证，则需要 CSRF 防护；若使用纯 Token（Authorization 头）认证，则通常无需 CSRF。**

**安装（官方推荐 csrf-csrf）：**

```bash
npm i csrf-csrf
```

```ts
import { doubleCsrf } from 'csrf-csrf';

const { doubleCsrfProtection, generateToken } = doubleCsrf({
  getSecret: () => process.env.CSRF_SECRET,
  cookieName: '__Host-psifi.x-csrf-token',
  cookieOptions: { sameSite: 'lax', secure: true, httpOnly: true },
});

app.use(doubleCsrfProtection);
```

**旧方案 `csurf` 已废弃，不推荐使用。**

**补充防护：** 使用 Cookie 的 `SameSite=Strict` 或 `Lax` 属性可有效降低 CSRF 风险。

---

## 5. Rate Limiting：限流防护

限流用于防止暴力破解、DDoS、接口滥用。官方包为 `@nestjs/throttler`。

**安装：**

```bash
npm i @nestjs/throttler
```

**全局配置：**

```ts
// app.module.ts
import { ThrottlerModule, ThrottlerGuard } from '@nestjs/throttler';
import { APP_GUARD } from '@nestjs/core';

@Module({
  imports: [
    ThrottlerModule.forRoot([
      {
        name: 'default',
        ttl: 60000, // 时间窗口，单位毫秒（60 秒）
        limit: 10,  // 窗口内最多 10 次请求
      },
    ]),
  ],
  providers: [
    { provide: APP_GUARD, useClass: ThrottlerGuard },
  ],
})
export class AppModule {}
```

**针对特定路由自定义/跳过：**

```ts
import { Throttle, SkipThrottle } from '@nestjs/throttler';

@Controller('auth')
export class AuthController {
  // 登录接口更严格：10 秒内最多 3 次
  @Throttle({ default: { limit: 3, ttl: 10000 } })
  @Post('login')
  login() {}

  // 跳过限流
  @SkipThrottle()
  @Get('health')
  health() {}
}
```

**分布式场景使用 Redis 存储（多实例共享计数）：**

```bash
npm i @nest-lab/throttler-storage-redis ioredis
```

```ts
import { ThrottlerStorageRedisService } from '@nest-lab/throttler-storage-redis';

ThrottlerModule.forRoot({
  throttlers: [{ ttl: 60000, limit: 10 }],
  storage: new ThrottlerStorageRedisService(process.env.REDIS_URL),
});
```

---

## 6. 认证 Authentication

认证解决"你是谁"的问题。NestJS 官方推荐使用 **Passport** 或直接实现 **JWT**。

### 6.1 相关包

```bash
# JWT 方案
npm i @nestjs/jwt

# Passport 方案
npm i @nestjs/passport passport
npm i passport-jwt passport-local
npm i -D @types/passport-jwt @types/passport-local
```

### 6.2 JWT 模块配置

```ts
// auth.module.ts
import { JwtModule } from '@nestjs/jwt';

@Module({
  imports: [
    JwtModule.registerAsync({
      useFactory: () => ({
        secret: process.env.JWT_SECRET,
        signOptions: { expiresIn: '15m' }, // 建议 access token 短期有效
      }),
    }),
  ],
})
export class AuthModule {}
```

### 6.3 签发与校验

```ts
// auth.service.ts
@Injectable()
export class AuthService {
  constructor(private jwtService: JwtService) {}

  async login(user: User) {
    const payload = { sub: user.id, username: user.username };
    return {
      access_token: await this.jwtService.signAsync(payload),
    };
  }
}
```

### 6.4 JWT 认证守卫（AuthGuard）

```ts
// jwt-auth.guard.ts
import { CanActivate, ExecutionContext, Injectable, UnauthorizedException } from '@nestjs/common';
import { JwtService } from '@nestjs/jwt';

@Injectable()
export class JwtAuthGuard implements CanActivate {
  constructor(private jwtService: JwtService) {}

  async canActivate(context: ExecutionContext): Promise<boolean> {
    const request = context.switchToHttp().getRequest();
    const token = request.headers.authorization?.split(' ')[1];
    if (!token) throw new UnauthorizedException();
    try {
      request.user = await this.jwtService.verifyAsync(token, {
        secret: process.env.JWT_SECRET,
      });
    } catch {
      throw new UnauthorizedException();
    }
    return true;
  }
}
```

### 6.5 认证安全最佳实践

- **Access Token 短期有效（如 15 分钟）**，配合 **Refresh Token** 机制续期。
- Refresh Token 应存储在服务端（可撤销），或使用 `httpOnly` + `secure` Cookie。
- 密码错误、账号锁定等场景配合限流防暴力破解。
- 支持多因素认证（MFA）以提升关键操作安全性。
- 登出时应使 Token 失效（黑名单 / 短有效期 + 刷新）。

---

## 7. 授权 Authorization

授权解决"你能做什么"的问题。常见模型：**RBAC（基于角色）** 和 **ABAC/权限（基于属性/能力）**。

### 7.1 基于角色的访问控制（RBAC）

**自定义 Roles 装饰器：**

```ts
// roles.decorator.ts
import { SetMetadata } from '@nestjs/common';
export const ROLES_KEY = 'roles';
export const Roles = (...roles: string[]) => SetMetadata(ROLES_KEY, roles);
```

**Roles 守卫：**

```ts
// roles.guard.ts
import { CanActivate, ExecutionContext, Injectable } from '@nestjs/common';
import { Reflector } from '@nestjs/core';
import { ROLES_KEY } from './roles.decorator';

@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const requiredRoles = this.reflector.getAllAndOverride<string[]>(ROLES_KEY, [
      context.getHandler(),
      context.getClass(),
    ]);
    if (!requiredRoles) return true;
    const { user } = context.switchToHttp().getRequest();
    return requiredRoles.some((role) => user.roles?.includes(role));
  }
}
```

**使用：**

```ts
@Roles('admin')
@UseGuards(JwtAuthGuard, RolesGuard)
@Delete(':id')
remove(@Param('id') id: string) {}
```

### 7.2 基于能力的授权（CASL）

对于更细粒度的权限（如"只能编辑自己的文章"），官方推荐 **CASL**。

```bash
npm i @casl/ability
```

```ts
// casl-ability.factory.ts
import { AbilityBuilder, createMongoAbility } from '@casl/ability';

export class CaslAbilityFactory {
  createForUser(user: User) {
    const { can, cannot, build } = new AbilityBuilder(createMongoAbility);
    if (user.isAdmin) {
      can('manage', 'all'); // 全部权限
    } else {
      can('read', 'Article');
      can('update', 'Article', { authorId: user.id }); // 只能改自己的
    }
    return build();
  }
}
```

---

## 8. 密码与数据加密

**永远不要明文存储密码。** 使用单向哈希算法（bcrypt / argon2），敏感数据传输使用 HTTPS。

### 8.1 bcrypt

```bash
npm i bcrypt
npm i -D @types/bcrypt
```

```ts
import * as bcrypt from 'bcrypt';

// 加密（saltRounds 建议 10~12）
const hash = await bcrypt.hash(plainPassword, 12);

// 校验
const isMatch = await bcrypt.compare(plainPassword, hash);
```

### 8.2 argon2（更现代，抗 GPU 破解，推荐）

```bash
npm i argon2
```

```ts
import * as argon2 from 'argon2';

const hash = await argon2.hash(plainPassword);
const isValid = await argon2.verify(hash, plainPassword);
```

### 8.3 对称/非对称加密

Node 内置 `crypto` 模块可用于加密敏感字段（如手机号）：

```ts
import { createCipheriv, createDecipheriv, randomBytes } from 'crypto';
// 使用 AES-256-GCM 等算法，密钥从环境变量/KMS 获取
```

### 8.4 传输安全

- 生产环境**强制 HTTPS**，通过反向代理（Nginx）或 `Strict-Transport-Security` 头。
- 使用 `secure` + `httpOnly` + `sameSite` 的 Cookie 属性。

---

## 9. 输入校验与数据清洗

所有外部输入都必须校验，防止注入、非法数据、XSS。NestJS 使用 `class-validator` + `class-transformer` 配合 `ValidationPipe`。

**安装：**

```bash
npm i class-validator class-transformer
```

**全局启用 ValidationPipe：**

```ts
// main.ts
import { ValidationPipe } from '@nestjs/common';

app.useGlobalPipes(
  new ValidationPipe({
    whitelist: true,            // 自动剔除 DTO 中未声明的属性
    forbidNonWhitelisted: true, // 出现未声明属性时直接报错
    transform: true,            // 自动类型转换
    transformOptions: { enableImplicitConversion: true },
  }),
);
```

**DTO 校验示例：**

```ts
import { IsEmail, IsString, MinLength, MaxLength, Matches } from 'class-validator';

export class CreateUserDto {
  @IsEmail()
  email: string;

  @IsString()
  @MinLength(8)
  @MaxLength(64)
  @Matches(/^(?=.*[A-Za-z])(?=.*\d).+$/, { message: '密码需包含字母和数字' })
  password: string;
}
```

**要点：**

- `whitelist: true` 是防止**属性注入（Mass Assignment）**的关键。
- 前端展示用户输入时需做输出转义，防止存储型 XSS。
- 富文本内容使用 `sanitize-html` 等库清洗。

---

## 10. 配置与密钥管理

**密钥、密码、Token 绝不能硬编码或提交到代码库。** 使用环境变量 + `@nestjs/config`。

```bash
npm i @nestjs/config
```

```ts
// app.module.ts
import { ConfigModule } from '@nestjs/config';

@Module({
  imports: [
    ConfigModule.forRoot({
      isGlobal: true,
      envFilePath: '.env',
      validationSchema: Joi.object({
        JWT_SECRET: Joi.string().required(),
        DATABASE_URL: Joi.string().required(),
        NODE_ENV: Joi.string().valid('development', 'production').default('development'),
      }),
    }),
  ],
})
export class AppModule {}
```

**要点：**

- `.env` 文件加入 `.gitignore`。
- 使用 `Joi` 或 `class-validator` 校验环境变量，缺失时快速失败。
- 生产环境密钥建议托管于 **Secret Manager / KMS / Vault**，而非明文 `.env`。
- 不同环境（dev/staging/prod）使用独立密钥。

---

## 11. 数据库与注入防护

- **使用 ORM/查询构建器的参数化查询**（TypeORM、Prisma、Sequelize），避免字符串拼接 SQL。
- 若必须写原生 SQL，使用参数占位符：

```ts
// TypeORM —— 正确（参数化）
await repo.query('SELECT * FROM users WHERE email = $1', [email]);

// 错误（SQL 注入风险）
await repo.query(`SELECT * FROM users WHERE email = '${email}'`);
```

- 数据库账号遵循最小权限原则（应用账号不使用超级用户）。
- NoSQL（MongoDB）注意 **NoSQL 注入**，对 `$` 开头的操作符输入做校验。
- 敏感字段（密码哈希、密钥）在查询时用 `select: false` 默认排除。

---

## 12. 日志、监控与错误处理

- **不要向客户端泄露堆栈信息**：生产环境关闭详细错误。使用全局异常过滤器统一返回结构化错误。

```ts
// 生产环境避免返回内部错误细节
@Catch()
export class AllExceptionsFilter implements ExceptionFilter {
  catch(exception: unknown, host: ArgumentsHost) {
    // 记录完整错误到日志，但只向客户端返回通用信息
  }
}
```

- **日志脱敏**：密码、Token、身份证号等敏感字段不得写入日志。
- 记录安全审计事件：登录成功/失败、权限变更、敏感操作。
- 集成监控告警（如异常登录、限流触发激增）。
- 推荐日志库：`pino`（`nestjs-pino`）、`winston`。

---

## 13. 依赖与供应链安全

- 定期运行审计：

```bash
npm audit
npm audit fix
```

- 使用 `package-lock.json` 锁定依赖版本，保证可重现构建。
- 借助工具持续监控：**Dependabot**、**Snyk**、**Renovate**。
- 谨慎引入不活跃/低星第三方包，减少攻击面。
- CI 中加入依赖扫描环节。

---

## 14. 常用安全包速查表

| 领域 | 推荐包 | 说明 |
| --- | --- | --- |
| HTTP 安全头 | `helmet` / `@fastify/helmet` | 设置安全响应头 |
| CORS | 内置 `app.enableCors()` | 跨域控制 |
| CSRF | `csrf-csrf` | CSRF 防护（`csurf` 已废弃） |
| 限流 | `@nestjs/throttler` | 请求限流 |
| Redis 限流存储 | `@nest-lab/throttler-storage-redis` | 分布式限流 |
| 认证 | `@nestjs/jwt`、`@nestjs/passport`、`passport-jwt`、`passport-local` | JWT / Passport 认证 |
| 授权 | `@casl/ability` | 细粒度权限 |
| 密码哈希 | `argon2`（推荐）/ `bcrypt` | 密码单向加密 |
| 输入校验 | `class-validator`、`class-transformer` | DTO 校验与转换 |
| 配置管理 | `@nestjs/config`、`joi` | 环境变量与校验 |
| 富文本清洗 | `sanitize-html` | 防存储型 XSS |
| 日志 | `nestjs-pino`、`winston` | 结构化日志 |
| 依赖扫描 | `npm audit`、Snyk、Dependabot | 供应链安全 |

---

## 15. 生产环境安全检查清单

- [ ] 启用 `helmet` 设置安全响应头
- [ ] CORS 使用白名单，未使用 `origin: '*'` 搭配 `credentials`
- [ ] 使用 Cookie/Session 时启用 CSRF 防护
- [ ] 全局启用 `@nestjs/throttler` 限流，登录等敏感接口更严格
- [ ] 全局 `ValidationPipe` 开启 `whitelist` + `forbidNonWhitelisted`
- [ ] 密码使用 argon2/bcrypt 哈希存储，无明文
- [ ] JWT 密钥强随机，Access Token 短期有效 + Refresh 机制
- [ ] 所有密钥/密码通过环境变量或 KMS 管理，未提交代码库
- [ ] `.env` 已加入 `.gitignore`，环境变量有校验
- [ ] 数据库使用参数化查询，账号最小权限
- [ ] 生产环境强制 HTTPS，Cookie 设置 `httpOnly`/`secure`/`sameSite`
- [ ] 生产环境关闭详细错误堆栈，日志脱敏
- [ ] 记录安全审计日志并接入告警
- [ ] 定期 `npm audit` 并接入依赖扫描工具
- [ ] 关闭不必要的调试端点与默认账号

---

## 参考资料

- NestJS 官方安全文档：https://docs.nestjs.com/security/authentication
- OWASP Top 10：https://owasp.org/www-project-top-ten/
- OWASP Cheat Sheet Series：https://cheatsheetseries.owasp.org/
- Node.js 安全最佳实践：https://nodejs.org/en/learn/getting-started/security-best-practices
