# twscrape 使用指南

<div align="center">

[<img src="https://badges.ws/pypi/v/twscrape" alt="版本" />](https://pypi.org/project/twscrape)
[<img src="https://badges.ws/pypi/python/twscrape" alt="Python 版本" />](https://pypi.org/project/twscrape)
[<img src="https://badges.ws/pypi/dm/twscrape" alt="下载量" />](https://pypi.org/project/twscrape)
[<img src="https://badges.ws/github/license/vladkens/twscrape" alt="许可证" />](https://github.com/vladkens/twscrape/blob/main/LICENSE)

</div>

> 基于 [SNScrape](https://github.com/JustAnotherArchivist/snscrape) 数据模型实现的 Twitter GraphQL API 封装库。

---

## 安装

```bash
# 安装稳定版
pip install twscrape

# 安装开发版（最新功能）
pip install git+https://github.com/vladkens/twscrape.git
```

---

## 核心功能（Features）

| 功能 | 说明 |
|------|------|
| **Search & GraphQL API** | 同时支持 Twitter 搜索接口与 GraphQL 接口 |
| **Async/Await** | 异步并发，可同时运行多个爬虫任务 |
| **登录流程（Login Flow）** | 支持邮箱验证码自动接收 |
| **Session 持久化** | 自动保存/恢复账号登录状态到本地数据库 |
| **原始响应 & 结构化模型** | 同时返回原始 API 响应和 SNScrape 数据模型 |
| **账号自动切换** | 触发 Rate Limit 时自动轮换账号，平滑限流 |

---

## 账号准备

> ⚠️ 本项目必须使用已授权的 X/Twitter 账号才能调用 API。

**两种账号方式：**

1. **自建账号**：可在 X/Twitter 官网自行注册，但由于严格的验证流程和**较高的封禁率（ban rate）**，注册难度较大。
2. **使用 Cookie 账号**：购买带有 Cookie 的现成账号，登录问题更少、更稳定。

**免责声明（Disclaimer）**：X/Twitter 服务条款（ToS）不鼓励多账号使用。本工具仅供**数据采集研究**目的，请自行评估风险，合理使用。

---

## 快速上手

```python
import asyncio
from twscrape import API, gather
from twscrape.logger import set_log_level

async def main():
    api = API()  # 默认使用 accounts.db，也可指定路径：API("path-to.db")

    # ── 添加账号 ──────────────────────────────────────────────────

    # 方式一：Cookie 方式（更稳定，推荐）
    cookies = "abc=12; ct0=xyz"  # 也可以是 JSON 格式：'{"abc": "12", "ct0": "xyz"}'
    await api.pool.add_account("user3", "pass3", "u3@mail.com", "mail_pass3", cookies=cookies)

    # 方式二：账号密码方式（需要邮箱接收验证码，不稳定）
    # 注意：部分邮箱服务商不支持 IMAP 协议（如 ProtonMail）
    await api.pool.add_account("user1", "pass1", "u1@example.com", "mail_pass1")
    await api.pool.add_account("user2", "pass2", "u2@example.com", "mail_pass2")
    await api.pool.login_all()  # 尝试登录所有账号以获取 Cookie

    # ── API 调用示例 ──────────────────────────────────────────────

    # 搜索推文（默认 Latest 最新标签）
    await gather(api.search("elon musk", limit=20))  # 返回 list[Tweet]

    # 切换搜索标签（product 可选：Top / Latest（默认）/ Media）
    await gather(api.search("elon musk", limit=20, kv={"product": "Top"}))

    # 推文详情
    tweet_id = 20
    await api.tweet_details(tweet_id)               # 返回 Tweet 对象
    await gather(api.retweeters(tweet_id, limit=20)) # 返回 list[User]

    # 推文回复（注意：X 限制每次分页约 5 条）
    await gather(api.tweet_replies(tweet_id, limit=20))  # 返回 list[Tweet]

    # 通过用户名获取用户信息
    await api.user_by_login("xdevelopers")  # 返回 User 对象

    # 通过用户 ID 获取相关数据
    user_id = 2244994945
    await api.user_by_id(user_id)                            # User
    await gather(api.following(user_id, limit=20))           # 关注列表 list[User]
    await gather(api.followers(user_id, limit=20))           # 粉丝列表 list[User]
    await gather(api.verified_followers(user_id, limit=20))  # 认证粉丝 list[User]
    await gather(api.user_tweets(user_id, limit=20))         # 用户推文 list[Tweet]
    await gather(api.user_tweets_and_replies(user_id, limit=20))  # 推文+回复 list[Tweet]
    await gather(api.user_media(user_id, limit=20))          # 媒体内容 list[Tweet]

    # List 时间线
    await gather(api.list_timeline(list_id=123456789))

    # 热门趋势（Trends）
    await gather(api.trends("news"))   # 新闻类趋势 list[Trend]
    await gather(api.trends("sport"))  # 体育类趋势 list[Trend]

    # ── 两种遍历方式 ──────────────────────────────────────────────

    # 方式一：gather 一次性获取所有结果（返回列表）
    tweets = await gather(api.search("elon musk", limit=10))

    # 方式二：async for 逐条处理（适合大量数据）
    async for tweet in api.search("elon musk"):
        print(tweet.id, tweet.user.username, tweet.rawContent)

    # ── 获取原始响应（Raw Response）────────────────────────────────
    # 所有方法都有对应的 _raw 版本，返回 httpx.Response 对象
    async for rep in api.search_raw("elon musk"):
        print(rep.status_code, rep.json())

    # ── 数据格式转换 ──────────────────────────────────────────────
    doc = await api.user_by_id(user_id)
    doc.dict()   # 转为 Python dict（适合 pandas DataFrame）
    doc.json()   # 转为 JSON 字符串

    # 调整日志级别（Log Level）
    set_log_level("DEBUG")

if __name__ == "__main__":
    asyncio.run(main())
```

---

## 循环中断（break）的正确写法

> ⚠️ 在循环中使用 `break` 时，必须用 `aclosing` 包裹，否则账号锁不会被及时释放。

```python
from contextlib import aclosing

async with aclosing(api.search("elon musk")) as gen:
    async for tweet in gen:
        if tweet.id < 200:
            break  # ✅ ��号锁会被正确释放
```

---

## 命令行（CLI）使用

### 查看帮助

```bash
twscrape              # 显示所有命令
twscrape search --help  # 查看特定命令的帮助
```

### 添加账号

```bash
twscrape add_accounts <账号文件路径> <列格式>
```

**列格式说明（Token）：**

| Token | 是否必填 | 说明 |
|-------|---------|------|
| `username` | ✅ 必填 | 账号用户名 |
| `password` | ✅ 必填 | 登录密码 |
| `email` | ✅ 必填 | 邮箱地址 |
| `email_password` | 可选 | 邮箱密码（用于 IMAP 自动接收验证码） |
| `cookies` | 可选 | Cookie 字符串/JSON/Base64 均可解析 |
| `_` | - | 跳过该列，不解析 |

**示例**（账号文件格式为 `username:password:email:email_password:user_agent:cookies`）：

```bash
# user_agent 列用 _ 跳过
twscrape add_accounts ./order-12345.txt username:password:email:email_password:_:cookies
```

### 登录账号

> 注意：若已通过 Cookie 方式添加账号，无需执行登录步骤。

```bash
twscrape login_accounts
```

登录成功后，Cookie 会自动保存到本地数据库文件，下次直接复用。

**手动输入邮箱验证码**（当邮件服务商不支持 IMAP 时）：

```bash
twscrape login_accounts --manual
twscrape relogin user1 user2 --manual
twscrape relogin_failed --manual
```

### 查看账号状态

```bash
twscrape accounts

# 输出示例：
# username  logged_in  active  last_used            total_req  error_msg
# user1     True       True    2023-05-20 03:20:40  100        None
# user2     True       True    2023-05-20 03:25:45  120        None
# user3     False      False   None                 120        Login error
```

### 重新登录

```bash
twscrape relogin user1 user2      # 重新登录指定账号
twscrape relogin_failed           # 重试所有登录失败的账号
```

### 使用独立账号数据库

```bash
twscrape --db test-accounts.db <命令>
```

### 常用搜索命令

```bash
# 搜索推文
twscrape search "QUERY" --limit=20

# 推文详情与互动
twscrape tweet_details TWEET_ID
twscrape tweet_replies TWEET_ID --limit=20
twscrape retweeters TWEET_ID --limit=20

# 用户信息
twscrape user_by_id USER_ID
twscrape user_by_login USERNAME
twscrape user_media USER_ID --limit=20
twscrape following USER_ID --limit=20
twscrape followers USER_ID --limit=20
twscrape verified_followers USER_ID --limit=20
twscrape user_tweets USER_ID --limit=20
twscrape user_tweets_and_replies USER_ID --limit=20

# 热门趋势
twscrape trends sport
```

**输出重定向到文件：**

```bash
# 默认输出到控制台（stdout），可重定向到文件，每行一条记录
twscrape search "elon musk lang:zh" --limit=20 > data.txt

# 获取原始 API 响应（Raw JSON）
twscrape search "elon musk" --limit=20 --raw
```

### 关于 `limit` 参数

X API 通过**分页（pagination）**返回数据，每个接口的每页默认条数由 X 决定，调用方无法修改。

`twscrape` 中的 `limit` 是**期望获取的对象总数**，工具会尽量返回**不少于**该数量的结果。若 X API 返回的结果多于或少于请求数量，以 X 实际返回为准。

---

## 代理（Proxy）配置

支持四种代理配置方式：

```python
# 方式一：为单个账号配置代理
proxy = "http://login:pass@example.com:8080"
await api.pool.add_account("user4", "pass4", "u4@mail.com", "mail_pass4", proxy=proxy)

# 方式二：为所有账号设置全局代理
api = API(proxy="http://login:pass@example.com:8080")

# 方式三：通过环境变量设置代理
# TWS_PROXY=socks5://user:pass@127.0.0.1:1080 twscrape user_by_login elonmusk

# 方式四：运行时动态切换代理
api.proxy = "socks5://user:pass@127.0.0.1:1080"
doc = await api.user_by_login("elonmusk")   # 使用新代理
api.proxy = None
doc = await api.user_by_login("elonmusk")   # 不使用代理
```

**代理优先级（高 → 低）：**

```
api.proxy  >  环境变量 TWS_PROXY  >  acc.proxy（账号级代理）
```

> 注意：若代理不可用，API 会直接抛出异常。

---

## 环境变量（Environment Variables）

| 变量名 | 默认值 | 说明 |
|--------|--------|------|
| `TWS_PROXY` | 无 | 全局代理地址（如 `socks5://user:pass@127.0.0.1:1080`） |
| `TWS_WAIT_EMAIL_CODE` | `30`（秒）| 等待邮箱验证码的超时时间 |
| `TWS_RAISE_WHEN_NO_ACCOUNT` | `false` | 无可用账号时是否直接抛出 `NoAccountError`（而非等待） |

---

## 限制说明（Limitations）

### Rate Limit（速率限制）

- 每个接口的请求限额**每 15 分钟独立重置**
- 每个账号对不同操作（搜索、浏览主页等）有**各自独立的限额**
- 限额大小因**账号年龄和账号状态**而异

### 数据量上限

| 接口 | 最大返回量 |
|------|-----------|
| `user_tweets` | 约 3200 条 |
| `user_tweets_and_replies` | 约 3200 条 |

### 可行性评估（针对任务 B0）

| 评估维度 | 结论 |
|---------|------|
| **是否可免费获取** | ⚠️ 工具免费，但需要 Twitter 账号（自建难度高，封禁率高） |
| **是否可持续获取** | ⚠️ 依赖非官方 GraphQL 接口，X 更新后可能随时失效 |
| **登录要求** | ✅ 必须，支持 Cookie 方式（更稳定）或账号密码方式 |
| **反爬机制** | 🔴 账号封禁率较高，建议配合代理池 + 多账号轮换 |
| **Rate Limit** | 每 15 分钟重置，多账号自动轮换可缓解 |

---

## 相关资源

- [twitter-advanced-search](https://github.com/igorbrigadir/twitter-advanced-search) — Twitter 高级搜索过滤器指南
- [TweeterPy](https://github.com/iSarabjitDhiman/TweeterPy) — 另一个 X/Twitter 客户端库
- [twitter-api-client](https://github.com/trevorhobenshield/twitter-api-client) — Twitter v1/v2/GraphQL API 实现

---

## 参考文章

- [2023年如何仍然抓取百万推文](https://medium.com/@vladkens/how-to-still-scrape-millions-of-tweets-in-2023-using-twscrape-97f5d3881434)