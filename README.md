
# 企业统一认证流程文档

## 一、整体架构

### 1.1 涉及组件

| 组件          | 说明                            |
| ----------- | ----------------------------- |
| Windows 客户端 | 用户工作环境，已登录 Windows 域账号        |
| 浏览器         | 用户访问企业应用的入口                   |
| 网站A         | 企业前端应用，包含 IMSUtil.js 认证模块     |
| GFAuth 服务   | 企业统一认证服务（J2EE 应用，部署于 Payara）  |
| 微软 OIDC 服务  | 外部身份提供方（IdP），负责微软账号认证         |
| 后端业务服务      | 企业后端 API，通过 Filter + 注解进行权限控制 |

### 1.2 核心数据结构

|结构|说明|
|---|---|
|UserCache|用户信息缓存，存储已认证用户|
|oidcRequestMap|Map<state, OIDCRequest>，state 与认证请求的映射|
|OIDCRequest|包含 redirectUrl（指向微软登录）和 state（随机数）|
|OIDCTokens|包含 accessToken 和 idToken（JWT）|

---

## 二、详细认证流程

### 阶段1：初始登录触发

**步骤 1：** 用户在 Windows 上登录 Windows 域账号（通常开机时完成）

**步骤 2：** 用户通过浏览器访问网站A

**步骤 3：** 网站A 中的 IMSUtil.js 检测到用户未认证，触发登录认证逻辑

**步骤 4：** IMSUtil.js 调用 GFAuth 服务的 `/oidc/json` 接口

- 请求参数：`type=REACT` / `type=JSP` / `type=GWT`
    
- 此请求携带 Windows 身份信息（通过 SPNEGO 协议）
    

---

### 阶段2：GFAuth 初始认证处理

**步骤 5：** GFAuth 接收请求，通过 SPNEGO/Kerberos/NTLM 协议协商，从 SecurityContext 中获取当前用户ID（如：e1000）

**步骤 6：** GFAuth 检查是否可跳过 MFA 认证：

text

跳过条件（满足任一即可）：
├── 请求 Header 中包含 IMSAUTOMATION 标记
└── 请求来源在 MFA domain 之外（内网可信环境）

**情况 A：跳过 MFA**

1. 将用户信息直接放入 UserCache
    
2. 认证通过，直接返回用户信息和 Token 给前端
    
3. 跳到 **阶段7**
    

**情况 B：需要 MFA（继续以下步骤）**

**步骤 7：** GFAuth 基于用户ID获取用户详细信息，写入 UserCache

**步骤 8：** GFAuth 构建 OIDCRequest 对象：

json

{
  "redirectUrl": "https://login.microsoftonline.com/...",
  "state": "随机生成的唯一标识"
}

**步骤 9：** 将 `state → OIDCRequest` 键值对存入 oidcRequestMap

**步骤 10：** GFAuth 将 OIDCRequest 对象返回给前端

---

### 阶段3：微软 OIDC 认证

**步骤 11：** 前端 IMSUtil.js 接收到 OIDCRequest，控制浏览器跳转到 redirectUrl（微软登录页面）

**步骤 12：** 微软检查当前浏览器会话状态：

**情况 A：用户已在当前浏览器登录微软账号（静默流程）**

- 微软检测到有效的登录会话（Cookie/Session）
    
- 跳过账号密码输入界面
    
- 立即自动重定向回 GFAuth 的 callback 接口
    
- 携带授权 code + state 参数
    
- **用户全程无感知，浏览器地址栏可能短暂跳转但用户无法察觉**
    

**情况 B：用户未登录微软账号（手动流程）**

- 显示微软登录页面
    
- 用户输入微软账号和密码（可能包含短信/验证器 MFA 验证）
    
- 登录成功后自动重定向到 GFAuth callback 接口
    
- 携带授权 code + state 参数
    

**步骤 13：** 浏览器重定向到 GFAuth 的 `/oidc/callback` 接口

- 携带参数：`code`（授权码）+ `state`（状态标识）
    

---

### 阶段4：OIDC 回调处理

**步骤 14：** GFAuth callback 接口接收请求，执行以下操作：

1. **验证 state**：检查 state 是否存在于 oidcRequestMap 中
    
    - 存在：继续处理
        
    - 不存在：拒绝请求（防止 CSRF 攻击）
        
2. **构建 TokenRequest**：
    
    - 使用 code、配置的 clientId、tokenEndpoint、callbackUrl 构建请求
        
    - 向微软 Token Endpoint 发起调用，验证 code 并换取 Token
        
3. **获取 OIDCTokens**：
    
    - 成功获取 accessToken 和 idToken
        
    - 从 idToken（JWT 格式）中解析出 email 字段
        

---

### 阶段5：按类型返回响应

**步骤 15：** GFAuth 根据最初登录请求的 type 参数，执行不同返回策略：

**REACT / GWT 类型：**

- 将 state 和原 request 对象重新放入 oidcRequestMap（供后续使用）
    
- 返回 HTTP 303 重定向
    
- 重定向 URL 为最初 OIDCRequest 中的 redirectUrl
    
- URL 格式：`http://网站A?state=xxxxx`
    

**JSP 类型：**

- 将 authUser、authKey、authToken 附加到重定向 URL 参数中
    
- 返回 HTTP 303 重定向到网站A
    

---

### 阶段6：前端二次认证确认

**步骤 16：** 浏览器重定向回网站A，IMSUtil.js 检测到 URL 中携带 state 参数

**步骤 17：** IMSUtil.js 再次调用 GFAuth `/oidc/json` 接口

- 请求携带 state 参数
    

**步骤 18：** GFAuth 处理：

- 检查 state 在 oidcRequestMap 中是否有对应的 request
    
- 有：确认 MFA 认证已完成，登录成功
    
- 返回完整的用户信息和 Token 给前端
    

**步骤 19：** 前端完成认证，用户成功登录网站A

---

### 阶段7：后续 API 访问与权限控制

**步骤 20：** 前端访问后端业务接口时，在请求 Header 中携带：

text

Authorization: Token xxxxxxxx

**步骤 21：** 后端服务的 Filter 拦截器拦截请求：

- 检查目标接口是否有 `@IMSAuth` 注解
    
- 无注解：直接放行（公开接口）
    
- 有注解：继续权限验证流程
    

**步骤 22：** 拦截器解析 `@IMSAuth` 注解参数：

java

@IMSAuth(app = "xxx", level = 1)

- `app`：应用标识
    
- `level`：所需权限等级
    

**步骤 23：** 拦截器携带 Token 调用 GFAuth 的 `getUserByToken` 接口

**步骤 24：** GFAuth 验证 Token 有效性，返回用户的权限信息（包含各 app 的权限 level）

**步骤 25：** 拦截器进行权限比对：

- 用户权限 level ≥ 接口所需 level：放行请求
    
- 用户权限不足：返回 HTTP 401 Unauthorized
    

---

## 三、完整流程图

text

┌─────────────────────────────────────────────────────────────────────┐
│                         用户认证全流程                                │
└─────────────────────────────────────────────────────────────────────┘
 用户         浏览器/前端           GFAuth              微软OIDC         后端服务
  │               │                   │                    │               │
  │ ①Windows登录  │                   │                    │               │
  ├──────────────>│                   │                    │               │
  │               │ ②访问网站A        │                    │               │
  │               ├──────────────────>│                    │               │
  │               │ ③/oidc/json       │                    │               │
  │               ├──────────────────>│                    │               │
  │               │                   │ ④SPNEGO获取用户ID  │               │
  │               │                   │ ⑤检查MFA跳过条件   │               │
  │               │                   │                    │               │
  │               │ ⑥返回OIDCRequest  │                    │               │
  │               │<──────────────────┤                    │               │
  │               │                   │                    │               │
  │               │ ⑦跳转redirectUrl  │                    │               │
  │               ├───────────────────────────────────────>│               │
  │               │                   │                    │               │
  │               │ ⑧已登录→静默跳过  │                    │               │
  │               │   未登录→手动认证 │                    │               │
  │               │                   │                    │               │
  │               │ ⑨重定向callback   │                    │               │
  │               ├──────────────────>│                    │               │
  │               │   (code+state)    │                    │               │
  │               │                   │ ⑩验证state         │               │
  │               │                   │ ⑪换取Token         │               │
  │               │                   │ ⑫获取用户email     │               │
  │               │                   │                    │               │
  │               │ ⑬303重定向       │                    │               │
  │               │<──────────────────┤                    │               │
  │               │                   │                    │               │
  │               │ ⑭/oidc/json+state│                    │               │
  │               ├──────────────────>│                    │               │
  │               │                   │ ⑮验证state确认登录  │               │
  │               │ ⑯返回用户+Token   │                    │               │
  │               │<──────────────────┤                    │               │
  │               │                   │                    │               │
  │               │ ⑰API请求+Token    │                    │               │
  │               ├──────────────────────────────────────────────────────>│
  │               │                   │                    │               │
  │               │                   │ ⑱getUserByToken   │               │
  │               │                   │<──────────────────────────────────┤
  │               │                   │ ⑲返回权限信息      │               │
  │               │                   ├──────────────────────────────────>│
  │               │                   │                    │               │
  │               │ ⑳返回数据/401     │                    │               │
  │               │<──────────────────────────────────────────────────────┤
  │               │                   │                    │               │

---

## 四、用户感知总结

|场景|用户体验|
|---|---|
|Windows 登录 + 已登录微软账号 + MFA domain 外|**完全无感**，打开网站即登录成功|
|Windows 登录 + 已登录微软账号 + 需 MFA|**完全无感**，微软静默验证，后台自动完成|
|Windows 登录 + 未登录微软账号 + MFA domain 外|**无感**，跳过 MFA，直接登录|
|Windows 登录 + 未登录微软账号 + 需 MFA|需要**手动输入一次**微软账号密码|

---

## Appendix：术语与协议介绍

### A. 认证协议

#### SPNEGO (Simple and Protected GSSAPI Negotiation Mechanism)

- **全称**：简单保护 GSSAPI 协商机制
    
- **作用**：允许客户端和服务端协商使用哪种认证协议
    
- **原理**：包装 GSSAPI，在 Kerberos 和 NTLM 之间自动选择
    
- **应用场景**：浏览器访问 Windows 域环境中的 Web 服务时，自动协商认证方式
    

#### Kerberos

- **全称**：Kerberos 网络认证协议（名称源自希腊神话三头犬）
    
- **作用**：基于票据（Ticket）的认证协议，实现安全的身份验证
    
- **核心组件**：
    
    - KDC（Key Distribution Center）：密钥分发中心
        
    - TGT（Ticket Granting Ticket）：票据授予票据
        
    - Service Ticket：服务票据
        
- **特点**：双向认证、防重放、支持单点登录（SSO）
    
- **应用场景**：Windows 域环境默认认证协议
    

#### NTLM (NT LAN Manager)

- **全称**：Windows NT LAN Manager 认证协议
    
- **作用**：Windows 早期的认证协议，Kerberos 的替代方案
    
- **原理**：基于挑战/响应（Challenge/Response）机制
    
- **特点**：较 Kerberos 安全性弱，但在工作组环境仍广泛使用
    
- **应用场景**：非域环境或 Kerberos 不可用时的备用方案
    

#### OIDC (OpenID Connect)

- **全称**：OpenID Connect 开放身份认证协议
    
- **作用**：基于 OAuth 2.0 的身份认证层，允许客户端验证用户身份
    
- **核心概念**：
    
    - ID Token：JWT 格式，包含用户身份信息
        
    - Access Token：访问令牌，用于访问受保护资源
        
    - Authorization Code：授权码，用于换取 Token
        
- **流程**：Authorization Code Flow（授权码流程）
    
- **应用场景**：第三方登录、SSO 单点登录
    

#### OAuth 2.0

- **全称**：开放授权协议 2.0
    
- **作用**：授权第三方应用访问用户资源，无需暴露用户密码
    
- **核心角色**：
    
    - Resource Owner：资源所有者（用户）
        
    - Client：客户端应用
        
    - Authorization Server：授权服务器
        
    - Resource Server：资源服务器
        
- **授权模式**：Authorization Code、Implicit、Client Credentials 等
    

### B. 技术组件

#### JWT (JSON Web Token)

- **全称**：JSON Web Token
    
- **结构**：`Header.Payload.Signature`（三段 Base64 编码）
    
- **Header**：声明签名算法（如 RS256、HS256）
    
- **Payload**：包含 claims（声明），如 email、exp、iss
    
- **Signature**：对 Header + Payload 的签名，防止篡改
    
- **特点**：无状态、自包含、可验证
    

#### Payara Server

- **定位**：开源 Java EE 应用服务器
    
- **前身**：GlassFish 的衍生版本
    
- **支持**：Jakarta EE、MicroProfile
    
- **应用场景**：企业级 Java 应用部署
    

#### SecurityContext

- **定义**：Java EE 安全上下文对象
    
- **作用**：在服务端获取当前认证用户信息
    
- **核心方法**：
    
    - `getUserPrincipal()`：获取用户主体
        
    - `isUserInRole()`：检查用户角色
        
- **应用场景**：服务端获取 SPNEGO 认证后的用户身份
    

### C. 核心概念

#### MFA (Multi-Factor Authentication)

- **全称**：多因素认证
    
- **定义**：使用两种或以上独立认证因素验证用户身份
    
- **认证因素**：
    
    - 知识因素（密码、PIN）
        
    - 拥有因素（手机、硬件 Token）
        
    - 生物因素（指纹、面部识别）
        
- **应用场景**：本流程中微软账号的二次验证
    

#### State 参数

- **作用**：OAuth 2.0 / OIDC 中的 CSRF 防护机制
    
- **原理**：客户端生成随机字符串，认证完成后验证返回值一致性
    
- **流程**：
    
    1. 客户端生成 state，存储
        
    2. 发送认证请求携带 state
        
    3. 回调时验证返回的 state
        
- **安全意义**：防止跨站请求伪造攻击
    

#### Authorization Code

- **定义**：OAuth 2.0 授权码流程中的临时凭证
    
- **特点**：
    
    - 短期有效（通常几分钟）
        
    - 一次性使用
        
    - 需配合 clientId/clientSecret 换取 Token
        
- **安全性**：不暴露在 URL 片段中，通过后端交换
    

#### HTTP 303 See Other

- **定义**：HTTP 重定向状态码
    
- **作用**：告诉客户端使用 GET 方法请求另一个 URL
    
- **与 302 区别**：明确要求使用 GET，避免 POST 重定向问题
    
- **应用场景**：认证完成后的页面跳转
    

#### CSRF (Cross-Site Request Forgery)

- **全称**：跨站请求伪造
    
- **定义**：诱导用户在已登录状态下执行非预期操作
    
- **防护手段**：state 参数、SameSite Cookie、CSRF Token
    
- **本流程防护**：state 参数 + oidcRequestMap 验证
    

#### SSO (Single Sign-On)

- **全称**：单点登录
    
- **定义**：用户一次登录，可访问多个相互信任的应用
    
- **实现方式**：
    
    - Kerberos Ticket
        
    - OIDC Session
        
    - 共享 Cookie
        
- **本流程体现**：Windows 登录 → 自动完成所有认证
    

### D. 架构模式

#### IdP (Identity Provider)

- **全称**：身份提供方
    
- **作用**：负责用户身份验证，颁发身份凭证
    
- **本流程中的 IdP**：GFAuth（企业级）+ 微软 OIDC（云账号）
    

#### SP (Service Provider)

- **全称**：服务提供方
    
- **作用**：依赖 IdP 的认证结果，提供业务服务
    
- **本流程中的 SP**：网站A、后端业务服务
    

#### Filter 拦截器模式

- **定义**：Java Servlet Filter，请求处理前的拦截机制
    
- **作用**：统一处理横切关注点（认证、日志、编码等）
    
- **本流程应用**：后端接口权限验证
    
- **配合注解**：`@IMSAuth(app, level)` 声明式权限控制
    

#### Token-Based Authentication

- **定义**：基于令牌的认证方式
    
- **特点**：
    
    - 无状态（服务端不存 Session）
        
    - 可扩展（分布式环境友好）
        
    - 灵活（可携带权限信息）
        
- **本流程体现**：`Authorization: Token xxxx` 请求头

## Digaram

```
sequenceDiagram
    autonumber
    participant User as 用户
    participant Browser as 浏览器/前端
    participant GFAuth as GFAuth认证服务
    participant MS as 微软OIDC
    participant Backend as 后端业务服务

    Note over User,Backend: 阶段一：初始登录
    User->>Browser: Windows域账号已登录
    Browser->>GFAuth: 调用 /oidc/json 登录接口
    GFAuth->>GFAuth: 获取Windows用户身份
    GFAuth->>GFAuth: 检查是否需要MFA认证
    
    alt 跳过MFA（内网/自动化）
        GFAuth-->>Browser: 直接返回用户信息+Token
        Browser-->>User: 登录成功
    else 需要MFA认证
        GFAuth->>GFAuth: 生成OIDCRequest(state+redirectUrl)
        GFAuth->>GFAuth: 存入oidcRequestMap
        GFAuth-->>Browser: 返回OIDCRequest对象

        Note over User,MS: 阶段二：微软OIDC认证
        Browser->>MS: 跳转redirectUrl
        
        alt 用户已登录微软账号
            MS->>MS: 检测到有效会话
            Note over MS: 静默跳过登录
        else 用户未登录微软账号
            MS-->>User: 显示微软登录页
            User->>MS: 输入微软账号密码
        end
        
        MS-->>Browser: 重定向callback (code+state)

        Note over Browser,Backend: 阶段三：回调验证
        Browser->>GFAuth: 访问 /oidc/callback
        GFAuth->>GFAuth: 验证state有效性
        GFAuth->>MS: 用code换取Token
        MS-->>GFAuth: 返回OIDCTokens
        GFAuth->>GFAuth: 从JWT提取用户email
        GFAuth->>GFAuth: 更新认证状态

        Note over Browser,Backend: 阶段四：前端二次确认
        GFAuth-->>Browser: 303重定向回网站A
        Browser->>GFAuth: 再次调用 /oidc/json (携带state)
        GFAuth->>GFAuth: 验证state存在
        GFAuth-->>Browser: 返回用户信息+Token
        Browser-->>User: 登录成功
    end

    Note over User,Backend: 阶段五：后续API访问
    User->>Browser: 发起业务请求
    Browser->>Backend: API请求 (Authorization: Token xxx)
    Backend->>Backend: 拦截器检查@IMSAuth注解
    Backend->>GFAuth: 调用getUserByToken验证
    GFAuth-->>Backend: 返回用户权限信息
    Backend->>Backend: 权限比对
    
    alt 权限通过
        Backend-->>Browser: 返回业务数据
        Browser-->>User: 显示结果
    else 权限不足
        Backend-->>Browser: 401 Unauthorized
        Browser-->>User: 提示无权限
    end
```

```
sequenceDiagram
    autonumber
    participant User as 用户
    participant Browser as 浏览器/前端
    participant GFAuth as GFAuth认证服务
    participant MS as 微软OIDC
    participant Filter as 后端拦截器
    participant Backend as 后端业务接口

    Note over User,MS: 阶段一：登录认证
    Browser->>GFAuth: 调用登录接口
    GFAuth->>GFAuth: 获取Windows用户身份
    
    alt 跳过MFA（内网/自动化）
        GFAuth-->>Browser: 直接返回Token
    else 需要MFA认证
        GFAuth-->>Browser: 返回OIDCRequest(含redirectUrl+state)
        Browser->>MS: 跳转微软登录页
        
        alt 已登录微软账号
            Note over MS: 静默跳过
        else 未登录微软账号
            User->>MS: 输入微软账号密码
        end
        
        MS-->>Browser: 重定向callback (code+state)
        Browser->>GFAuth: 访问callback
        GFAuth->>MS: 验证code并换取Token
        MS-->>GFAuth: 返回Token
        GFAuth-->>Browser: 重定向回网站A(携带state)
        Browser->>GFAuth: 二次确认(state)
        GFAuth-->>Browser: 返回用户信息+Token
    end
    
    Browser-->>User: 登录成功

    Note over User,Backend: 阶段二：API访问与权限验证
    Browser->>Filter: API请求(携带Token)
    Filter->>Filter: 检查接口@IMSAuth注解
    
    alt 无@IMSAuth注解
        Filter->>Backend: 公开接口直接放行
        Backend-->>Browser: 返回业务数据
    else 有@IMSAuth注解
        Filter->>Filter: 解析app和level参数
        Filter->>GFAuth: 验证Token获取权限
        GFAuth-->>Filter: 返回用户权限信息
        Filter->>Filter: 权限比对
        
        alt 权限通过
            Filter->>Backend: 放行请求
            Backend-->>Browser: 返回业务数据
        else 权限不足
            Filter-->>Browser: 401 Unauthorized
        end
    end
```
