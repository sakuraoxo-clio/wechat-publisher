# wechat-publisher

> 把一篇文章排版成微信公众号合规的精排 HTML，自动上传图片、可选生成封面，一键写进公众号草稿箱。
> A Claude skill that typesets articles into WeChat-ready HTML and pushes them straight to your draft box.

> 作者 / Author：**sakuraoxo（喂鱼）**

> **原始项目声明：** 本仓库是 **喂鱼（sakuraoxo）** 发布的 `wechat-publisher` 原始公开来源。任何 fork、二创、教程、公开展示或再分发版本，都必须保留原作者署名与原始仓库链接，不得暗示修改版是完全原创或原作者官方发布。署名信息见文末[「署名 / Attribution」](#署名--attribution)。
>
> **Original project notice:** This repository is the original public source of `wechat-publisher` by **喂鱼 (sakuraoxo)**. Forks, derivative works, tutorials, public showcases, and redistributed versions must retain the original attribution and the link to this repository, and must not imply that modified versions are original or official releases by the original author.

跟市面上「Markdown 一键转公众号」的工具不一样的地方在于：它不是套一个固定模板，而是让 **AI 照着一套组件库，为你这篇文章的内容现场手写排版**——卡片、编号小标题、深色金句框、还有**内联 SVG 信息图**（对比图、时间线、飞轮，矢量清晰且不占图片配额）。所以同样一篇稿子，出来的不是「文档感」，而是「设计过」。

---

## 效果预览

> 把你用它排出来的成品截图放这里（封面卡 / 一张 SVG 信息图 / callout 框 / 文末总结块各一张最直观）。
>
> 例如：
> `docs/preview-cover.png` · `docs/preview-svg.png` · `docs/preview-callout.png`

---

## 它能做什么

- **组件化精排版**：黑白灰高级感配色 + 一套现成组件（封面卡、编号小标题、callout 框、金句框、步骤卡、文末总结、标签 chips）。
- **内联 SVG 信息图**：数据可视化直接写进正文，微信原生支持，不占图片配额。
- **联网配图 → base64 内嵌**：需要真实照片/实拍图时让 Claude 联网搜图，`img2base64.ts` 把图直链下载、校验、压到 ≤1MB、转成 base64 内嵌进正文 —— 复制预览时随渲染内容一起进公众号，无需上传。
- **自动处理图片**：正文里的本地图 / 远程图 / base64 会自动上传到微信并替换链接。
- **可选自动生成封面**：调用图像模型生成封面图。
- **一键进草稿箱**：通过公众号官方 `draft/add` 接口写入，你在后台预览满意后再发。
- **两种排版模式**：组件化手写 HTML（精排，默认）/ Markdown 套固定主题（快速兜底）。

---

## 两种用法，按你的条件选

### 用法一：只排版，零门槛（推荐先从这里上手）

**不需要任何 API、不需要认证公众号。** 用它把文章排成 HTML，然后手动粘进公众号编辑器即可。适合所有人，先看到效果再说。

1. 准备一篇文章（`.md` 或纯文本）。
2. 让 Claude 照 `references/components.md` 把它排成组件化 HTML（见下方「配合 Claude 使用」）。
3. 在浏览器里打开生成的 HTML 预览效果（`references/article-template.html` 自带手机预览框）。
4. 满意后，把正文 HTML 复制粘贴进公众号后台编辑器。

### 用法二：一键推送到草稿箱（需要认证公众号）

适合已经有**已认证服务号 / 订阅号**、想省掉「复制粘贴 + 重新传图」这步的人。

**前提：**
- 公众号为**已认证**号，具备 `draft/add` 接口权限；
- 运行这台机器的公网 IP 已加入公众号后台「IP 白名单」。

> 不熟悉拿密钥 / 填白名单的，看详细图文教程 👉 https://sakuraoxo.feishu.cn/wiki/TuA7wvDjiiQZQVkGNp1cK6uynae

```bash
cd scripts
npm install
cp ../.env.example ../.env   # 然后填入凭证（见下）

# 发布一篇排好的 HTML
npx tsx publish.ts <文章.html> --title "标题" --author "作者" [--digest "摘要"] [--cover <封面图> | --gen-cover]

# 或者：喂一个 Markdown，套固定主题快速排版
npx tsx publish.ts <文章.md> --title "标题" --author "作者"
```

跑完会输出草稿 `media_id`，然后去 `https://mp.weixin.qq.com` → 内容管理 → 草稿箱 → 预览 → 发布。

---

## 环境变量

复制 `.env.example` 为 `.env` 填写。**`.env` 已被 `.gitignore` 忽略，绝对不要提交到仓库。**

| 变量 | 用途 | 哪里拿 |
|------|------|--------|
| `WECHAT_APP_ID` | 公众号 AppID | 公众号后台 → 开发 → 基本配置 |
| `WECHAT_APP_SECRET` | 公众号 AppSecret | 同上 |
| `OPENAI_API_KEY` | 仅 `--gen-cover` 生成封面时需要 | OpenAI 后台 |

---

## 配合 Claude 使用（精排版的关键）

这套东西的精排能力，是靠 **Claude 当排版执行者**实现的——你给内容，Claude 照组件库做设计。把整个文件夹作为一个技能（Skill）交给 Claude Code / Claude 桌面端，然后说一句：

> 帮我把这篇文章排成公众号 HTML，配 2 张 SVG 信息图，然后写进草稿箱。

Claude 会按 `SKILL.md` 的流程：读组件库 → 复制模板填充 → 配图 → 本地预览 → 调 `publish.ts` 发布。

你也可以完全手动：自己照 `references/components.md` 写 HTML，再单独跑 `publish.ts`。

---

## 文件结构

```
wechat-publisher/
├── SKILL.md                        # 给 Claude 看的工作流说明（也是人看的总览）
├── .env.example                    # 凭证模板
├── references/
│   ├── components.md               # 组件库 + 设计 token + SVG 信息图模板（排版前必读）
│   ├── article-template.html       # 带手机预览框的文章骨架，复制它开工
│   ├── wechat-html-spec.md         # 微信 HTML/CSS 支持与过滤规范
│   └── styles/
│       ├── tech-card-green.md      # 技术卡片风预设（教程 / AI 工具 / SaaS 文档）
│       └── ai-news-signal-green.md # AI 新闻信号卡风预设（模型发布 / 快讯解读）
└── scripts/
    ├── publish.ts                  # 发布入口，支持 .html 和 .md
    ├── render.ts                   # Markdown 固定主题渲染（模式 B）
    ├── imagegen.ts                 # 封面 / 配图生成
    ├── img2base64.ts               # 图片直链/本地图 → 下载校验压缩 → base64 data URI
    └── wechat.ts                   # 微信 API 封装
```

---

## 微信排版的硬规则（写 HTML 前先知道）

- **全部样式必须写成内联 `style`**。`<style>` 标签、`class`、`id` 在微信全部失效。
- 不支持 JavaScript（`<script>` 被剔除）、`position`、`@media`、CSS 动画。
- 静态 `flex` 布局可用，但别依赖 `gap` 做关键间距，必要时用 `margin`。
- **内联静态 `<svg>` 微信支持**，是做数据可视化的首选。
- 正文图片单张 ≤ 1MB（微信 `media/uploadimg` 限制），封面 ≤ 10MB。

更全的支持/过滤清单见 `references/wechat-html-spec.md`。

---

## 常见错误

| 报错 | 处理 |
|------|------|
| `Missing WECHAT_APP_ID` | 没配 `.env` |
| `errcode=40164` / IP 限制 | 运行机器的公网 IP 没加进公众号后台「IP 白名单」 |
| `errcode=48001` 接口未授权 | 公众号未认证或无 `draft/add` 权限 |
| `Body image too large` | 正文图超 1MB，压缩后重试 |
| `No title found` | 传 `--title` 或在 HTML 里放一个 `<h1>` |

---

## 限制

- 不含 SVG 互动动画——互动效果走 `draft/add` 接口会被微信过滤掉。
- 一键发布依赖**已认证**公众号；个人订阅号通常没有 `draft/add` 权限，请用「用法一：只排版」。

---

## 署名 / Attribution

- **作者 ID：喂鱼**（全平台同名）· GitHub：**sakuraoxo**
- **公众号：喂鱼**
- **原始仓库：** https://github.com/sakuraoxo-clio/wechat-publisher

本项目以 MIT 协议开源，欢迎 fork 和二创。修改版、教程或再分发版本须在其 README、注释或文档中保留以上署名与原始仓库链接，并注明所做修改；不得将修改版表述为原作者的官方版本。

Licensed under MIT — forks and derivatives are welcome. Modified or redistributed versions must retain the attribution above and the original repository link in their README, comments, or documentation, state their modifications, and must not present themselves as official releases by the original author.

## License

[MIT](./LICENSE) © 2026 sakuraoxo（喂鱼）
