# mimotion Code Wiki

> 本文档为 `TonyJiangWJ/mimotion` 仓库的结构化代码说明，用于帮助开发者快速理解项目架构、模块职责、关键实现与运行方式。

## 一、项目概述

**mimotion** 是一个基于 GitHub Actions 的 **小米运动 / Zepp Life 自动刷步数** 工具。

- **目标**：按计划定时（北京时间每天多次）模拟小米运动 APP 登录并提交伪造的运动步数，从而同步到支付宝、微信运动等第三方平台。
- **核心特性**：
  - 支持邮箱 / 手机号登录小米运动账号（非小米账号）。
  - 支持多账号批量执行，可选串行或并发执行。
  - 步数随时间线性增长（北京时间 22 点达到最大值），更接近真实行为。
  - 使用 AES 加密保存账号 Token，避免频繁登录被风控。
  - 支持通过 PushPlus、企业微信 Webhook、Telegram Bot 推送执行结果。
  - 自动随机化 GitHub Actions 的 cron 触发分钟数，降低被识别为脚本的概率。
  - 提供配置信息提取工具，用于忘记 Secret 时恢复配置。

## 二、项目整体架构

### 2.1 目录结构

```text
mimotion/
├── main.py                     # 程序主入口：刷步数核心逻辑
├── inspect_configs.py          # 辅助工具：从 Secret 中提取/加密配置信息
├── cron_convert.sh             # Shell 脚本：随机化 cron 表达式并记录执行日志
├── cron_change_time            # 运行时生成：记录最近一次 cron 变更信息
├── encrypted_tokens.data       # 运行时生成：AES 加密后的多账号 token 缓存
├── util/
│   ├── aes_help.py             # AES-128-CBC 加解密工具
│   ├── zepp_helper.py          # Zepp / 华米 API 封装（登录、Token、提交步数）
│   └── push_util.py            # 多渠道消息推送（PushPlus / 企业微信 / Telegram）
├── local/
│   └── decrypt_data.py         # 本地调试工具：解密 AES base64 内容
├── .github/workflows/
│   ├── run.yml                 # 主工作流：定时执行刷步数
│   ├── cron.yml                # 工作流：随机化 cron 并提交日志
│   ├── inspect_configs.yml     # 工作流：手动触发提取配置信息
│   └── star.yml                # 工作流：被 star 时打印日志
├── README.md                   # 用户使用说明
├── LICENSE
├── .gitignore
└── .gitattributes
```

### 2.2 分层架构

项目采用 **脚本化、职责分离** 的轻量级分层结构，无框架依赖：

```text
┌────────────────────────────────────────────────────────────┐
│  GitHub Actions (调度层)                                    │
│  - run.yml 定时触发 main.py                                  │
│  - cron.yml 随机化 cron 表达式                               │
│  - inspect_configs.yml 触发配置提取                          │
└─────────────────────────────┬──────────────────────────────┘
                              │ 通过环境变量注入 Secrets
┌─────────────────────────────▼──────────────────────────────┐
│  业务编排层 (main.py)                                       │
│  - 读取 CONFIG/AES_KEY                                      │
│  - 多账号循环 / 并发执行                                     │
│  - 持久化 token 缓存                                         │
│  - 调用推送模块                                              │
└─────────────────────────────┬──────────────────────────────┘
                              │
       ┌──────────────────────┼──────────────────────┐
       ▼                      ▼                      ▼
┌──────────────┐     ┌─────────────────┐    ┌──────────────────┐
│ util/        │     │ util/           │    │ util/            │
│ zepp_helper  │     │ aes_help        │    │ push_util        │
│ 华米 API 交互 │     │ 加解密工具       │    │ 消息推送          │
└──────────────┘     └─────────────────┘    └──────────────────┘
```

### 2.3 核心执行流程

```text
触发执行 (cron / 手动)
   │
   ▼
main.py __main__
   │
   ├─ 1. 读取 AES_KEY → 解密 encrypted_tokens.data → user_tokens
   ├─ 2. 读取 CONFIG (JSON) → 初始化 push_config / 用户 / 步数范围
   ├─ 3. 根据当前北京时间计算时间线性比例 min_step/max_step
   │
   ▼
execute()
   │
   ├─ 按账号 # 分割用户名与密码
   ├─ use_concurrent ? 并发(ThreadPool) : 串行(sleep_seconds)
   │       │
   │       └─► run_single_account()
   │               │
   │               └─► MiMotionRunner.login_and_post_step()
   │                       │
   │                       ├─ login(): 三级 Token 兜底链
   │                       │    1) 缓存 app_token 有效 → 直接使用
   │                       │    2) app_token 失效 → login_token grant 新 app_token
   │                       │    3) login_token 失效 → access_token grant 新 login/app/userid
   │                       │    4) 全失效 → 账号密码重新 login_access_token
   │                       │
   │                       └─ zeppHelper.post_fake_brand_data(step, ...)
   │
   ├─ 4. 持久化新 token 到 encrypted_tokens.data (AES + 随机 IV)
   └─ 5. push_util.push_results() 多渠道推送结果
   │
   ▼
workflow run.yml 末尾：git commit + push 更新 encrypted_tokens.data
   │
   ▼
workflow cron.yml 触发：调用 cron_convert.sh 随机化 cron 分钟值并提交日志
```

## 三、模块职责说明

### 3.1 `main.py` — 业务编排层

主入口与执行编排，负责参数初始化、多账号调度、Token 缓存与推送触发。

**职责**：
- 解析 `CONFIG`（JSON 字符串）和 `AES_KEY` 环境变量。
- 加载 / 持久化加密的 token 缓存文件 `encrypted_tokens.data`。
- 根据当前北京时间计算线性步数范围。
- 串行 / 并发执行多账号刷步数。
- 收集结果并触发推送。

### 3.2 `util/zepp_helper.py` — 华米 / Zepp API 封装

封装与华米（华米科技 = 小米运动后端）服务器的全部 HTTP 交互。

**职责**：
- `login_access_token`：用账号密码换取 `access_token`（请求体 AES 加密）。
- `grant_login_tokens`：用 `access_token` 换取 `login_token` / `app_token` / `user_id`。
- `grant_app_token`：用 `login_token` 单独刷新 `app_token`。
- `check_app_token`：通过 `getUserInfo` 接口验证 `app_token` 是否有效。
- `renew_login_token`：刷新 `login_token`（脚本中暂未调用，预留接口）。
- `post_fake_brand_data`：构造伪手环数据包，提交步数到 `band_data.json`。

### 3.3 `util/aes_help.py` — 加解密工具

封装 AES-128-CBC 加解密及 PKCS7 填充、Base64 转换。

**双重用途**：
- 内置华米固定密钥 `HM_AES_KEY` / `HM_AES_IV`，用于华米登录接口的请求体加密（参考自 [hanximeng/Zepp_API](https://github.com/hanximeng/Zepp_API)）。
- 提供通用 `encrypt_data` / `decrypt_data`，当 `iv=None` 时使用随机 IV 并把 IV 拼在密文前（用于本地加密 token 缓存）。

### 3.4 `util/push_util.py` — 消息推送

统一封装三类推送渠道，并提供整点过滤机制。

**职责**：
- `PushConfig`：聚合所有推送相关配置。
- `push_plus`：调用 [pushplus](http://www.pushplus.plus/send) 推送 HTML 内容到微信。
- `push_wechat_webhook`：调用企业微信机器人 Webhook（markdown_v2 格式）。
- `push_telegram_bot`：调用 Telegram Bot API 发送 HTML 消息。
- `not_in_push_time_range`：根据 `PUSH_PLUS_HOUR` 与 `cron_change_time` 文件判断当前是否应推送，避免 Actions 延迟导致漏推。
- `push_results`：汇总所有账号结果，按配置分发到各渠道。

### 3.5 `inspect_configs.py` — 配置提取工具

用于遗忘配置时恢复 `CONFIG` / `PAT` / `AES_KEY`。

**职责**：
- 读取 `INSPECT_AES_KEY` → 用其作为 key 和 iv 加密配置后 base64 输出到日志（用户可在线 AES 解密）。
- 读取 `INSPECT_WECHAT_HOOK_KEY` / `INSPECT_TELEGRAM_BOT_TOKEN` + `INSPECT_TELEGRAM_CHAT_ID` → 把配置明文推送到企业微信或 Telegram。

### 3.6 `local/decrypt_data.py` — 本地调试工具

独立小脚本，用于本地解密一段 base64 + AES 加密内容。开发者修改其中的 `encrypted_data` 与 `aes_key` 即可运行。

### 3.7 `cron_convert.sh` — cron 随机化脚本

被 `cron.yml` 工作流 `source` 引用，提供以下函数：

| 函数                       | 作用                                                                  |
|----------------------------|-----------------------------------------------------------------------|
| `inspect_hours`            | 从 cron 字符串中提取小时字段。                                         |
| `inspect_next`             | 计算下一次执行时间并打印 UTC/北京时间。                                |
| `hours_except_now`         | 从 `CRON_HOURS` 中剔除当前小时，避免同小时内二次执行造成步数混乱。     |
| `convert_utc_to_shanghai`  | 将 UTC cron 转为北京时间 cron 输出（仅用于日志展示）。                 |
| `persist_execute_log`      | 将本次执行信息写入 `cron_change_time`，并用随机分钟值更新 `run.yml`。 |

### 3.8 GitHub Actions 工作流

| 工作流文件               | 名称              | 触发方式                                              | 作用                                                              |
|--------------------------|-------------------|-------------------------------------------------------|-------------------------------------------------------------------|
| `run.yml`                | 刷步数            | `schedule` (cron) + `workflow_dispatch`               | 主流程：安装依赖 → 运行 `main.py` → 提交 token 缓存回仓库。       |
| `cron.yml`               | Random Cron       | `workflow_run` (run.yml 完成后) + `workflow_dispatch` | 调用 `cron_convert.sh` 随机化 `run.yml` 中的 cron 分钟并提交日志。 |
| `inspect_configs.yml`    | 提取配置信息      | `workflow_dispatch` (仅手动)                          | 运行 `inspect_configs.py` 输出 / 推送配置明文或密文。             |
| `star.yml`               | star watcher      | `watch` (被 star)                                     | 打印一行日志，仅作演示。                                          |

## 四、关键类与函数说明

### 4.1 `main.py`

#### 类 `MiMotionRunner`

单账号执行器，封装一个账号的登录与提交逻辑。

| 成员                  | 类型     | 说明                                                                                       |
|-----------------------|----------|--------------------------------------------------------------------------------------------|
| `__init__`            | 方法     | 初始化账号、密码、device_id（UUID）。自动识别邮箱或手机号并补 `+86` 前缀。设置 `invalid` 标记。 |
| `login`               | 方法     | **核心方法**。按 app_token → login_token → access_token 三级回退链获取可用 `app_token`，并把更新写回 `user_tokens` 全局缓存。 |
| `login_and_post_step` | 方法     | 登录后生成随机步数并调用 `post_fake_brand_data` 提交，返回 `(消息, 是否成功)`。             |

#### 模块级函数

| 函数                                       | 说明                                                                                      |
|--------------------------------------------|-------------------------------------------------------------------------------------------|
| `get_int_value_default(_config, key, default)` | 从 dict 取值并转 int，不存在则填默认值。                                                |
| `get_min_max_by_time(hour, minute)`        | 根据当前时间占全天 0~22 点的比例，线性计算 `min_step` / `max_step` 的实际生效值。         |
| `fake_ip()`                                | 生成 `223.x.x.x` 段的随机国内 IP（当前代码中已注释未启用）。                              |
| `desensitize_user_name(user)`              | 账号脱敏：保留首尾若干字符，中间用 `*` 替换。                                            |
| `get_beijing_time()` / `format_now()` / `get_time()` | 获取北京时间、格式化字符串、毫秒时间戳。                                          |
| `get_access_token(location)` / `get_error_code(location)` | 从登录重定向 Location 中正则提取 `access=` / `error=` 字段。              |
| `run_single_account(total, idx, user, pwd)` | 单账号执行包装：构造日志、捕获异常、返回结果 dict。                                       |
| `execute()`                                | **主流程**：分割账号、串行/并发执行、汇总结果、持久化 token、触发推送。                   |
| `prepare_user_tokens()`                    | 从 `encrypted_tokens.data` 解密并加载已缓存的 token。                                     |
| `persist_user_tokens()`                    | 将 `user_tokens` 加密写回 `encrypted_tokens.data`。                                       |

#### 全局变量（`__main__` 块内）

| 变量              | 来源                          | 说明                                       |
|-------------------|-------------------------------|--------------------------------------------|
| `time_bj`         | `get_beijing_time()`          | 程序启动时的北京时间。                      |
| `encrypt_support` | `AES_KEY` 是否合法            | 是否启用 token 加密持久化。                |
| `user_tokens`     | `prepare_user_tokens()`       | 全局多账号 token 缓存 dict。               |
| `config`          | `os.environ["CONFIG"]` (JSON) | 用户配置字典。                            |
| `push_config`     | `push_util.PushConfig(...)`   | 推送配置对象。                            |
| `users` / `passwords` | `config["USER"]` / `config["PWD"]` | `#` 分隔的多账号字符串。        |
| `min_step` / `max_step` | `get_min_max_by_time()`   | 当前时间生效的步数区间。                  |
| `use_concurrent`  | `config["USE_CONCURRENT"]`    | 是否启用多线程执行。                       |
| `sleep_seconds`   | `config["SLEEP_GAP"]`         | 串行模式下账号间间隔秒数（默认 5）。       |

### 4.2 `util/zepp_helper.py`

| 函数                                      | 入参                                            | 出参                                          | 说明                                                                            |
|-------------------------------------------|-------------------------------------------------|-----------------------------------------------|---------------------------------------------------------------------------------|
| `login_access_token(user, password)`      | 账号、密码                                       | `(access_token, error_msg)`                   | 向 `api-user.zepp.com/v2/registrations/tokens` POST 加密后的表单，从 303 重定向 Location 中提取 `access_token`。 |
| `get_access_token(location)`              | 重定向 URL                                       | `str \| None`                                 | 正则提取 `access=` 字段。                                                       |
| `get_error_code(location)`                | 重定向 URL                                       | `str \| None`                                 | 正则提取 `error=` 字段。                                                        |
| `grant_login_tokens(access_token, device_id, is_phone)` | access_token、device_id、是否手机号    | `(login_token, app_token, user_id, error_msg)` | 调用 `account.huami.com/v2/client/login`，区分邮箱 / 手机两种 `third_name`。    |
| `grant_app_token(login_token)`            | login_token                                     | `(app_token, error_msg)`                      | 调用 `account-cn.huami.com/v1/client/app_tokens` 单独刷新 app_token。           |
| `check_app_token(app_token)`              | app_token                                       | `(ok: bool, error_msg)`                       | 调用 `getUserInfo.json` 验证 app_token 有效性。                                 |
| `renew_login_token(login_token)`          | login_token                                     | `(login_token, error_msg)`                    | 调用 `renew_login_token` 接口刷新 login_token（当前未在主流程调用）。            |
| `post_fake_brand_data(step, app_token, userid)` | 步数、app_token、userid                  | `(ok: bool, message)`                         | 构造伪手环数据包（含心跳、步数分段、summary 等），通过正则替换 date 与 step 后 POST 到 `band_data.json`。 |

### 4.3 `util/aes_help.py`

| 函数 / 常量                          | 说明                                                                                    |
|--------------------------------------|-----------------------------------------------------------------------------------------|
| `HM_AES_KEY` / `HM_AES_IV`           | 华米接口固定密钥与 IV（各 16 字节）。                                                    |
| `AES_BLOCK_SIZE`                     | AES 块大小（16）。                                                                       |
| `_pkcs7_pad(data)` / `_pkcs7_unpad(data)` | PKCS7 填充与去填充，带校验。                                                       |
| `_validate_key(key)`                 | 校验 key 必须为 16 字节 bytes。                                                          |
| `encrypt_data(plain, key, iv=None)`  | AES-128-CBC 加密。`iv=None` 时生成随机 IV 并拼在密文前；传入 iv 时使用固定 IV。           |
| `decrypt_data(data, key, iv=None)`   | 对应解密。`iv=None` 时从数据前 16 字节提取 IV。                                          |
| `bytes_to_base64(data)` / `base64_to_bytes(data)` | bytes ↔ base64 字符串转换。                                                |

### 4.4 `util/push_util.py`

#### 类 `PushConfig`

聚合所有推送相关字段：`push_plus_token`、`push_plus_hour`、`push_plus_max`、`push_wechat_webhook_key`、`telegram_bot_token`、`telegram_chat_id`。

#### 模块级函数

| 函数                                            | 说明                                                                                    |
|-------------------------------------------------|-----------------------------------------------------------------------------------------|
| `push_plus(token, title, content)`              | 调用 pushplus `/send` 推送 HTML 内容。                                                  |
| `push_wechat_webhook(key, title, content)`      | 调用企业微信机器人 webhook，使用 `markdown_v2` 消息类型。                                |
| `push_telegram_bot(bot_token, chat_id, content)`| 调用 Telegram Bot `sendMessage`，HTML 解析模式。                                        |
| `buildWeChatContent(title, content)`            | 组装企业微信 markdown_v2 文本。                                                          |
| `not_in_push_time_range(config)`                | **推送整点过滤**。先比较当前北京时间小时数；不匹配时再读 `cron_change_time` 文件中 `next exec time` 的小时数兜底。 |
| `push_results(exec_results, summary, config)`   | 总入口：先过滤时间，再依次分发到 PushPlus / 企业微信 / Telegram。                       |
| `push_to_push_plus(...)` / `push_to_wechat_webhook(...)` / `push_to_telegram_bot(...)` | 各渠道结果组装与发送。账号数 ≥ `push_plus_max` 时只推送概要。 |

### 4.5 `inspect_configs.py`

| 函数                                              | 说明                                                                  |
|---------------------------------------------------|-----------------------------------------------------------------------|
| `build_inspect_configs_content(...)`              | 组装 Markdown 格式（带代码块）的配置明文，用于企业微信推送。          |
| `build_inspect_configs_content_for_telegram(...)` | 组装 Telegram HTML 格式（`<pre>` 标签）的配置明文。                    |
| `display_content_by_aes(...)`                     | 用 `INSPECT_AES_KEY` 同时作为 key 与 iv 加密三项配置，输出 base64。    |
| `display_encrypted_info(desc, content, key)`      | 单条加密 + base64 打印。                                              |

### 4.6 `cron_convert.sh`

| 函数                       | 说明                                                                                  |
|----------------------------|---------------------------------------------------------------------------------------|
| `inspect_hours`            | 提取 cron 字符串第二字段（小时）。                                                    |
| `inspect_next`             | 根据当前 UTC 时间与 cron 分钟，计算下一次触发小时，输出 `next exec time: UTC(...) 北京时间(...)`。 |
| `hours_except_now`         | 从逗号分隔的小时列表中剔除当前 UTC 小时，避免同小时二次执行。                          |
| `convert_utc_to_shanghai`  | 将 UTC cron 转换为北京时间 cron 字符串（仅日志展示用）。                                |
| `persist_execute_log`      | **主入口函数**。写 `cron_change_time` 日志、读取 `CRON_HOURS` 变量、调用 `sed` 把 `run.yml` 中 cron 的分钟替换为 `$((RANDOM % 59))`。 |

## 五、依赖关系

### 5.1 模块间依赖

```text
main.py
  ├── util.aes_help        (encrypt_data, decrypt_data)
  ├── util.zepp_helper     (login_access_token, grant_login_tokens, grant_app_token, check_app_token, post_fake_brand_data)
  └── util.push_util       (PushConfig, push_results)

util.zepp_helper
  └── util.aes_help        (encrypt_data, HM_AES_KEY, HM_AES_IV)

util.push_util
  └── (标准库 + requests)

inspect_configs.py
  ├── util.aes_help        (encrypt_data, bytes_to_base64)
  └── util.push_util       (push_wechat_webhook, push_telegram_bot)

local/decrypt_data.py
  └── util.aes_help        (decrypt_data, base64_to_bytes)
```

### 5.2 第三方 Python 依赖

由 [run.yml](file:///workspace/.github/workflows/run.yml) 与 [inspect_configs.yml](file:///workspace/.github/workflows/inspect_configs.yml) 中的 `pip3 install` 指定：

| 包名           | 用途                                           |
|----------------|------------------------------------------------|
| `requests`     | HTTP 请求（华米 API、推送接口）                 |
| `pytz`         | 时区处理（北京时间 `Asia/Shanghai`）            |
| `pycryptodome` | AES 加解密（`Crypto.Cipher.AES`、`Crypto.Random`）|

### 5.3 标准库依赖

`json`、`re`、`time`、`os`、`math`、`uuid`、`traceback`、`urllib.parse`、`datetime`、`base64`、`concurrent.futures`（可选并发）。

### 5.4 外部服务依赖

| 服务                 | 接口 / 域名                                                       | 用途                              |
|----------------------|-------------------------------------------------------------------|-----------------------------------|
| Zepp / 华米用户服务  | `api-user.zepp.com/v2/registrations/tokens`                       | 账号密码登录获取 access_token     |
| 华米账号服务         | `account.huami.com/v2/client/login`                               | 换取 login_token / app_token      |
| 华米账号服务（CN）   | `account-cn.huami.com/v1/client/app_tokens`                       | 刷新 app_token                    |
| Zepp 用户信息接口    | `api-mifit-cn3.zepp.com/huami.health.getUserInfo.json`            | 校验 app_token                    |
| 华米数据接口         | `api-mifit-cn.huami.com/v1/data/band_data.json`                   | 提交伪手环步数数据                |
| PushPlus             | `http://www.pushplus.plus/send`                                   | 微信推送                          |
| 企业微信 Webhook     | `https://qyapi.weixin.qq.com/cgi-bin/webhook/send`               | 企业微信推送                      |
| Telegram Bot API     | `https://api.telegram.org/bot{token}/sendMessage`                | Telegram 推送                     |
| GitHub Actions       | `actions/checkout@v5`、`actions/setup-python@v6`                 | CI 调度与运行环境                 |

## 六、配置与运行方式

### 6.1 必需的 GitHub Secrets

| Secret 名称                    | 必需 | 说明                                                                  |
|--------------------------------|------|-----------------------------------------------------------------------|
| `CONFIG`                       | 是   | JSON 字符串，包含账号、密码、步数范围、推送配置等（见下表）。          |
| `AES_KEY`                      | 否   | 16 字符 AES 密钥，用于本地加密缓存 token。不配置则不持久化 token。    |
| `PAT`                          | 是   | GitHub Personal Access Token，用于工作流回写 `encrypted_tokens.data` 与 `run.yml`。 |
| `INSPECT_WECHAT_HOOK_KEY`      | 否   | 提取配置时使用的企业微信 webhook key。                                |
| `INSPECT_TELEGRAM_BOT_TOKEN`   | 否   | 提取配置时使用的 Telegram bot token。                                 |
| `INSPECT_TELEGRAM_CHAT_ID`     | 否   | 提取配置时使用的 Telegram chat id。                                   |
| `INSPECT_AES_KEY`              | 否   | 提取配置时用于加密输出的 AES key（16 字符），同时作为 key 与 iv。     |

### 6.2 可选的 GitHub Variables

| Variable 名称 | 说明                                                                                  |
|---------------|---------------------------------------------------------------------------------------|
| `CRON_HOURS`  | 逗号分隔的 UTC 小时数，如 `0,2,4,6,8,14`。设置后 `cron.yml` 会用它覆盖 `run.yml` 中的小时字段，并剔除当前小时避免重复执行。 |

### 6.3 `CONFIG` 字段说明

```json
{
  "USER": "abcxxx@xx.com",
  "PWD": "password",
  "MIN_STEP": "18000",
  "MAX_STEP": "25000",
  "PUSH_PLUS_TOKEN": "",
  "PUSH_PLUS_HOUR": "",
  "PUSH_PLUS_MAX": "30",
  "PUSH_WECHAT_WEBHOOK_KEY": "",
  "TELEGRAM_BOT_TOKEN": "",
  "TELEGRAM_CHAT_ID": "",
  "SLEEP_GAP": "5",
  "USE_CONCURRENT": "False"
}
```

| 字段                     | 说明                                                                                              |
|--------------------------|---------------------------------------------------------------------------------------------------|
| `USER`                   | 小米运动账号（手机号或邮箱），多账号用 `#` 分隔。不支持小米账号。                                  |
| `PWD`                    | 对应密码，多账号用 `#` 分隔，数量须与 `USER` 匹配。                                                |
| `MIN_STEP` / `MAX_STEP`  | 22 点时的步数上下限，其他时间按时间线性比例缩放。                                                  |
| `PUSH_PLUS_TOKEN`        | PushPlus 个人 token。                                                                              |
| `PUSH_PLUS_HOUR`         | 限制只在某个北京整点推送，整数。不填或非数字则每次都推。                                            |
| `PUSH_PLUS_MAX`          | 推送详情账号数上限，默认 30，超过则只推概要。                                                       |
| `PUSH_WECHAT_WEBHOOK_KEY`| 企业微信机器人 key。                                                                               |
| `TELEGRAM_BOT_TOKEN`     | Telegram bot token（需同时配置 chat_id）。                                                         |
| `TELEGRAM_CHAT_ID`       | Telegram chat id。                                                                                 |
| `SLEEP_GAP`              | 串行模式下账号间隔秒数，默认 5。                                                                   |
| `USE_CONCURRENT`         | 是否启用多线程，`"True"` 启用（实验性），启用后 `SLEEP_GAP` 失效。                                 |

### 6.4 运行方式

#### 方式一：GitHub Actions（推荐）

1. Fork 仓库。
2. 按第 6.1 节配置 Secrets（最少需要 `PAT` 与 `CONFIG`）。
3. （可选）按第 6.2 节配置 `CRON_HOURS` 变量。
4. 在 Actions 页面启用工作流，左侧选择 `刷步数`，可手动 `Run workflow` 测试。
5. 默认每天按 [run.yml](file:///workspace/.github/workflows/run.yml) 中 cron 定时执行 6+ 次，每次执行后 `cron.yml` 会随机化下一次的分钟值。

#### 方式二：本地运行

```bash
# 安装依赖
pip3 install requests pytz pycryptodome

# 设置环境变量（Linux/macOS）
export CONFIG='{"USER":"your_account","PWD":"your_password","MIN_STEP":"18000","MAX_STEP":"25000"}'
export AES_KEY='1234567890abcdef'   # 可选，16 字符

# 运行
python3 main.py
```

> 注意：本地运行不会触发 `cron.yml` 与推送整点过滤中的 `cron_change_time` 文件读取逻辑（文件不存在时会跳过）。

#### 方式三：提取遗忘的配置

1. 配置 `INSPECT_WECHAT_HOOK_KEY` 或 `INSPECT_TELEGRAM_*` 或 `INSPECT_AES_KEY` 中至少一项。
2. 在 Actions 页面手动触发 `提取配置信息` 工作流。
3. 通过企业微信 / Telegram 收到明文，或从日志复制 base64 后用在线 AES 工具解密（CBC / PKCS7 / 128bit，key 与 iv 均为 `INSPECT_AES_KEY`）。

### 6.5 运行时产物

| 文件                       | 生成位置                 | 是否提交回仓库 | 说明                                              |
|----------------------------|--------------------------|----------------|---------------------------------------------------|
| `encrypted_tokens.data`    | `main.py` 运行时         | 是（由 `run.yml` 提交） | AES 加密的多账号 token 缓存。                  |
| `cron_change_time`         | `cron_convert.sh` 运行时 | 是（由 `cron.yml` 提交） | cron 变更日志，被 `push_util.not_in_push_time_range` 读取。 |
| `run.yml` 中的 cron 表达式 | `cron_convert.sh` 修改   | 是（由 `cron.yml` 提交） | 每次随机化分钟值，可选覆盖小时值。              |

## 七、关键设计说明

### 7.1 Token 三级回退链（核心反风控设计）

为减少账号密码登录次数、降低被风控概率，[main.py](file:///workspace/main.py) 中 `MiMotionRunner.login` 实现了三级 token 兜底：

1. **app_token 优先**：缓存中若存在 `app_token`，先调用 `zeppHelper.check_app_token` 验证。有效则直接使用。
2. **login_token 兜底**：app_token 失效时，用缓存的 `login_token` 调用 `grant_app_token` 刷新。
3. **access_token 兜底**：login_token 也失效时，用缓存的 `access_token` 调用 `grant_login_tokens` 重新换取全套 token。
4. **账号密码兜底**：以上全失效或无缓存时，才调用 `login_access_token` 走账号密码登录。

每次成功获取任一级 token 都会更新 `user_tokens` 全局缓存与对应的时间戳，便于排查失效原因。

### 7.2 步数线性增长

[main.py](file:///workspace/main.py) 中 `get_min_max_by_time`：

```python
time_rate = min((hour * 60 + minute) / (22 * 60), 1)
min_step = int(time_rate * MIN_STEP)
max_step = int(time_rate * MAX_STEP)
```

- 以北京时间 22 点为满值基准。
- 例如 10 点执行：`time_rate = 10/22 ≈ 0.45`，若 `MIN_STEP=18000`、`MAX_STEP=25000`，则实际随机区间约为 `8181 ~ 11363`。
- 22 点之后 `time_rate` 被 `min(..., 1)` 截断为 1，避免超过配置上限。

### 7.3 Token 缓存加密

- 仅当 `AES_KEY` 为合法 16 字符时启用（[main.py](file:///workspace/main.py) `__main__` 中校验）。
- 加密使用 `encrypt_data(plain, key, iv=None)`，即 **随机 IV + 密文** 模式，IV 拼接在密文前 16 字节。
- 解密时 `decrypt_data` 自动从头部提取 IV。
- 缓存文件 `encrypted_tokens.data` 通过 `run.yml` 末尾的 git 提交流程回写仓库，下次执行可直接复用。
- **注意**：Fork 后首次配置自己的 `AES_KEY` 时，原仓库的加密文件会解密失败并打印 "密钥不正确或者加密内容损坏 放弃token"，属正常现象，运行完成后会生成新密钥对应的文件。

### 7.4 随机化 cron

由 [cron_convert.sh](file:///workspace/cron_convert.sh) 实现，被 [cron.yml](file:///workspace/.github/workflows/cron.yml) `source` 调用：

- 每次主工作流执行成功后，`cron.yml` 通过 `workflow_run` 触发。
- `persist_execute_log` 函数用 `sed -E` 把 `run.yml` 中 `cron: 'XX ...'` 的分钟部分替换为 `$((RANDOM % 59))`。
- 若配置了 `CRON_HOURS` 变量，则同时把小时部分替换为 `CRON_HOURS` 剔除当前小时后的值（`hours_except_now`），避免同小时二次执行导致步数异常。
- 同时写入 `cron_change_time` 文件，记录触发方式、当前时间、变更前后 cron、下次执行时间。

### 7.5 推送整点过滤

[push_util.py](file:///workspace/util/push_util.py) 中 `not_in_push_time_range`：

1. 若未设置 `PUSH_PLUS_HOUR`，总是推送。
2. 若当前北京整点 == `PUSH_PLUS_HOUR`，推送。
3. 否则读取 `cron_change_time` 最后一行 `next exec time: UTC(x:y) 北京时间(z:y)`，提取其中的北京小时数。若与 `PUSH_PLUS_HOUR` 相等，也推送（兜底 Actions 延迟场景）。
4. 都不匹配则跳过本次推送。

### 7.6 多账号并发

- 默认串行，账号间 `time.sleep(sleep_seconds)`，避免接口过于频繁触发 429。
- `USE_CONCURRENT="True"` 时启用 `concurrent.futures.ThreadPoolExecutor`，`SLEEP_GAP` 失效。
- README 中标注为实验性功能，未充分测试，多账号场景需自行评估华米接口的限流策略。

## 八、注意事项与已知限制

1. **账号类型**：必须是 `Zepp Life / 小米运动` 账号（手机号或邮箱注册），**不是** 小米账号。
2. **接口限流**：华米新版本接口对同 IP 多账号登录可能返回 429，多账号场景建议串行 + 合理 `SLEEP_GAP`。
3. **同步范围**：小米运动本身不显示步数，只同步到关联的第三方（支付宝、微信运动等）。若第三方未更新，需在 Zepp Life 中注销账号、清空数据、重新登录并绑定第三方。
4. **cron 时区**：GitHub Actions cron 使用 UTC，北京时间需 -8。例如要 8/10/12/14/16/22 点执行，cron 应为 `0 0,2,4,6,8,14 * * *`。
5. **Actions 排队**：GitHub Actions 在整点高峰可能延迟 1~2 小时执行，建议从 2 点开始而非 0 点。
6. **加密文件备份**：配置 `AES_KEY` 后，每次同步上游代码会覆盖 `encrypted_tokens.data`，更新前需手动备份再恢复。
7. **PAT 权限**：Fine-grained token 需勾选 `Actions` / `Contents` / `Metadata` / `Workflows` 四项权限，否则 `cron.yml` 与 token 持久化会失败。
8. **`renew_login_token`**：[zepp_helper.py](file:///workspace/util/zepp_helper.py) 中定义了该接口但主流程未调用，作为预留扩展。
9. **`fake_ip`**：[main.py](file:///workspace/main.py) 中定义了虚拟 IP 生成函数但已注释未启用，预留扩展。

## 九、扩展与二次开发指引

- **新增推送渠道**：在 [push_util.py](file:///workspace/util/push_util.py) 中实现类似 `push_to_xxx` 函数，并在 `PushConfig` 与 `push_results` 中接入；同步在 `CONFIG` 中新增字段、在 [README.md](file:///workspace/README.md) 配置表中补充说明。
- **新增华米接口**：在 [zepp_helper.py](file:///workspace/util/zepp_helper.py) 中按现有模式（固定 headers + requests + json 解析）新增函数。
- **更换加密算法**：[aes_help.py](file:///workspace/util/aes_help.py) 中 `encrypt_data` / `decrypt_data` 是 token 缓存的唯一加解密入口，替换算法时需保持文件格式兼容或提供迁移逻辑；华米登录加密用的 `HM_AES_KEY` / `HM_AES_IV` 不可改动，否则登录失败。
- **调整步数曲线**：修改 `get_min_max_by_time` 中的 `time_rate` 公式（如改为非线性、调整满值时间点 22）。
- **本地调试加解密**：修改 [local/decrypt_data.py](file:///workspace/local/decrypt_data.py) 中的 `encrypted_data` 与 `aes_key` 后直接运行。

## 十、参考致谢

- 原始项目：`xunichanghuan/mimotion`（已不可用）、[huangshihai/mimotion](https://github.com/huangshihai/mimotion)
- 登录加密密钥来源：[hanximeng/Zepp_API](https://github.com/hanximeng/Zepp_API/blob/main/index.php)
- 详细使用文档：[README.md](file:///workspace/README.md)
