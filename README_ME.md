# 我的使用手册（个人备忘）

> 给未来的自己看的：我用这套程序干了什么、怎么跑、出问题怎么查。
> Token 的具体抓法见 [TOKEN_README.md](./TOKEN_README.md)，官方文档见 [docs/](./docs/)。
> 本文件**不要写入真实 token**。

---

## 1. 我采用的方案

| 场景 | 方式 |
| --- | --- |
| 日常自动签到 | **GitHub Actions**（私有仓库，每天北京时间 09:00） |
| 本地手动验证 / 排查 | 本地 `.venv` + `export` + `python main.py` |
| 不用的方案 | Docker、华为云 FunctionGraph（暂未部署） |

Actions 里用 `actions/cache` 保存 `.state`，让 token 能跨运行自动续期（见第 5 节）。

---

## 2. 一次性准备清单

- [ ] 私有仓库为 Fork 或自建，**Actions 已启用**
- [ ] Repository secrets：
  - `LONGFOR_TOKEN`：小程序登录 token（= 抓包里的 `X-LF-UserToken`）
  - `LONGFOR_DXRISK_TOKEN`：顶象风控 token（= `X-LF-DXRisk-Token`）
  - `BARK_KEY`：可选，手机推送
- [ ] 本地虚拟环境：`/Users/shengxiangke/Code/auto-sign-in/.venv`（Python 3.9）
- [ ] 依赖已装：`../.venv/bin/pip install -r requirements.txt`

---

## 3. 本地怎么跑

```bash
cd /Users/shengxiangke/Code/auto-sign-in/longfor-sign
source ../.venv/bin/activate

export LONGFOR_TOKEN="抓包得到的 X-LF-UserToken"
export LONGFOR_DXRISK_TOKEN="抓包得到的 X-LF-DXRisk-Token"

python main.py
echo "exit=$?"     # 0=全部成功，1=有账号未成功
```

- `.state/credentials.json` 会保存续期后的 token（**别删**，已 gitignore）
- 想看验证码图片：`export SAVE_CAPTCHA_IMAGES=true`，输出到 `captcha_collection/`

---

## 4. token 怎么更新（多久一次）

**只要 Actions 每天在跑，通常不需要手动更新。** 出现下面的情况才需要重新抓包：

- 日志出现 `801001 您的登录已过期`
- 或 cache 显示 not found、且旧 token 已过期
- 或自己在小程序里退出登录过（会顶掉旧 token）

抓包步骤（详见 TOKEN_README.md）：

1. 抓包工具（Stream/Charles/Proxyman）开好，打开龙湖天街小程序
2. 进入「我的」→ 退出登录 → 重新登录
3. 筛 `gw2c-hw-open.longfor.com`，打开任意请求（如 `v1/user-info`）
4. 复制 `X-LF-UserToken` 和 `X-LF-DXRisk-Token`
5. 更新 GitHub Secrets（或本地 `export`）

> 注意：**重新登录会让旧 token 立刻失效**（表现为 `801001`），所以抓完立刻更新 Secrets。

---

## 5. Actions 里我加了什么（怕以后忘了为什么）

`.github/workflows/daily-sign.yml`：

| 步骤 | 作用 |
| --- | --- |
| `恢复续期状态` `actions/cache/restore@v4` | 用前缀 `longfor-state-` 取上一次的 `.state` |
| `执行签到` | `python main.py` |
| `保存续期状态` `actions/cache/save@v4` | key = `longfor-state-<run_id>-<run_attempt>`，签到失败也存 |
| `失败提示` | 把 801001 / 801810 / 8040012 的处置方式打到日志 |
| `上传验证码图片` | 原有逻辑，保留 30 天 |

关键点：

- `run_id` 每次运行唯一，`run_attempt` 区分重跑（重跑同一次时 key 不会重复）
- `concurrency: longfor-daily-sign` **不要删**，否则并发续期会互相顶号
- cache 7 天不访问会被清理；连续每天跑就能一直保活
- 私有仓库不受「60 天不活动自动停用」限制

检查缓存：

```bash
gh cache list --key longfor-state
gh api repos/:owner/:repo/actions/caches --jq '.actions_caches[] | {key,size_in_bytes,last_accessed_at}'
```

---

## 6. 结果怎么看

**Actions 日志关键字（按顺序）**

| 日志 | 含义 |
| --- | --- |
| `Cache restored from key: longfor-state-...` | 续期链路正常 |
| `登录 token 续期成功` | token2key → key2token 成功 |
| `签到成功` | 当天完成 |
| `需要人工核实或验证（code=801810）` | 活动侧风控，非 token 问题 |
| `801001` | token 失效，需重新抓包 |
| `Cache saved with key: ...` | 状态已落盘，第二天可续 |

**退出码**：`0` = 全部成功；`1` = 有账号未成功（不等于 token 有问题，要看 code）。

---

## 7. page-info 怎么读（重要，别被字段名骗了）

```bash
# 只读接口，随便调，不会触发风控/签到
curl -s -X POST "https://gw2c-hw-open.longfor.com/lmarketing-task-api-mvc-prod/openapi/task/v1/signature/page-info" \
  -H "Content-Type: application/json;charset=UTF-8" \
  -H "Origin: https://longzhu.longfor.com" -H "Referer: https://longzhu.longfor.com/" \
  -H "token: $LONGFOR_TOKEN" -H "X-LF-UserToken: $LONGFOR_TOKEN" \
  -H "X-GAIA-API-KEY: c06753f1-3e68-437d-b592-b94656ea5517" \
  -H "X-LF-Bu-Code: C20400" -H "X-LF-Channel: C2" \
  -d '{"activity_no":"11111111111686241863606037740000"}' | python3 -m json.tool
```

| 字段 | 正确理解 |
| --- | --- |
| `task_show_status` | **恒为 20**，只是"活动正常"，**不能**用来判断今天签没签 |
| 当日 `sign_status` | `10` = 未签，`20` = 已签（只有这一项可靠） |
| `day_received_rewards` | 非空 = 今天已领奖 |
| `sign_again_num` | 签到成功后会递减（实测 3 → 2） |

**判断今天签没签：只看「当日 sign_status == 20」或「day_received_rewards 非空」。**

---

## 8. 返回码对照

| code | 含义 | 我要做什么 |
| --- | --- | --- |
| `0000` | 签到成功 / 请求成功 | 无 |
| `801001` | 登录已过期，token 被顶掉 | 重新抓包 → 更新 Secrets |
| `801810` | 「活动太火爆了，明日再来试试~」，活动侧风控/当日限制，**官方小程序也会返回** | 看 page-info：未签就等次日或人工在小程序验证 |
| `801811` / `801812` | 同类风控码 | 同上 |
| `8040012` / `8040016` | 触发滑块验证码 | 程序会自动处理；失败则人工过一次 |

---

## 9. 踩过的坑（血泪备忘）

1. **`801810` 不是"已签到"的固定含义，也不是被拉黑。** 要结合 `page-info` 的当日 `sign_status` 判断；官方客户端点签到同样返回它。
2. **在小程序里退出/重新登录会把旧 token 顶掉**，脚本立刻变 `801001`，必须换 token。
3. **`page-info` 安全、`clock` 有成本**：排查时先查日历，别反复打 `clock`，失败请求会累积风控分。
4. **别同时开多个入口**（Actions + Docker + 云函数），同一账号并发会互相顶号。
5. **多账号时风控值要一一对应**，新版不再自动复用第一个 `DXRisk token`。
6. **`.state` 别删**：删了等于丢掉续期链，就得重新抓包。
7. **Actions 是"尽力而为"**：cron 可能延迟，cache 可能被清；要求高可用就换 Docker。
8. **下载 cache = 暴露 token**，别贴到任何公开地方。

---

## 10. 升级流程

```bash
cd /Users/shengxiangke/Code/auto-sign-in/longfor-sign
git pull
../.venv/bin/pip install -r requirements.txt   # 依赖可能变
../.venv/bin/python main.py                    # 先本地验证一次
git add .github/workflows/daily-sign.yml       # 如有 CI 改动
```

Actions 侧不需要额外操作，下次定时自动生效。

---

## 11. 关键信息速查

| 项目 | 值 |
| --- | --- |
| 签到活动号 | `11111111111686241863606037740000` |
| 接口域名 | `gw2c-hw-open.longfor.com` |
| X-GAIA-API-KEY（签到） | `c06753f1-3e68-437d-b592-b94656ea5517` |
| X-GAIA-API-KEY（key2token） | `f4a2290e-c382-45b7-949c-3253d07e9b5b` |
| 业务线 | `X-LF-Channel: C2` / `X-LF-Bu-Code: C20400` |
| 续期链 | `token2key` → `key2token`（有效期 30 天） |
| 状态目录 | `.state/`（gitignore，含凭据） |
| 本地虚拟环境 | `../.venv`（Python 3.9） |
