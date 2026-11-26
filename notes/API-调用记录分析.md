# 龙湖天街小程序 API 调用记录分析

> 抓包来源:`Stream-2026-10-13 18:07:19.har`
> 抓包时间:2026-10-13 18:04:46 ~ 18:05:00(北京时间,共 14 秒)
> 场景:打开龙湖天街小程序 → 进入签到页 → 完成当日签到
> 服务器:`8.146.203.244`(`gw2c-hw-open.longfor.com`)
> 已过滤:CONNECT 隧道、OPTIONS 跨域预检、重复请求

---

## 🎯 总览:调用时序(按时间顺序)

| # | 时间 | 方法 | 路径 | 作用 | 重要性 |
|---|------|------|------|------|--------|
| 1 | 18:04:46 | GET | `/supera/member/.../v1/user-info` | 获取会员信息(等级/昵称/成长值) | 🟡 |
| 2 | 18:04:47 | POST | `/supera/mine/.../publicApi/project/near` | 查询附近天街门店(定位) | ⚪ |
| 3 | 18:04:47 | POST | `/supera/C2/.../publicApi/city/getCityInfo` | 根据定位获取城市信息 | ⚪ |
| 4 | 18:04:47 | POST | `/supera/basic/.../getBySceneCode` | 首页弹窗频控配置 | ⚪ |
| 5 | 18:04:47 | POST | `/supera/basic/.../popup/listPopups` | 拉取首页弹窗列表 | ⚪ |
| 6 | 18:04:49 | POST | `/supera/mine/.../message/findUnreadCount` | 未读消息数 | ⚪ |
| 7 | 18:04:49 | GET | `/supera/member/.../v2/user-lz-balance` | 珑珠余额 | 🟡 |
| 8 | 18:04:49 | POST | `/supera/member/.../v2/getCouponInfo` | 优惠券数量 | ⚪ |
| 9 | 18:04:49 | POST | `/supera/member/.../v1/pageConfig` | 「我的」页面配置 | ⚪ |
| 10 | 18:04:50 | POST | `/supera/member/.../v1/my-info` | 我的页面模块数据 | ⚪ |
| 11 | 18:04:55 | POST | `/supera/mine/.../token/token2key` | token 换取密钥 | 🟡 |
| 12 | 18:04:57 | POST | `/llt-gateway-prod/api/v1/activity/common/project/info` | 签到活动项目信息 | 🟡 |
| 13 | 18:04:57 | GET | `/lmarketing-task-api-mvc-prod/openapi/task/v1/verify-token` | **签到前置:验证 token** | 🔴 |
| 14 | 18:04:58 | POST | `/lmarketing-task-api-mvc-prod/openapi/task/v1/signature/page-info` | **签到页数据(连签状态)** | 🔴 |
| 15 | 18:04:59 | POST | `/lmarketing-task-api-mvc-prod/openapi/task/v1/signature/clock` | **⭐ 执行签到** | 🔴 |
| 16 | 18:04:59 | POST | `/supera/cms/.../getPageData/C2CheckPage` | 签到页 CMS 轮播图 | ⚪ |

**🔴 核心签到链 = #13 → #14 → #15**(即 `longfor.py` 只需调用的接口)

---

## 🔴 核心接口详情

### 1️⃣ verify-token — 签到前置校验

```
GET https://gw2c-hw-open.longfor.com/lmarketing-task-api-mvc-prod/openapi/task/v1/verify-token?__no_cache__=0.26659371355817496
```

**关键请求头:**
```
X-GAIA-API-KEY:  c06753f1-3e68-437d-b592-b94656ea5517
X-LF-UserToken:  bd89771db57f4556a1f1fee9a93033fc    ← 登录 token
X-LF-Bu-Code:    C20400
X-LF-Channel:    C2
Origin:          https://longzhu.longfor.com
```

**响应(45ms 完成):**
```json
{"code":"0000","message":"","data":"SUCCESS"}
```

> 💡 `__no_cache__` 参数是随机数,防止 CDN 缓存。校验失败大概率先触发风控。

---

### 2️⃣ page-info — 签到页数据(可选,用于展示)

```
POST https://gw2c-hw-open.longfor.com/lmarketing-task-api-mvc-prod/openapi/task/v1/signature/page-info
Content-Type: application/json;charset=UTF-8
token: bd89771db57f4556a1f1fee9a93033fc        ← 双 header 鉴权
X-LF-UserToken: bd89771db57f4556a1f1fee9a93033fc
```

**请求体:**
```json
{"activity_no": "11111111111686241863606037740000"}
```

**响应(节选):**
```json
{
  "code": "0000",
  "data": {
    "task_name": "日日签 日日赚",
    "task_show_status": 20,          // 20 = 活动正常进行(签到前/后都是 20,不代表已签)
    "sign_again_num": 3,             // 签到前 3 → 签到后 2(递减,疑似剩余次数)
    "day_received_rewards": [],      // 空 = 今日还没领奖
    "seven_days_signs": [            // 7天签到日历
      {"sign_day": 13, "sign_status": 10, "sign_rewards": [{"reward_num": 5.0}]},
      {"sign_day": 14, "sign_status": 10, "...": "连签3/5/7天有额外奖励"}
    ]
  }
}
```

> 💡 `sign_status`:10 = 未签,20 = 已签。
>
> ⚠️ **判断“今天是否已签”只看两项**：当日 `sign_status == 20`、`day_received_rewards` 非空。
> `task_show_status` 恒为 20，与是否签到无关（2026-10-13 抓包：签到前 `sign_again_num=3`、当日 `sign_status=10`；签到成功后变为 `sign_again_num=2`、当日 `sign_status=20`、`day_received_rewards=[{reward_num:5.0}]`）。

---

### 3️⃣ clock — ⭐ 执行签到(核心)

```
POST https://gw2c-hw-open.longfor.com/lmarketing-task-api-mvc-prod/openapi/task/v1/signature/clock
Content-Type: application/json;charset=UTF-8
Cookie: acw_tc=2760776b1791... (来自 verify-token 响应)
```

**关键请求头:**
```
X-GAIA-API-KEY:           c06753f1-3e68-437d-b592-b94656ea5517
token / X-LF-UserToken:   bd89771db57f4556a1f1fee9a93033fc
X-LF-Bu-Code:             C20400
X-LF-Channel:             C2
X-LF-DXRisk-Source:       5
X-LF-DXRisk-Token:        6ace0249ImdSe8IGS8PU3j5tFuRSWuxFI2OsqnL1   ← 顶象风控 token
X-LF-DXRisk-Captcha-Token: undefined                                  ← 无验证码时原样传 undefined
```

**请求体:**
```json
{"activity_no": "11111111111686241863606037740000"}
```

**响应(582ms):**
```json
{
  "code": "0000",
  "message": "",
  "data": {
    "is_popup": 1,
    "reward_info": [
      {"reward_type": 20, "reward_num": 5.0000, "sign_type": 10}
    ],
    "current_date_time": "1791885899774"
  }
}
```

> 💡 `reward_type: 20` = 成长值。本次签到获得 **+5 成长值**。
> 💡 有趣的细节:`X-LF-DXRisk-Captcha-Token: undefined` —— 服务端不校验该头是否存在有效值,只要 `X-LF-DXRisk-Token` 风控评分通过即可。触发风控时才需要滑块验证产生的真实 captcha-token。

---

## 🟡 辅助接口详情

### user-info — 会员信息(可作为 token 有效性探测)

```
GET https://gw2c-hw-open.longfor.com/supera/member/api/bff/pages/v2_4_2/v1/user-info
X-Gaia-Api-Key: 98717e7a-a039-46af-8143-be7558a089c0   ← 注意:与签到接口的 KEY 不同!
lmToken: bd89771db57f4556a1f1fee9a93033fc
```

**响应:**
```json
{
  "code": "0000",
  "data": {
    "nickName": "柯某",
    "desensitizePhone": "173****0883",
    "levelName": "V2",
    "growthValue": 103,          // 成长值(签到前)
    "nextGrowthValue": 2000,
    "equityQuantity": 15
  }
}
```

> 💡 **签到后 growthValue 应变为 108**(+5)。这个接口适合用来探测 `lmToken` 是否还有效。

### token2key — token 换密钥

```
POST https://gw2c-hw-open.longfor.com/supera/mine/v2_4_2/token/token2key
```

**响应:**
```json
{"code": "200", "data": {"key": "Y2U4MDU3MmFLYTgxMEs0ZTEzSzkxY2ZLOTljYmY3MDBjNWI0"}}
```

> 💡 Base64 解码后:`ce80572aKa810K4e13K91cfK99cbf700c5b4`(`K` 为分隔符)。
> 推测用于后续接口的请求加密/签名,签到链路未用到。

### activity/common/project/info — 活动项目信息

```
POST https://gw2c-hw-open.longfor.com/llt-gateway-prod/api/v1/activity/common/project/info
X-GAIA-API-KEY: 2f9e3889-91d9-4684-8ff5-24d881438eaf   ← 又一个不同的 KEY
```

**请求体:**
```json
{"tool_no": "11111111111686241863606037740000"}
```

**响应:**
```json
{"code": "0000", "data": {}}
```

---

## 🔐 鉴权体系观察

| 特征 | supera/* 接口 | lmarketing-task/* 接口 |
|---|---|---|
| 登录凭证 | `lmToken` | `token` + `X-LF-UserToken`(双头) |
| API Key | `98717e7a-...` | `c06753f1-...` |
| 请求签名 | 需要 `X-LONGZHU-Sign` + `X-LONGZHU-TimeStamp` | ❌ 不需要 |
| 风控头 | 不需要 | `X-LF-DXRisk-Token`(必须) |

**结论:**
- 🔴 签到三件套(verify-token / page-info / clock)**无请求签名**,只靠 token + 固定 API-Key + 顶象风控 token → 这正是脚本可自动化的原因。
- ⚪ `supera/*` 系列需要 `X-LONGZHU-Sign`(请求体+时间戳的 MD5/HMAC 签名),脚本未实现也无法轻易伪造 → 所以脚本只调签到链路。
- 服务端共出现 **3 个不同的 `X-GAIA-API-KEY`**,按网关分组:`98717e7a`(member/bff)、`c06753f1`(lmarketing-task)、`2f9e3889`(llt-gateway)。

---

## 📊 风控与验证码线索

1. **正常签到路径**(本次抓包):`verify-token` 通过 → `clock` 直接触发,响应 `code: 0000`。
2. **风控路径**(README/脚本描述):`clock` 返回 `code: 8040012` → 前端拉起顶象滑块 → 通过后携带 `X-LF-DXRisk-Captcha-Token: <真实token>` 重试 `clock`。
3. 本次 `X-LF-DXRisk-Captcha-Token` 传的是字符串 `"undefined"`,说明低风险场景下该头**形同虚设**,真正的风控门槛在 `X-LF-DXRisk-Token`。

---

## 🗒️ 附:本次签到结果

| 项目 | 值 |
|---|---|
| 签到日期 | 2026-10-13 |
| 连签天数 | 第 3 天(`sign_again_num: 3`) |
| 获得 | 成长值 +5(`reward_type: 20`) |
| 后续连签奖励 | 10-15(连3天)+3、10-17(连5天)+5、10-19(连7天)+5 |
| 7天周期 | 10-13 ~ 10-19 |
