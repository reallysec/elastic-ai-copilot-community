# Elastic AI Copilot — 企业 SSO 部署

读者:客户运维 / 实施工程师。

默认的 `docker-compose.prod.yml` 用 Caddy HTTP Basic Auth(单一登录口令)。
本文档介绍**企业 SSO** 形态:分析师用各自的企业账号登录,网关知道**每个用户是谁**
(审计可追溯到人)。

---

## 1. 架构(forward-auth)

```
分析师浏览器
   │  HTTPS
   ▼
Caddy(TLS 终结)
   │
   ▼
oauth2-proxy ──OIDC──► 企业 IdP(Entra / Okta / Keycloak …)
   │  注入已验证身份头 X-Auth-Request-* / X-Forwarded-*
   ▼
AI 网关
```

- **oauth2-proxy** 是一个成熟的 OIDC 反向代理。未登录的请求被重定向到 IdP;
  登录成功后,它把已验证身份作为请求头注入,转发给网关。
- **网关不自己实现登录**——它只信任 oauth2-proxy 注入的身份头。信任成立的前提:
  ① oauth2-proxy 会用会话里的已验证身份**覆盖**这些头,客户端无法伪造;
  ② 网关只在代理链内部可达(`SharedSecretMiddleware` 拒掉任何不带共享密钥的 `/api/*`)。
- 协议无关:网关只认身份头。换一个会注入身份头的前置代理,就能支持 SAML / LDAP
  (见 §6)。

文件:`docker-compose.sso.yml`、`Caddyfile.sso`、`keycloak/realm-rst.json`。

---

## 2. 快速开始 — 自带 Keycloak 演示

`docker-compose.sso.yml` 内置一个 Keycloak(`--profile bundled-idp`),开箱即可演示:

```bash
# 仓库检出内演示:compose 不再带 build:(交付包是 prebuilt 镜像、无 Dockerfile),
# 所以先自己构建一个 dev tag。
docker build -t rst-elastic-ai-copilot-gateway:dev .
GATEWAY_IMAGE_TAG=dev docker compose -f docker-compose.sso.yml --profile bundled-idp up -d
```

浏览器打开 `http://localhost:18180/` → 自动跳转 Keycloak 登录页。
测试账号:**analyst / analyst123**。登录后回到产品,右上角显示当前用户 + 登出。

> 自带 Keycloak 的 realm(`keycloak/realm-rst.json`)是**起步模板**:client secret、
> `redirectUris: ["*"]`、`sslRequired: none` 都是演示值。投产前务必收紧,或直接
> 改用客户已有 IdP(§3)。

---

## 3. 生产 — 对接客户 IdP

客户已有 IdP(Azure Entra ID / Okta / 自有 Keycloak)时,**不要**用 `--profile
bundled-idp`,改为让 oauth2-proxy 指向客户 IdP。

1. 在客户 IdP 注册一个 OIDC 应用(机密客户端 / confidential client):
   - 回调 URI:`https://<你的站点>/oauth2/callback`
   - 拿到 `client_id` / `client_secret` / `issuer_url`。
2. 在项目根 `.env` 填写:

```bash
OIDC_ISSUER_URL=https://login.microsoftonline.com/<tenant>/v2.0   # IdP 的 issuer
OIDC_CLIENT_ID=<client id>
OIDC_CLIENT_SECRET=<client secret>
OAUTH2_PROXY_SKIP_OIDC_DISCOVERY=false        # 真实 IdP 用标准 discovery
OAUTH2_PROXY_COOKIE_SECRET=<openssl rand -base64 32>
OAUTH2_PROXY_REDIRECT_URL=https://<你的站点>/oauth2/callback
OAUTH2_PROXY_COOKIE_SECURE=true
OAUTH2_PROXY_REVERSE_PROXY=true
CADDY_SITE_ADDRESS=<你的站点>
RST_GATEWAY_SHARED_SECRET=<openssl rand -hex 32>
```

3. 启动(不带 `--profile bundled-idp`,Keycloak 不会启动):

```bash
# 客户主机上用 ./deploy.sh(认证方式选 2 = SSO),它会加载镜像并 pin GATEWAY_IMAGE_TAG。
# 手动等价:先 docker load 交付包里的镜像 tar,把 tag 写进 .env,再:
docker compose -f docker-compose.sso.yml up -d
```

---

## 4. 关键环境变量

| 变量 | 说明 |
|---|---|
| `OIDC_ISSUER_URL` | IdP 的 OIDC issuer |
| `OIDC_CLIENT_ID` / `OIDC_CLIENT_SECRET` | 在 IdP 注册的客户端凭据 |
| `OAUTH2_PROXY_SKIP_OIDC_DISCOVERY` | 真实 IdP 设 `false`(标准发现);自带 Keycloak 演示用 `true`(分离 URL,见 compose 注释) |
| `OAUTH2_PROXY_COOKIE_SECRET` | 会话 cookie 加密密钥,`openssl rand -base64 32` |
| `OAUTH2_PROXY_REDIRECT_URL` | `https://<站点>/oauth2/callback`,须与 IdP 注册一致 |
| `OAUTH2_PROXY_COOKIE_SECURE` | 生产(HTTPS)设 `true` |
| `RST_SSO_ENABLED` | 网关:`true` 时前端显示用户+登出(compose 已设) |
| `RST_SSO_ENFORCE` | 网关:`true` 时所有受控 `/api/*` 必须带已验证身份,否则 401(compose 默认 `true`) |
| `RST_SSO_USER_HEADER` / `RST_SSO_GROUPS_HEADER` | 可选:覆盖网关读取身份的请求头名(默认已覆盖 oauth2-proxy 的 `X-Auth-Request-*` 与 `X-Forwarded-*`) |

---

## 5. 网关如何用身份

- **审计**:每条审计事件的 `user` 字段来自 SSO 身份(可信)——追溯到具体的人。
- **`/api/me`**:前端调它显示当前登录用户与登出链接(`/oauth2/sign_out`)。
- **强制**:`RST_SSO_ENFORCE=true` 时,缺身份的受控 `/api/*` 直接 401 —— 防止
  代理配置漏注入身份头导致"匿名"调用。
- 用户的 IdP **组**经 `groups` claim 进入身份的 `roles`,写进审计。
  > 注:本轮 SSO 只做**身份认证**(用户是谁)。基于组的 **RBAC 权限控制**
  > 是单独一项,尚未实现 —— 组信息已采集、备将来用。

---

## 6. 传统 AD 域(LDAP 用户联邦)

分析师用 **AD/LDAP 域账号**登录:Keycloak 通过**用户联邦**对接目录,登录时做
LDAP 绑定校验,再正常出 OIDC 给 oauth2-proxy —— 网关与 oauth2-proxy 不变。

**自带演示**:`--profile bundled-idp` 内置一个 OpenLDAP(`keycloak/ldap-seed.ldif`
种子),`realm-rst.json` 已配好 LDAP 联邦。演示账号:**analyst.ldap / Ldap123!**
(该账号只在 LDAP 里、Keycloak 中没有,登录成功即证明联邦生效)。

**对接客户真实目录**:改 `realm-rst.json` 里 `components` → `corp-ad-ldap` 的
`config`,或在 Keycloak 管理台 → User Federation 配置。**微软 AD** 的关键差异:

| 字段 | OpenLDAP(演示) | 微软 Active Directory |
|---|---|---|
| `vendor` | `other` | `ad` |
| `connectionUrl` | `ldap://openldap:389` | `ldaps://dc.corp.local:636`(建议 LDAPS) |
| `usernameLDAPAttribute` | `uid` | `sAMAccountName`(或 `userPrincipalName`) |
| `rdnLDAPAttribute` | `uid` | `cn` |
| `uuidLDAPAttribute` | `entryUUID` | `objectGUID` |
| `userObjectClasses` | `inetOrgPerson, organizationalPerson` | `person, organizationalPerson, user` |
| `bindDn` | `cn=admin,dc=rst,dc=local` | 一个 AD 服务账号的 DN |

> 组 → 角色:再加一个 `group-ldap-mapper`,把 AD 安全组映射进 `groups` claim,
> 随 OIDC 进入审计的 `roles`。

---

## 7. 传统 SSO(SAML 2.0 身份代理)

客户已有 **SAML 2.0 IdP**(ADFS / 其他)时,Keycloak 作为 **SAML SP** 做身份代理:
SAML 登录 → Keycloak 校验断言 → 转成 OIDC 给 oauth2-proxy —— 下游链路不变。

**自带演示**:`realm-saml-idp.json` 是一个充当"客户 SAML IdP"的 Keycloak realm,
`realm-rst.json` 的 `identityProviders` 已配好对它的代理。登录页会多出一个
**「Corp SAML 登录」** 按钮。演示账号:**saml.user / Saml123!**。

**对接客户真实 SAML IdP**:Keycloak 管理台 → Identity Providers → 添加 SAML v2.0
→ **从客户 IdP 的元数据 URL/文件导入**(自动填好 SSO URL、实体 ID、签名证书)。
生产务必:`validateSignature: true` + 导入 IdP 签名证书;按客户断言里的属性名配好
`email` / 姓名的 identity-provider 属性映射。

> ADFS 既讲 SAML 也讲 WS-Fed —— 走 SAML 接入即可,配置同上。

---

## 8. 排错

| 现象 | 排查 |
|---|---|
| 登录后仍循环跳转 IdP | `OAUTH2_PROXY_REDIRECT_URL` 与 IdP 注册的回调 URI 不一致;cookie 域/`COOKIE_SECURE` 与实际 scheme 不符 |
| 登录成功但右上角不显示用户 | `/api/me` 看 `authenticated`;若 `false`,代理未注入身份头 —— 查 oauth2-proxy 的 `set-xauthrequest` / `pass-user-headers`,或用 `RST_SSO_USER_HEADER` 指定头名 |
| oauth2-proxy 启动报 OIDC discovery 失败 | `OIDC_ISSUER_URL` 不可达或 issuer 不匹配;自带 Keycloak 注意 `KC_HOSTNAME` 与 issuer 一致 |
| 受控接口 401 "缺少已验证身份" | `RST_SSO_ENFORCE=true` 但请求未经 SSO 代理;确认走的是 oauth2-proxy 入口 |
