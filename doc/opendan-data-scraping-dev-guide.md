# Building Data Scraping Skills for OpenDAN

> 为 OpenDAN Agent 开发数据抓取能力：第三方开发者指南

---

## 0. 为什么在 OpenDAN 上做，而不是自己写脚本？

你当然可以自己写一个 Python 爬虫跑在 crontab 里。但你大概已经发现了这些问题：

| 你遇到的问题 | 自己写脚本 | n8n / Make 等自动化平台 | OpenDAN |
|---|---|---|---|
| 页面结构变了，脚本挂了 | 手动修 selector，重新部署 | 同上，workflow 断掉需要人工介入 | Agent 有上下文理解能力，可以根据 Skill 里的判断标准自行调整抓取策略 |
| 需要过登录墙 / 验证码 | 自己集成 Playwright + 各种 hack | 平台提供的浏览器能力有限 | Agent 可以调用浏览器能力，像人一样操作页面 |
| 抓到的数据只有自己能用 | 数据格式随意，换个项目又得重新写 | 平台内部闭环，数据导出麻烦 | 统一 Schema，本地所有 Agent 共享同一份数据 |
| 想加 AI 处理（摘要/分类/理解） | 自己对接 API，管 token、管 key | 需要额外付费接 AI 节点 | AICC 统一模型调用，开发者不用管供应商差异 |
| 多个抓取任务需要协同 | 自己写调度逻辑 | 平台限制了组合方式 | 多个 Skill 天然可组合，Agent 按目标自主调度 |
| 想让别人也能用你的能力 | 发个 GitHub repo，用户自行部署 | 发布到平台 marketplace，但受平台规则约束 | 发布 Skill + Tool 即可，其他用户的 Agent 直接调用 |

**一句话：你写的不是一个跑一次就扔的脚本，而是一个可以被任何 Agent 反复使用的能力。**

传统爬虫是"写死的流程"——一旦目标网站变化，流程就断。
OpenDAN Skill 是"给 Agent 的操作说明"——Agent 根据说明自主决策，遇到异常可以自行调整。

这意味着你的工作成果有更长的生命周期，也有更大的复用面。

---

## 1. 你要做的三件事

为 OpenDAN Agent 提供数据抓取能力，只需要准备三样东西：

**Tool** — 真正执行抓取的程序。CLI 工具、脚本、小程序都行。

**Skill** — 告诉 Agent 怎么用这些 Tool 的操作说明。不是自动化脚本，是"目标 + 工具 + 判断标准"的组合。

**Schema** — 抓取结果的数据格式。保证不同开发者、不同 Agent 之间的数据可以互通。

---

## 2. 从 Tool 开始，先跑通一个最小抓取

### 选语言

推荐 **TypeScript / tsx**，Python 也可以。

### 写什么

你的 Tool 就是一个普通 CLI 程序，做的事情和你过去写爬虫一样：

- 调 API
- 跑 Playwright / 无头浏览器
- 解析页面
- 输出结构化 JSON

### 怎么发布

第一版完全不需要学习 OpenDAN / BuckyOS 的包管理体系。
你过去怎么发 npm 包，现在就怎么发：

- `npx`
- `pnpm`
- 或者就是本地脚本

等跑通以后，再考虑正式打包。

---

## 3. 写 Skill：给 Agent 的操作说明书

Skill 不是"严格一步不差的自动化脚本"。
它更像是你写给一个聪明但不了解具体业务的同事的操作指南。

一个好的 Skill 要写清楚以下内容：

### 目标

> 给定一个用户账号，抓取其公开资料、帖子、媒体内容。

或者：

> 给定一个商品链接，抓取其标题、价格、评论、历史价格。

### 可用工具

列出 Agent 可以调用的 Tool，例如：

- `fetch-x-profile` — 抓取 X 用户资料
- `fetch-x-posts` — 抓取 X 用户帖子
- `fetch-instagram-media` — 抓取 Instagram 媒体

### 判断标准

Agent 需要知道每一步"做对了"是什么样子：

- profile 拿到了且字段完整 → 第一步成功
- 帖子列表为空 → 需要判断是"真的没有"还是"抓取失败"
- 页面返回登录墙 → 切换到浏览器模式重试

### 常见错误和修正建议

- 账号不存在 → 停止，返回明确错误
- 页面结构变化 → 尝试用浏览器模式重新抓取
- 请求频率过高 → 降速重试
- 需要登录 → 提示用户提供凭据或切换策略

**核心原则：给方向、给工具、给判断方法。** Agent Loop 自身有纠错能力，Skill 的重点不是"零错误执行"，而是让 Agent 知道目标在哪、手里有什么、怎么判断是否在逼近目标。

---

## 4. 定义数据格式：目录结构就是数据库

抓取结果必须有统一结构，否则不同 Skill、不同 Agent 之间没法复用。

推荐用目录结构直接组织数据：

```
data/
  x/
    alice/
      profile.json          # 用户资料
      posts/
        post_001.json
        post_002.json
      media/
        img_001.jpg
  instagram/
    alice/
      profile.json
      posts/
      media/
  bindings/
    person_alice.json        # 跨平台身份绑定
```

**规则很简单：**

- `{platform}/{account_id}/` 是基本单元
- `profile.json` 存用户资料
- `posts/` 存帖子
- `media/` 存媒体文件
- `bindings/` 把同一个人在不同平台的账号关联起来

### 统一格式带来的两个直接好处

**多 Skill 协同补数据。** 一个 Skill 抓 profile，一个 Skill 抓 posts，一个 Skill 抓 comments——最终都落到同一个目录结构里，互不干扰。

**不同 Agent 复用结果。** Agent A 已经抓过的数据，Agent B 直接读取就行，不用重复抓。在 OpenDAN / BuckyOS 体系里，Agent 可以暴露服务接口，这意味着同一套 Skill 不只是"教 Agent 怎么抓"，还会自然形成**抓取结果的共享网络**。

---

## 5. 什么时候需要用 OpenDAN 的平台能力

不是所有场景都需要学额外的东西。按你的需求分三档：

### 情况 A：普通抓取 → 不需要任何平台能力

如果你的 Tool 只做 HTTP 请求、页面解析、Playwright 抓取、文件处理，就按普通程序写，不用学任何额外概念。

### 情况 B：Tool 内部需要 AI → 接入 AICC

如果你想在工具内部做语义解析、内容摘要、评论分类、图像理解，接入 AICC 即可。

AICC 的作用是统一模型调用。你给模型名、给 prompt、拿结果。不用自己折腾不同模型供应商的接入。

**只有 Tool 里需要 AI 时，才需要了解 AICC。**

### 情况 C：必须操作浏览器 / 桌面 → 使用 AgentPC

有些网站没有 API 或反爬很强，需要浏览器自动化：获取页面截图、点击、输入、滚动、观察结果、循环执行。

这时使用 OpenDAN 的 AgentPC / 浏览器能力。如果流程特别复杂，建议把浏览器操作任务交给专门的 Agent 处理，而不是全塞进一个 Tool。

---

## 6. 最小可用交付物

你的第一版只需要做到这样：

```
my-scraping-skill/
  skill.md                    # 操作说明书
  tools/
    fetch-x-profile.ts        # CLI 工具
    fetch-x-posts.ts
  schemas/
    profile.schema.json        # 数据结构定义
    post.schema.json
  data-example/                # 实际结果示例
    x/
      alice/
        profile.json
        posts/
          post_001.json
```

各部分职责：

- **`skill.md`** — 适用范围、输入参数、推荐工具、成功判断、常见错误、输出目录
- **`tools/`** — 你的 CLI 工具或脚本
- **`schemas/`** — 你约定的数据结构（JSON Schema）
- **`data-example/`** — 给其他开发者看的实际结果示例

---

## 7. 推荐上手顺序

不要一开始做"大而全"。按这个顺序最容易跑通：

1. **选一个网站** — 比如 X、Instagram、Amazon 其中一个
2. **先定义结果格式** — 想清楚 profile、post、media 怎么存
3. **写一个最小 Tool** — 先抓到最核心的一种数据
4. **写一个最小 Skill** — 把目标、工具、判断标准写清楚
5. **跑通一次完整流程** — 输入一个账号，输出一套目录
6. **再补充能力** — 评论、图片、视频、跨平台绑定

---

## 8. 总结

为 OpenDAN Agent 提供数据抓取能力，本质上就是：

> **写一个抓取工具，写一份 Agent 能看懂的使用说明，再把结果按统一格式存下来。**

三条核心原则：

- **Tool 负责执行** — 做具体的抓取动作
- **Skill 负责说明** — 告诉 Agent 怎么用、怎么判断
- **Schema 负责复用** — 保证结果能被任何 Agent 直接使用

和传统爬虫的本质区别在于：你写的不是一个只能自己用的脚本，而是一个**可以被整个 Agent 生态复用的能力模块**。
