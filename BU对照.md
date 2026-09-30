# BU 对照

同一套卡平台后端按 BU 分品牌。C 端不看浏览器域名决定 BU，看这次构建写进去的 AppId。新管理后台才按访问域名匹配。

核对日期 2026-09-22。库是 DBX 连接 `pro`、`test` 的 `virtronex_business`，只读。代码是目录「只分析」那一列当时的检出：H5 `test` 的 `bu.config.cjs`、`env/`，WebApp `newtest` 的 `package.json` 与 `.env.bu*`，Android `dev` 的 `ChannelRegistry.kt`，卡平台 `dev` 的 `Reseller`、`ResellerBrandController`。iOS `release` 只搜了域名和 `X-App-Id`，没有按 BU 拆包。

AppId 是请求头 `X-App-Id`，下面照录。密钥、PEM、SMTP 不写。

## 三套编号不要混

| 叫什么 | 落在哪 | 谁用它 |
|--------|--------|--------|
| 前端构建名 `bu1`、`bu2`… | H5：`VITE_BU`，配置在 `bu.config.cjs`。Web：`NEXT_PUBLIC_BU`，脚本 `dev:buN` / `build:buN` | 编译期选品牌资源、主题、这份 env |
| BU ID | 表 `reseller.id`。实体注释写的就是 business user | 商户 `reseller_id`、邀请码、邮件、后台数据范围 |
| AppId | 表 `reseller_api.app_id`，挂在某个 `reseller_id` 上 | 验签。是不是平台凭证看 `reseller_api.id` 在不在白名单，不看这一列是不是 0 |

H5 的 `buid` 写在 `bu.config.cjs`，注释说是后端 BU ID，运行时不用它验签，也不按 `mDomain` 切品牌。`getBuRuntimeConfig` 只读构建时的 `VITE_BU` 或 mode（如 `bu2.test`）。`mDomain` 注释写明给文档和 Firebase 授权，不参与匹配。

Web 的 AppId 来自 `OPENAPI_APP_ID`。`package.json` 先加载 `.env.shared`，再加载 `.env.buN`，后者覆盖前者。某个 `.env.buN` 没写 `OPENAPI_APP_ID` 时，用的是 shared 里那一条。

新管理后台登录前：`ResellerBrandController` `/reseller/brand` 用请求 Host 对 `reseller.new_admin_url`（`ResellerBrandSupport`）。对上了才返回这个 BU 的品牌名、logo、favicon。

`reseller` 上每个地址只存一格：`admin_url` 旧管理后台，`new_admin_url` 新管理后台，`web_app_url` Web PC（库里常带 `/auth`）。H5 的 `m.` 域名不在这三格里。

注册时邀请码范围在 `MerchantRegisterInviteCodeSupport`：`scopedResellerId` 为 `null` 或 `0` 全库查；否则只查该 `reseller_id`。平台凭证会把这个值传成 `null`。只把 `reseller_id` 改成 0、主键不在白名单里，邀请码已经是全库，登录仍按 `reseller_id = 0` 过滤。见下一节。注册落库细节见 `02-邮箱注册登录`。

## 平台凭证怎么认

2026-09-23。代码是卡平台 `dev` 的 `OpenApiPlatformScopeSupport`、`VirtronexRuntimeProperties`、`OpenApiAuthService`、`AuthController.login`、`MerchantRegisterInviteCodeSupport`、`ResellerApiBuServiceImpl.create`。库是 DBX 连接 `test` 的 `virtronex_business`，只读，不贴密钥。

两套判断不要并成一句「`reseller_id = 0` 就是平台凭证」。

是不是平台凭证，只看 `reseller_api.id` 在不在 Nacos `reseller.api.platform-ids`。`OpenApiPlatformScopeSupport.isPlatform` 用的就是这个集合。代码默认 `1,2,3,4,5,6,7,8,9,10`，空配置表示没有平台凭证。运行中的值这次没读 Nacos。`VirtronexRuntimeProperties` 注释写明不用 `reseller_id` 做白名单：平台行的 `reseller_id` 都是 0，按这一列分不开各端。

`reseller_id = 0` 是这些行的落库值，表示没有真实 BU。验签 `OpenApiAuthService.verifyAndGetIdentity` 把 `reseller_api.id` 放进 `resellerApiId`，把 `reseller_id` 放进身份上的 `resellerId`。没有 `X-App-Id` 时抛 `api.missing.app.id`，文案是 “Application credentials are missing. Please try again later.”

| 走到哪 | 平台凭证（id 在白名单） | 只是 `reseller_id = 0`，id 不在白名单 | BU 凭证 |
|--------|------------------------|----------------------------------------|---------|
| 登录找商户 `AuthController.login` | 只按 `email` 全局找一条 | 按 `merchant.reseller_id = 0` 找 | 按签名上的 `reseller_id`，并且 `username` 或 `email` 相等 |
| 邀请码范围 | `scopedResellerId` 为 `null`，全库 | `normalizeScopeResellerId` 把 `0` 也当成全库 | 只查该 BU |
| 已登录请求 `OpenApiSignatureFilter` | 跳过「签名 reseller 必须等于会话 reseller」 | 不跳过。会话里是商户自己的 BU，对不上就是 401 | 必须相等 |

会话里的 reseller 用 `merchant.reseller_id`，不用签名上的 0。登录成功之后的这一段见 `02-邮箱注册登录`。

客户端要填的和库里留下的不是同一半。`ResellerApiBuServiceImpl.create` 入库的是加密后的 `app_secret`、加密后的请求私钥、明文响应公钥。当场返回、只此一次的是明文 secret、请求公钥、响应私钥。Android 写在 `openapi/<channel>.properties`（gitignore，键是 `APP_ID`、`APP_SECRET`、`REQUEST_PUBLIC_PEM`、`RESPONSE_PRIVATE_PEM`）。没有这个文件时 `OpenApiSupport.loadProperties` 给空配置，`OpenApiInterceptor` 发现 app id 或 secret 为空就不签名、不加密。所以单读 `reseller_api` 配不齐客户端那一半。

验签读的是 Redis 里的 `ResellerApiCredentialCacheEntry`，里面有 `id` 和 `resellerId`，不每次查表。创建接口会 `refreshByAppId`。直接改库不会刷新，改完仍按缓存里的旧 id 和旧 BU 验，直到全量刷新或 api 重启预热。

测试库 `reseller_api` 里，`reseller_id = 0` 且 status 为 1 的有：id 1 Platform iOS App、2 Platform Android App、3 Platform WAP App、4 名字为空、5 名字 `android-tst`。按代码默认白名单，1 到 5 都是平台凭证。id 1014 名字 `android test` 仍挂在 reseller 101，是 BU 凭证。

平台凭证、请求里没带邀请码时，只用 Nacos `merchant.inviteCode`，代码默认 `cccccc`。测试库这条码是 `agent_invite_code.id` 84，`reseller_id` 60，`is_default = 0`。所以公司码空着、用的又是平台凭证时，新商户落到 60，不是密钥最初挂的那个 BU。reseller 101 的默认码是 `agent_invite_code.id` 148，只有 BU 凭证没填码才会用它。

邮件「已配置」指生产库 `reseller_email_config` 有六类且 `status = 1`：`alert`、`auto`、`info`、`no-reply`、`notify`、`report`。下表这些生产 BU 都是这样。SMTP 不写。

生产和测试不是同一套 `reseller.id`。生产、UAT 的 AppId 在 `pro`。测试 AppId 在 `test`。下面域名表是部署对照；某一格和库里那一列不一致时，写在「和库、当前构建对不上」。

## 品牌和构建

H5 `bu.config.cjs` 里 `bu1` 上面有注释「已下架，不再参与构建」，对象仍导出。`bu5` 整段被注释，不在导出里。Web `package.json` 有 `bu1`、`bu2`、`bu4` 到 `bu14`、`bu999`，没有 `bu15`、`bu16`。

| 品牌 | 构建名 | 生产 BU ID | 生产品牌名（`brand_name`） | 生产 / UAT AppId | 邮件 |
|------|--------|------------|------------------------------|------------------|------|
| Virtronex | bu2 | 44 | Virtronex | `E403DB3CC0DC498ABE9A425CD23E8B29` | 已配置 |
| CryptoPag | bu1 | 45 | CryptoPag | `490FF4132DC94BD49A68A2EFCCD5D9BE` | 已配置 |
| PayJust | bu4 | 48 | PayJust | `B0A53E0A0C04400EA1B914F2956A8C6C` | 已配置 |
| AurexPay | bu5 | 54 | AurexPay | `878CD977D13B49089D18DF1140DD3A89` | 已配置 |
| PayStation | bu6 | 55 | PayStation | `1B7F2B3C963B4B018EAE9BF83E4CE666` | 已配置 |
| Site workers | bu7 | 57 | site-workers | `0B38B31EEBA3428A97CE04A6EEBAC656` | 已配置 |
| WorkArd | bu8 | 58 | WorkArd | `007D9CDEC96747089C354A2FC45317D9` | 已配置 |
| TrueCard | bu9 | 59 | TrueCard | `587F8E64A78C4B25B10B6BC9BAD5E099` | 已配置 |
| Empirepay | bu10 | 60 | Empirepay | `802B5FF23D0F428EA9AADEB738345A0A` | 已配置 |
| Atriapay | bu11 | 61 | Atriapay | `E2B65BBC84B64794AACB49E05968E83A` | 已配置 |
| Americard | bu12 | 62 | Americard | `D126C57DBF124BFFAA505FC731D6DE7C` | 已配置 |
| Tyconcard | bu13 | 63 | Tyconcard | `FA10E331F3BB43C0B0BDFE79F868A6FC` | 已配置 |
| Cartoblu | bu14 | 64 | Cartoblu | `230CC3C21A8C47B18CFDA388E7F06AC9` | 已配置 |
| Sucseesstory | bu15 | 65 | Sucseesstory | `37A3F097F3FB4480AE22919D47B8106A` | 已配置 |
| sucsessstory | bu16 | 66 | sucsessstory | `7AE9F040DF2343B59DA170DFC89C2647` | 已配置 |
| demo | bu999 | 见测试库 110 | 配置里 `buid` 写的是 999，生产库没有 id 999 | 无生产 AppId | 生产无此行 |

上表生产 AppId 与 `pro.reseller_api` 里 `status = 1` 的那条「外部 api」一致。PayJust 另外还有两条，不在原表里：Android `2997F1729960410B8C70F2B9DED6B237`，iOS `AC751FDF88264E72B9905960647D4E32`，都挂在 reseller 48。

`reseller_id = 0` 的平台凭证另计，不是上表某一品牌。测试库里平台 WAP / 当前 H5 共用的一条是 `7EAC723EB1D940B78F6C23713B0C6247`。

## 生产域名

库里每个 BU 只存一个旧后台、一个新后台、一个 Web。格里写了两个地址的，第二个不在 `reseller` 这三列里，当部署别名留下。Web 列库里多半带 `/auth`，这里写站点根，和原表一致。H5 的 `m.` 与 `bu.config.cjs` 的 `mDomain` 一致（`bu5` 只在被注释的那段里）。

| 构建 | 旧管理后台 | 新管理后台（xfeng） | Web PC（kafuka） | Web App / H5（sean） |
|------|------------|---------------------|------------------|----------------------|
| bu2 | https://virtronex.virtronex.com | https://evirtronex.virtronex.com | https://app.virtronex.com | https://m.virtronex.com |
| bu2 另有 | https://some.virtronex.com | https://nobody.virtronex.com | | |
| bu1 | https://afdsafds.cryptopagregistro.com.br | https://eafdsafds.cryptopagregistro.com.br | https://app.cryptopagregistro.com.br | https://m.cryptopagregistro.com.br |
| bu4 | https://jfiow7689.payjust.io | https://ejfiow7689.payjust.io | https://app.payjust.io | https://m.payjust.io |
| bu5 | https://aurex.aurexpay.io | https://eaurex.aurexpay.io | https://app.aurexpay.io | https://m.aurexpay.io |
| bu6 | https://paystation.paystation.money | https://epaystation.paystation.money | https://app.paystation.money | https://m.paystation.money |
| bu7 | https://site-workers.site-workers.com | https://esite-workers.site-workers.com | https://app.site-workers.com | https://m.site-workers.com |
| bu8 | https://workard.workard.co | https://eworkard.workard.co | https://app.workard.co | https://m.workard.co |
| bu9 | https://truecard.truecard.pro | https://etruecard.truecard.pro | https://app.truecard.pro | https://m.truecard.pro |
| bu10 | https://empirepay.empirepay.co | https://eempirepay.empirepay.co | https://app.empirepay.co | https://m.empirepay.co |
| bu11 | https://atriapay.atriapay.co | https://eatriapay.atriapay.co | https://app.atriapay.co | https://m.atriapay.co |
| bu12 | https://americard.americard.co | https://eamericard.americard.co | https://app.americard.co | https://m.americard.co |
| bu13 | https://tyconcard.tyconcard.com | https://etyconcard.tyconcard.com | https://app.tyconcard.com | https://m.tyconcard.com |
| bu14 | https://cartoblu.cartoblu.co | https://ecartoblu.cartoblu.co | https://app.cartoblu.co | https://m.cartoblu.co |
| bu15 | https://sucseesstory.sucseesstory.com | https://esucseesstory.sucseesstory.com | https://app.sucseesstory.com | https://m.sucseesstory.com |
| bu16 | 库里 `admin_url` 为空 | https://esucsessstory.sucsessstory.com | https://app.sucsessstory.com | https://m.sucsessstory.com |

WorkArd 的站点是 `workard.co`，`reseller.domain` 存的是 `workard.com`。

## UAT 域名

原表把 UAT 和同一套生产 AppId 放在一起。这些 `uat-` 主机除 Virtronex 旧后台、Web 以外，没有各自的 `reseller` 行。生产库另有 reseller 49，实体名 `UAT_Virtronex`：`admin_url` 是 `https://uat-virtronex.virtronex.com`，`web_app_url` 是 `https://uat-app.virtronex.com/auth`，`new_admin_url` 为空。

| 构建 | 旧管理后台 | 新管理后台 | Web PC | Web App / H5 |
|------|------------|------------|--------|----------------|
| bu2 | https://uat-virtronex.virtronex.com | https://uat-evirtronex.virtronex.com | https://uat-app.virtronex.com | https://uat-m.virtronex.com |
| bu2 另有 | | https://uat-nobody.virtronex.com | | |
| bu1 | https://uat-afdsafds.cryptopagregistro.com.br | https://uat-eafdsafds.cryptopagregistro.com.br | https://uat-app.cryptopagregistro.com.br | https://uat-m.cryptopagregistro.com.br |
| bu4 | https://uat-jfiow7689.payjust.io | https://uat-ejfiow7689.payjust.io | https://uat-app.payjust.io | https://uat-m.payjust.io |
| bu5 | https://uat-aurex.aurexpay.io | | https://uat-app.aurexpay.io | https://uat-m.aurexpay.io |
| bu6 | https://uat-paystation.paystation.money | | https://uat-app.paystation.money | https://uat-m.paystation.money |
| bu7 | https://uat-site-workers.site-workers.com | | https://uat-app.site-workers.com | https://uat-m.site-workers.com |
| bu8 | https://uat-workard.workard.co | | https://uat-app.workard.co | https://uat-m.workard.co |
| bu9 | https://uat-truecard.truecard.pro | | https://uat-app.truecard.pro | https://uat-m.truecard.pro |
| bu10 | https://uat-empirepay.empirepay.co | | https://uat-app.empirepay.co | https://uat-m.empirepay.co |
| bu11 | https://uat-atriapay.atriapay.co | | https://uat-app.atriapay.co | https://uat-m.atriapay.co |
| bu12 | https://uat-americard.americard.co | | https://uat-app.americard.co | https://uat-m.americard.co |
| bu13 | https://uat-tyconcard.tyconcard.com | | https://uat-app.tyconcard.com | https://uat-m.tyconcard.com |
| bu14 | https://uat-cartoblu.cartoblu.co | | https://uat-app.cartoblu.co | https://uat-m.cartoblu.co |
| bu15 | | https://uat-sucseesstory.sucseesstory.com | https://uat-app.sucseesstory.com | https://uat-m.sucseesstory.com |
| bu16 | | https://uat-esucsessstory.sucsessstory.com | https://uat-app.sucsessstory.com | https://uat-m.sucsessstory.com |

## 测试部署域名

这些是原表左侧的测试站点，不是测试库里「这个构建签到哪个 reseller」的那一行。对不上的见下一节。

| 构建 | 旧管理后台 | 新管理后台 | Web PC | Web App / H5 |
|------|------------|------------|--------|----------------|
| bu2 | https://testbu3.vtron-card.com | https://test-new-admin.vtron-card.com 、 https://test-new-bu1.vtron-card.com | https://testbu3-web.vtron-card.com | https://test-m.vtron-card.com |
| bu1 | https://testbu2.vtron-card.com | https://test-new-bu.vtron-card.com | https://test-web-bu2.vtron-card.com | https://test-bu1-m.vtron-card.com |
| bu4 | https://testbu4.vtron-card.com | https://test-new-bu1.vtron-card.com | https://testbu4-web.vtron-card.com | https://test-bu4-m.vtron-card.com |
| bu5 | https://testbu5.vtron-card.com | | https://testbu5-web.vtron-card.com | https://test-bu5-m.vtron-card.com |
| bu6 | https://testbu6.vtron-card.com | | https://testbu6-web.vtron-card.com | https://test-bu6-m.vtron-card.com |
| bu7 | https://testbu7.vtron-card.com | | https://testbu7-web.vtron-card.com | https://test-bu7-m.vtron-card.com |
| bu8 | https://testbu8.vtron-card.com | | https://testbu8-web.vtron-card.com | https://test-bu8-m.vtron-card.com |
| bu9 | https://testbu9.vtron-card.com | | https://testbu9-web.vtron-card.com | https://test-bu9-m.vtron-card.com |
| bu10 | https://testbu10.vtron-card.com | | https://testbu10-web.vtron-card.com | https://test-bu10-m.vtron-card.com |
| bu11 | https://testbu11.vtron-card.com | | https://testbu11-web.vtron-card.com | https://test-bu11-m.vtron-card.com |
| bu12 | https://testbu12.vtron-card.com | | https://testbu12-web.vtron-card.com | https://test-bu12-m.vtron-card.com |
| bu13 | https://testbu13.vtron-card.com | | https://testbu13-web.vtron-card.com | https://test-bu13-m.vtron-card.com |
| bu14 | https://testbu14.vtron-card.com | | https://testbu14-web.vtron-card.com | https://test-bu14-m.vtron-card.com |
| bu15 | | | https://testbu15-web.vtron-card.com | https://test-bu15-m.vtron-card.com |
| bu16 | 原表空白 | | | |
| bu999 | https://demo-admin.msynergy.co 、 https://demo-bu999.msynergy.co | 原表把这两个都写在旧后台列 | https://demo-web.msynergy.co | https://demo-m.msynergy.co |

## 测试库里实际签到哪

原表右侧是测试环境的实体名、邀请码、BU ID、AppId。以 `test.reseller` / `reseller_api` / `agent_invite_code` 为准。AppId 能对上的，BU ID 就用该行的 `reseller_id`。

| 构建 | 库中实体名 | 邀请码 | 测试 BU ID | 测试 AppId | 这条 AppId 现在谁在用 |
|------|------------|--------|------------|------------|------------------------|
| bu2 | realbu6 | `test666` 在，`is_default = 0`。同 BU 还有 `test6`，也不是默认 | 88 | `54D25B51A17B4B7AABD5837AC3448B44` | Web `.env.bu2`。H5 `.env.bu2.test` 这一行被注释，当前用的是平台 AppId `7EAC723E…` |
| bu1 | sara | `new777` 在，不是默认。默认是 `new222` | 58 | `C37BEDBEDD134635B3CE70D592F2EC5C` | H5 `.env.bu1.*`。Web `.env.shared` 也是这一条，所以没单独写 AppId 的 Web 构建都会签到 58 |
| bu4 | xfbu（原表写成 payjust） | `xfagent`，且是默认 | 101 | `9ABE293FCEA7441083C327B8DAB08798` | 在库里，名称 android test。当前 H5 `.env.bu4.test` 用的是 88 的 `54D25B51…`。Web `.env.bu4` 没覆盖 AppId，实际是 shared 的 58 |
| bu5 | connie | 默认码是 `connie`，原表这一格是空的 | 106（原表写成 88） | `BD121CD25C1E41858C3E08C13D52F17C` | Web `.env.bu5`。H5 `.env.bu5.test` 用的是 88 的 `54D25B51…`，而且 `bu5` 已从 `bu.config.cjs` 导出里拿掉 |
| bu6–bu15 | 原表名称为空。H5 这些 env 签的是 88 / realbu6 | | 88 | `54D25B51A17B4B7AABD5837AC3448B44` | H5 `.env.bu6` 到 `.env.bu15` 的 test/dev。Web 对应 `.env.buN` 多数没写 AppId，实际落 shared，即 reseller 58 |
| bu16 | 原表测试列为空 | | H5 env 仍是 88 | 同上 `54D25B51…` | 只有 H5。Web 没有 `bu16` 脚本 |
| bu999 | KDemo | 原表为空。库里默认是 `KlicklTest` | 110 | `C4C6BD54EF424AA39E71FA856309AD95` | H5 `.env.bu999.test`、Web `.env.bu999`。`bu.config.cjs` 的 `buid` 仍是 999 |

reseller 88 存在库里的后台地址是旧后台 `https://testbu6.vtron-card.com`、新后台 `https://test-new-bu4.vtron-card.com`、Web `https://testbu6-web.vtron-card.com/auth`，品牌名存成 PayStation。和上表 bu2 的测试部署域名不是同一组。

demo 还有 reseller 109，实体名 Rivine Inv，`new_admin_url`、`web_app_url` 也指向 `demo-bu999` / `demo-web`。H5 `.env.bu999.dev` 用的是另一条 AppId `7CEE9E735CDA440E98FADB9AFB87905B`，挂在 109，不是 110。

## 和库、当前构建对不上

- 原表 AurexPay 测试 BU ID 写成 88，和它旁边的 AppId 不是一行。该 AppId 属于 reseller 106。
- 原表 PayJust 测试实体名写成 payjust。库里 101 的 `entity_name` 是 xfbu。当前 H5、Web 的 bu4 构建也没有用 101 这条 AppId。
- 原表 Virtronex 测试 AppId 是 `54D25B51…`。Web bu2 仍是这条。H5 `env/.env.bu2.test` 改成了平台凭证，原来的 AppId 在注释里。
- 原表 bu6 到 bu15 的测试身份是共用 88。H5 的 env 是这样。Web 没覆盖 AppId 的构建签到的是 reseller 58。
- 原表左侧测试域名和右侧那个 reseller 行的 `admin_url` / `new_admin_url` / `web_app_url` 经常不是同一组主机。部署域名看上面「测试部署域名」，签名落到哪个 BU 看「测试库里实际签到哪」。
- 生产 Virtronex 的 `some.`、`nobody.` 以及各品牌多数 `uat-` 新后台，不在 `reseller` 那一格里。新后台按 Host 匹配时，只认 `new_admin_url` 里存的那一个。

## 测试后台登录账号

DBX 连接 `test`，库 `virtronex_business.admin_user`。2026-09-23 对过。这两条不是 C 端 `merchant`，`merchant` 里没有同名邮箱。`type` 见 `AdminUserType`：2 是 Reseller，3 是 Agent。两条都是 `status = 1`、`del_flag = 0`、`google_status = 0`。密码是哈希，这里不写。

| 角色 | admin_user.id | type | 挂在哪 | 邮箱 |
|------|----------------|------|--------|------|
| BU | 177 | 2 | reseller 101，`entity_name` 是 xfbu | 本地部分 `xfbu`，域名 `gmail.com` |
| Agent | 183 | 3 | `parent_id = 177`。邀请码 `xfagent`，`agent_invite_code.id` 148，`is_default = 1`，`reseller_id` 101 | 本地部分 `xfagent`，域名是 `gamil.com`，不是 `gmail.com` |

## Android、iOS

Android `dev` 不走 `buN`。`ChannelRegistry` 只有两个渠道：

| 渠道 | 包名 | tst | uat | product |
|------|------|-----|-----|---------|
| virtronex | `com.vtron.pay` | `https://test-ad-api.vtron-card.com/api/v1/` | `https://uat-api.virtronex.com/api/v1/` | `https://android-api.virtronex.com/api/v1/` |
| payjust | `com.payjust.pay` | 同上测试 API | `https://uat-ad-api.payjust.io/api/v1/` | `https://ad-api.payjust.io/api/v1/` |

iOS `release` 搜到的主机是 virtronex（`uat-api.virtronex.com`、`terms.virtronex.com`、`file.virtronex.com`）。`X-App-Id` 从编译进包的密钥读，这次没有解出具体值。没有 `buN` 这套构建名。
