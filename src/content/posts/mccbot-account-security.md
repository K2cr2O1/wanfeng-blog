---
title: "mccbot 如何保护你的 Minecraft 账号：OAuth2 免密托管与平台安全机制"
published: 2026-08-06
description: "面向用户的安全说明：mccbot 用 LittleSkin OAuth2 实现免密码托管，平台拿不到你的账号密码，授权随时可单方面撤销。本文讲清 OAuth2 流程、token 自动刷新、双端口隔离、HMAC 签名等保障。"
tags: [Minecraft, MCC, OAuth2, LittleSkin, 账号安全, mccbot]
category: 技术
draft: false
---

> 这是一篇写给**用户**的安全说明，不是开发者文档。如果你正在考虑用 mccbot 托管假人，但担心账号密码泄露，本文回答你的所有疑问。

## 一、你最担心的那个问题

「我想用 mccbot 托管我的假人，但平台会不会看到我的 LittleSkin 密码？」

这是用户问得最多的问题。短答：**不会**。mccbot 不存你的密码，也拿不到你的密码。

长答见下文。

## 二、问题的本质：托管 = 信任

托管假人意味着让一个程序替你登录 Minecraft 服务器，这必然涉及三样东西：

- 你的 Minecraft 账号凭据
- 你的登录状态（session）
- 你的游戏内行为（移动、聊天、挖矿）

传统做法是「密码托管」：你把 LittleSkin 账号密码交给平台，平台替你登录。这有个致命问题——**密码一旦交出去，平台能做任何事**：改密码、删角色、消费余额、关联到其他使用 LittleSkin 登录的服务。

更好的做法是「令牌托管」：你不交密码，只交一个**临时令牌**，令牌只能做有限的事，且随时可以撤销。

mccbot 用的就是后者，具体是 LittleSkin 的 OAuth2 协议。

## 三、OAuth2 是什么：用微信登录的类比

OAuth2 是一种「授权码」协议。如果你用过「用微信登录」「用 GitHub 登录」「用 Google 登录」，那就是 OAuth2。

核心思想：**你不把密码给第三方，只给一个授权码**，第三方拿授权码去微信换 token，token 只能访问你授权的那部分功能。

在 mccbot 场景里：

| OAuth2 角色 | 对应实体 |
| --- | --- |
| 资源服务器 | LittleSkin（你的 Minecraft 账号在 LittleSkin 托管） |
| 授权服务器 | 也是 LittleSkin（OAuth2 provider） |
| 第三方应用 | mccbot（要替你登录 Minecraft 服务器） |
| 资源所有者 | 你 |

你授权 mccbot 拿到的是一个 token，这个 token 只能用来**换取 Minecraft 的登录 token**，不能用来改你的 LittleSkin 密码、不能用来消费你的 LittleSkin 余额、不能用来登录 LittleSkin 网站。

## 四、授权流程：你在浏览器里发生了什么

```
1. 你在 mccbot 网页点「LittleSkin OAuth 授权」
        │
        ▼
2. mccbot 把你跳转到 LittleSkin 的授权页：
   https://littleskin.cn/oauth/authorize?client_id=...&scope=...
        │
        │  ⚠ 关键：这一步是浏览器直接连到 LittleSkin
        │           mccbot 完全不参与
        ▼
3. 你在 LittleSkin 输入账号密码、点「同意授权」
        │
        │  ⚠ mccbot 看不到你的密码
        ▼
4. LittleSkin 把你跳回 mccbot，URL 里带一个 code 参数
        │
        ▼
5. mccbot 拿这个 code 去 LittleSkin 换 access_token
   （服务器到服务器，但只换 token，不换密码）
        │
        ▼
6. mccbot 用 access_token 去 Yggdrasil API 换 mc_access_token
        │
        ▼
7. mccbot 把 mc_access_token 写进 MCC 的 SessionCache.db
   MCC 启动时直接复用，进服成功
```

整个流程里，mccbot 全程**没见过你的 LittleSkin 密码**。LittleSkin 密码只出现在第 3 步，且只在你的浏览器和 LittleSkin 之间。

## 五、平台拿到了什么 / 拿不到什么

### ✅ 拿到了什么

| Token | 用途 | 有效期 |
| --- | --- | --- |
| `access_token` | LittleSkin OAuth 访问令牌，用来换 mc_access_token | 短（约 2 小时） |
| `refresh_token` | 刷新令牌，只能用来换新 access_token | 长（30 天） |
| `mc_access_token` | Minecraft 登录令牌，MCC 用它进服 | 24 小时 |
| `client_token` | 和 mc_access_token 配对的客户端令牌 | 长期 |

### ❌ 拿不到什么

- 你的 LittleSkin 密码
- 改你 LittleSkin 账号信息的能力（OAuth scope 限定）
- 登录 LittleSkin 网站的能力（access_token 不是 session cookie）
- 你在其他使用 LittleSkin 登录的服务的权限（每个 OAuth 应用独立）

### 🔄 你能随时单方面撤销

去 LittleSkin 的「授权管理」页面，把 mccbot 这个应用 revoke 掉，所有 token 立刻失效：

- **不需要通知 mccbot**：下一次 token 刷新就会失败，平台会自动停掉你的假人
- **不需要改密码**：OAuth2 的 revoke 是即时的
- **不留尾巴**：access_token 和 refresh_token 同时失效

这是 OAuth2 相对密码托管最大的优势。密码托管你改了密码，平台还可能用旧的 session 继续登录；OAuth2 你 revoke 了，token 就废了。

## 六、Token 怎么存、怎么刷新

Minecraft accessToken 有效期约 24 小时，过期了 MCC 进服会失败。mccbot 做了**自动刷新**，你授权一次，永久免维护：

```
每次启动 / 重启实例前：
   │
   ▼
refresh_yggdrasil_token()
   │
   ├─ 用 refresh_token 调 /oauth/token 换新 access_token
   │
   ├─ 用新 access_token 调 Yggdrasil /authserver/refresh
   │     换新 mc_access_token
   │
   ├─ 写回 Redis（长期保存）
   │
   └─ 写回 SessionCache.db（MCC 启动时复用）

MCC 启动 → 直接用新 token 进服 → 不需要密码
```

**用户视角**：你授权一次，平台永久帮你续 token。只要你不主动 revoke，假人就能一直挂着。

**安全视角**：refresh_token 只在 mccbot 内部使用，不暴露给任何外部服务。LittleSkin 也不存 refresh_token，它只是签发方，签发完就忘。

## 七、平台其他安全保障

### 1. 双端口隔离

平台分两个端口对外：

- `8800` **内部端口**：所有管理 API（agent、tool、mcp）只监听 `127.0.0.1`，**外网完全打不到**
- `8801` **公开端口**：只暴露前端页面和静态资源，经 nginx 反代

即使攻击者知道你的 QQ，也无法从公网直接调管理 API 改你的假人配置。内部接口在公网直接返回 404，根本进不来。

### 2. HMAC 签名 + 防重放

浏览器和 mcc-service 之间的所有请求都带三层防护：

| 防护 | 防什么 |
| --- | --- |
| HMAC-SHA256 签名 | 防篡改（改一个字节签名就对不上） |
| X-Timestamp 时间戳 | 防重放（5 分钟外的请求直接拒绝） |
| Nonce 一次性随机数 | 防重放（同一 Nonce 只能用一次） |

签名密钥 `panel_secret` 编译进前端 WASM，不在网络传输，也难以从前端反推。

### 3. WebAuthn 免密登录

mccbot 的网页登录本身也是免密的：

```
用户网页输入 QQ
   ↓
提示「请私聊机器人发送：登录」
   ↓
用户私聊 neobot「登录」
   ↓
neobot 生成一次性 login_token（5 分钟有效）
   ↓
neobot 把 login_token 私聊给你
   ↓
你在网页输入 → 换 session_token（7 天有效）
```

- `login_token`：5 分钟过期，用一次就删
- `session_token`：7 天有效，每次校验自动续期
- **平台不存密码**：全程没有密码字段

### 4. Passkey（指纹 / Face ID）二次加强

第一次登录后，你可以注册 Passkey（WebAuthn）：

- 公钥存 Redis，**私钥永远在你的设备里**
- mccbot 拿不到私钥，无法伪造你的登录
- 之后登录一键指纹 / Face ID，不再走 QQ 私聊流程
- 签名计数器防重放，同一指纹无法在不同设备复用

### 5. RSA 私钥远程拉取

用于 Passkey 签名验证的 RSA 私钥，可以配置**从独立的安全服务器拉取**：

- mcc-service 主机不落盘
- 即使 mcc-service 主机被入侵，攻击者也拿不到私钥
- 无法伪造你的 Passkey 签名

这是「最小信任」设计：即使最坏情况发生（mccbot 主机被攻破），你的账号也不会被冒充登录。

### 6. 危险工具拦截

MCC 暴露的 62 个工具里，有两个是「会导致假人掉线」的：

- `mcc_quit_client`：让 MCC 进程退出
- `mcc_disconnect`：让假人断开服务器

这两个工具在 mccbot 的转发层直接拒绝调用。即使你的 session 被盗，攻击者也无法用工具把你的假人踢下线——他们能做的只是让它移动、挖矿、聊天，**不能让它消失**。

### 7. 权限隔离

- 你的私有实例，**只有你的 QQ** 能控制
- 借用的假人，**只有借用人** 能控制
- QQ 群里的 agent 是**只读**的，不能操控假人（移动、攻击、发消息等写操作必须到浏览器进行）
- 工具调用每次都从 session 取出归属 QQ，再校验实例归属，不一致直接 403

## 八、三种登录方式对比

mccbot 支持三种 Minecraft 账号登录方式，安全级别递增：

| 方式 | 平台是否存密码 | 你能撤销吗 | 推荐度 |
| --- | --- | --- | --- |
| offline（离线） | 不涉及 | 不需要 | 公开测试服用 |
| Yggdrasil 密码托管 | **存进 ini 文件** | 改密码 | 不推荐 |
| **Yggdrasil OAuth2** | **不存** | revoke 即时生效 | **强烈推荐** |

如果你有 LittleSkin 账号，**强烈建议用 OAuth2**。密码托管方式只是为不支持 OAuth2 的旧版账号保留的兼容方案。

## 九、你能做什么：用户侧的安全建议

平台做了很多，但账号安全的第一道防线还是你自己。建议：

1. **优先用 OAuth2，不用密码托管**
   LittleSkin 账号密码尽量不要交给任何第三方，包括 mccbot
2. **定期检查 LittleSkin 授权列表**
   发现不再用的应用及时 revoke，授权列表保持精简
3. **开启 Passkey**
   减少 session_token 被钓鱼的风险，指纹比 token 难偷
4. **不要把 login_token 截图发给别人**
   虽然只有 5 分钟有效，但足够让人登录你的账号
5. **不要在公共电脑登录 mccbot**
   session_token 存浏览器 localStorage，下机前手动退出
6. **借用假人到期前主动归还**
   避免假人停在不该停的地方，也方便别人继续借用
7. **怀疑账号被盗，先 revoke mccbot**
   单方面撤销授权，比改密码快，比删账号安全

## 十、设计回顾：最小信任

mccbot 的安全设计核心是「最小信任」——平台只拿必须拿的，能不拿的都不拿：

| 设计原则 | 落地 |
| --- | --- |
| 不存密码 | OAuth2 + token 自动刷新 |
| 不暴露接口 | 双端口隔离 + HMAC 签名 |
| 不长期持有会话 | login_token 5 分钟，session 7 天 |
| 不给危险权限 | 工具拦截 + 权限隔离 |
| 不留单点 | RSA 私钥远程拉取 |
| 不依赖平台自觉 | 你可单方面 revoke OAuth2 |

你授权 mccbot 的，只是「替你的假人登录 Minecraft 服务器」这一件事。其他一切，平台都拿不到。

如果未来你不再使用 mccbot，只需要做两件事：

1. 去 LittleSkin 把 mccbot 的 OAuth2 授权 revoke
2. 在 mccbot 网页删除你的私有实例

完成这两步，mccbot 手里关于你的所有可用凭据全部失效，相当于「分手快乐」。

---

附录：技术实现细节见 `oauth.py` / `microsoft_auth.py` / `passkey.py` / `crypto.py` / `auth.py`。
