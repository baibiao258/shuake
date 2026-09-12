<p align="center">
  <a href="https://www.springing.top" target="blank">
    <img src="images/logo.png" alt="Logo" width="156" height="156">
  </a>
  <h2 align="center" style="font-weight: 600">Spring-Superstar</h2>
  <p align="center">
    学习通在线刷课脚本 · 超星 · 云端刷课
  </p>
</p>

本项目宗旨是帮助大学生们解放双手，根据教程操作后就可以完成刷课。支持**本地运行**和 **GitHub Actions 云端自动化**两种方式，账号密码、题库等敏感信息均可通过 **Actions Secrets** 注入，避免明文提交。

> 如果本项目对你有帮助，希望同学们给本仓库点一个免费的 Star 或给小春子点一个 Follow，谢谢大家！

![截图](/images/star.png)

---

## 📚 目录

- [功能特性](#功能特性)
- [⚠️ 用前必读（安全）](#️-用前必读安全)
- [项目结构](#项目结构)
- [快速开始（本地运行）](#快速开始本地运行)
- [配置详解](#配置详解)
- [题库支持](#题库支持)
- [GitHub Actions 云端刷课](#github-actions-云端刷课)
- [环境变量参考表](#环境变量参考表)
- [Docker 部署](#docker-部署)
- [常见问题](#常见问题)
- [免责声明](#免责声明)

---

## 功能特性

- 自动登录超星学习通（账号密码登录 / Cookie 登录）
- 自动完成多种任务点：**视频、音频、文档、章节测验、阅读、直播、签到**
- 视频按倍速播放、可多线程（多章节）同时推进
- **自动答题**：接入多种题库与大模型，支持多题库自动回退
- 支持 **GitHub Actions 云端 24h 定时刷课**（Secrets 安全注入）
- 可选推送通知（Server 酱 / Qmsg / Bark / Telegram）
- 题目答案本地缓存，命中直接复用

---

## ⚠️ 用前必读（安全）

1. **默认 fork 之后仓库是公开状态**，账号密码若直接写入代码/配置文件并提交，会处于**明文暴露**状态，非常危险！请务必：
   - 把仓库权限改为 **私有**；
   - 不要把 `config.ini`、`cookies.txt`、`chaoxing.log` 推送到远程（项目 `.gitignore` 已默认忽略）。
2. 本项目优先推荐使用 **GitHub Actions Secrets 或环境变量**注入账号密码与题库 token。
3. 之前曾发生过因明文账号密码泄露导致账号被盗的案例，请务必保护好自己的凭据。

---

## 项目结构

```
.
├── main.py                    # 主程序入口（刷课、答题、通知）
├── app.py                     # Flask + Celery 框架（预留/服务化扩展）
├── config_template.ini        # 配置文件模板（复制为 config.ini 使用）
├── requirements.txt           # Python 依赖
├── Dockerfile                 # Docker 镜像构建
├── api/
│   ├── base.py                # 超星 API 封装（登录/课程/任务/进度上报）
│   ├── answer.py              # 题库核心（Tiku 及所有题库适配器）
│   ├── answer_check.py        # 答案格式校验
│   ├── cipher.py              # AES 加密（登录参数加密）
│   ├── cookies.py             # Cookie 登录/保存
│   ├── captcha.py / decode.py # 字体解密、验证码
│   ├── live.py                # 直播任务
│   ├── notification.py        # 外部通知推送
│   ├── config.py              # 全局常量（头、AES 密钥等）
│   └── logger.py              # 日志
├── images/  resource/         # 图片与字体资源
└── .github/workflows/main.yml # GitHub Actions 刷课 workflow
```

---

## 快速开始（本地运行）

### 环境要求
- Python **3.11 / 3.12**（3.13 亦可，但 3.12 依赖 wheel 兼容性最好）
- pip

### 安装

```bash
pip install -r requirements.txt
```

### 配置

复制模板并填写配置：

```bash
cp config_template.ini config.ini
# Windows: copy config_template.ini config.ini
```

在 `config.ini` 中填写：

```ini
[common]
use_cookies=false
username = 你的手机号
password = 你的密码
course_list =            ; 课程ID，逗号隔开，留空则全部课程
```

> `config.ini`、`cookies.txt`、`chaoxing.log` 均已被 `.gitignore` 忽略，不会误提交。

### 运行

方式一：使用配置文件（推荐）

```bash
python main.py -c config.ini
```

方式二：命令行参数（账号密码会出现在 shell 命令中，注意安全）

```bash
python main.py -u "手机号" -p "密码" -l "课程ID1,课程ID2"
```

方式三：命令行 + 环境变量（账号密码从环境变量读取，适合 CI/脚本）

```bash
export CHAOXING_USERNAME="手机号"
export CHAOXING_PASSWORD="密码"
python main.py
```

### 常用启动参数

| 参数 | 说明 | 默认 |
|------|------|------|
| `-c/--config` | 使用配置文件 | 无 |
| `-u/--username` | 手机号账号 | 无 |
| `-p/--password` | 登录密码 | 无 |
| `-l/--list` | 课程ID列表，逗号隔开 | 无（全部课程） |
| `-s/--speed` | 视频倍速（1~2） | 1.0 |
| `-j/--jobs` | 同时进行的章节数 | 4 |
| `--use-cookies` | 使用 cookies 登录（从 cookies.txt） | 否 |
| `--auto-sign` | 自动签到 | 否 |
| `-a/--notopen-action` | 未开放任务处理：`retry/ask/continue` | retry |
| `-v/--debug` | 输出 DEBUG 日志 | 否 |

---

## 配置详解

### `[common]` 基础配置

| 键 | 说明 |
|----|------|
| `use_cookies` | `true` 则忽略账号密码，从 `cookies.txt` 读取 cookie 登录 |
| `username` / `password` | 学习通手机号账号/密码（也可用环境变量/命令行注入） |
| `course_list` | 要学习的课程ID列表，逗号隔开 |
| `speed` | 视频倍速（默认 1，最大 2） |
| `jobs` | 同时进行的章节数 |
| `notopen_action` | 遇到关闭任务点时的行为：`retry/continue` |

### `[tiku]` 题库配置

| 键 | 说明 |
|----|------|
| `provider` | 题库类型，逗号隔开可多题库回退，如 `TikuGo,TikuYanxi` |
| `tokens` | 题库 token，逗号隔开按顺序使用 |
| `url` | TikuAdapter / TikuSuper 的题库服务地址 |
| `submit` | `true` 提交答题；`false` 只保存搜到的题目不提交 |
| `cover_rate` | 最低题库覆盖率（0~1） |
| `delay` | 查询题目间隔（秒） |
| `endpoint/key/model` | AI 大模型配置 |
| `siliconflow_key/model/endpoint` | 硅基流动配置 |
| `go_authorization` 等 | GO 题库配置 |
| `likeapi_*` | LIKE 知识库配置 |

### `[notification]` 通知配置

`provider` 可选：`ServerChan` / `Qmsg` / `Bark` / `Telegram`；填入对应的推送 `url` 即可，完成时推送消息。

---

## 题库支持

通过 `provider` 指定，均支持**多题库按顺序回退**（前一个未命中自动换下一个）：

| provider | 说明 |
|----------|------|
| `TikuYanxi` | 言溪题库 https://tk.enncy.cn/ |
| `TikuLike` | LIKE 知识库 https://www.datam.site/ |
| `TikuGo` | GO题/网课小工具题库 https://q.icodef.com/ |
| `TikuAdapter` | 开源 tikuAdapter https://github.com/DokiDoki1103/tikuAdapter |
| `AI` | 自建 OpenAI 兼容大模型接口 |
| `SiliconFlow` | 硅基流动 AI：https://siliconflow.cn/ |
| `TikuSuper` | **Super 题库**（对接 SuperAutoStudy 的 TiKu 服务，默认 `http://tk.xxtmooc.com/api/q`，在 `url` 填写地址） |

示例：

```ini
[tiku]
provider=TikuGo,TikuYanxi,TikuLike   ; 多题库回退
provider=TikuSuper                    ; 只用 Super 题库
url=http://tk.xxtmooc.com/api/q
submit=false
cover_rate=0.9
delay=1.0
```

> 若题库未配置或不可用，程序会自动禁用答题功能，但仍正常完成视频/文档等刷课任务，不会报错中断。

---

## GitHub Actions 云端刷课

无需本地运行，账号密码与题库凭据全部通过 **Secrets** 注入，**仓库代码中无任何明文敏感信息**。

### 1. 配置 Secrets

进入仓库 **Settings → Secrets and variables → Actions → New repository secret**，创建：

| 名称 | 说明 | 必填 |
|------|------|------|
| `CHAOXING_USERNAME` | 学习通手机号 | ✅ |
| `CHAOXING_PASSWORD` | 登录密码 | ✅ |
| `CHAOXING_COURSE_LIST` | 课程ID列表（逗号隔开），留空=全部 | |
| `TIKU_PROVIDER` | 题库类型，如 `TikuSuper` | |
| `TIKU_URL` | 题库服务地址，如 `http://tk.xxtmooc.com/api/q` | |
| `TIKU_TOKENS` / `TIKU_ENDPOINT` / `TIKU_KEY` / `TIKU_MODEL` | 相应题库/AI 配置 | |
| `SILICONFLOW_KEY` 等 | 硅基流动配置 | |

### 2. 触发运行

- **手动**：Actions 页面 → 选择「刷课」workflow → **Run workflow**。
- **自动**：向 `main` 分支 push（或取消 `main.yml` 中 `schedule` 的注释以定时刷课）。

> 若未配置账号 secrets，会得到清晰的"缺少账号密码"错误，而不会崩溃。

---

## 环境变量参考表

账号、题库、课程均可通过环境变量注入（与 Secrets 名称一致，便于 CI）：

| 环境变量 | 用途 |
|----------|------|
| `CHAOXING_USERNAME` / `CHAOXING_PASSWORD` / `CHAOXING_COURSE_LIST` | 账号/密码/课程列表 |
| `TIKU_PROVIDER` / `TIKU_TOKENS` / `TIKU_URL` | 题库类型/token/地址 |
| `TIKU_ENDPOINT` / `TIKU_KEY` / `TIKU_MODEL` | AI 题库配置 |
| `SILICONFLOW_KEY` / `SILICONFLOW_MODEL` / `SILICONFLOW_ENDPOINT` | 硅基流动配置 |
| `GO_AUTHORIZATION` | GO 题库 token |
| `LIKEAPI_MODEL` | LIKE 知识库模型 |

---

## Docker 部署

```bash
docker build -t chaoxing .
# 运行时挂载自己的 config.ini
docker run -d -v /你的路径/config.ini:/config/config.ini chaoxing
```

Dockerfile 使用 Python 3.13，入口为 `python main.py -c /config/config.ini`，镜像内置 `config_template.ini` 作默认配置。

---

## 常见问题

- **Actions 显示"缺少账号密码"**：未配置 `CHAOXING_USERNAME / CHAOXING_PASSWORD` secrets。
- **题库不生效**：检查 `provider` 拼写、`tokens`/`url` 是否正确；未配置时系统会自动禁用答题。
- **视频倍速超 2**：脚本强制限制 1~2。
- **账号被踢下线**：`forbidotherlogin` 默认 0，多端登录可能相互踢；可自行调整。

---

## 免责声明

本项目仅用于学习与个人辅助，请勿用于违反课程纪律的投机行为。使用本脚本产生的一切后果由使用者自行承担。由于学习通接口常有变动，脚本可能需按需维护更新。