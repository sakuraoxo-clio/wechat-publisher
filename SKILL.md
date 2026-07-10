---
name: wechat-publisher
description: 把文章排版成微信公众号合规 HTML、自动上传图片、可选自动生成封面图，并通过官方 API 一键写入公众号草稿箱，用户只需在后台预览后发布。优先用组件化手写 HTML 做精排版，也支持 Markdown 快速排版。当用户提到「发布公众号」「推送到草稿箱」「公众号排版」「post to wechat」「微信公众号文章」时使用。
---

# WeChat Publisher — 文章一键进公众号草稿箱

把文章变成公众号草稿：排版 → 上传图片 →（可选）生成封面 → 写入草稿箱。用户在 `mp.weixin.qq.com` 后台预览满意后发布。

**排版分两种模式，默认用模式 A：**

| 模式 | 用法 | 适用 |
|------|------|------|
| **A 组件化手写 HTML（默认）** | Claude 照组件库手写带卡片/SVG 信息图的 HTML | 正式文章、要质感的内容 |
| B Markdown 快速排版（兜底） | 喂 `.md`，脚本套固定主题渲染 | 草稿、临时、对排版无要求 |

固定主题（`render.ts`）天花板有限——千篇一律、像文档。**正式文章一律走模式 A。**

## 两种交付方式（先看这个）

排好版后，有两条路把文章送进公众号。**默认先给「复制预览」，再主动问要不要一键入库**：

1. **复制预览（零门槛，谁都能用）** —— 把排好的 HTML 在浏览器打开，用户全选复制、手动粘进公众号编辑器即可。**不需要任何凭证、不需要认证公众号。** SVG 信息图和 base64 图片粘贴时公众号会自动收图。
2. **一键进草稿箱（需已认证公众号）** —— 通过官方 `draft/add` 接口直接写入，省掉复制粘贴和重新传图。需要凭证 + IP 白名单（见下）。

排版完成后，先交付预览，再问用户：「要不要我直接一键发到你的草稿箱？需要的话我带你配一次凭证（一次性，3 步）。」用户不需要就到此为止。

## 一键发布的凭证配置（仅「一键进草稿箱」需要）

前提：公众号为**已认证**服务号 / 订阅号（有 `draft/add` 权限）。带用户走三步：

1. **拿凭证**：让用户登录**自己的**公众号后台（mp.weixin.qq.com → 开发 → 基本配置），复制 AppID / AppSecret。
   - 默认（小白最快）：用户把 AppID / AppSecret 发来，你写进项目根目录的 `.env`（`WECHAT_APP_ID` / `WECHAT_APP_SECRET`）。**写完不要在对话里把 AppSecret 明文复述出来**；`.env` 已被 `.gitignore` 忽略，只留在本机。
   - DIY：用户也可自己把 `.env.example` 复制成 `.env` 填好，脚本会自动读取。
2. **填 IP 白名单**：运行 `curl -s https://api.ipify.org` 取本机公网 IP，直接把这串 IP 告诉用户，让他加进后台「IP 白名单」。（家庭宽带 IP 是动态的，变了要回去重填。）
3. **装依赖**：`cd scripts && npm install`；用自动生成封面再填 `OPENAI_API_KEY`。

不会操作的用户，指给详细图文教程 👉 https://sakuraoxo.feishu.cn/wiki/TuA7wvDjiiQZQVkGNp1cK6uynae

`.env` 缺失或报 `40164`/`48001` 时，按上面引导用户，**不要编造或猜测凭证**。

---

## 模式 A：组件化手写 HTML（默认）

### 流程

```
- [ ] Step 1: 读 references/components.md（组件库）和 references/article-template.html（骨架）
- [ ] Step 2: 复制 article-template.html 到文章目录，按内容设计、填充组件
- [ ] Step 3: 配图 —— 内联 SVG 信息图 / HTML 截图 / 网图抓取转 base64 / gpt-image-2 生图
- [ ] Step 4: 本地预览（在浏览器打开 HTML，看手机框内效果）
- [ ] Step 5: 交付预览给用户复制；再主动问要不要一键进草稿箱
- [ ] Step 6: 用户要 → 确认/配好凭证后运行 publish.ts；不要 → 到此为止
- [ ] Step 7: 报告结果
```

### Step 2 — 设计排版

- **必须先读** `references/components.md`，用里面的组件：封面卡、编号小标题、callout 框、金句框、步骤卡、文末总结块、标签 chips。
- 如用户指定某种风格，或内容明显适合已有风格预设，再读取 `references/styles/` 下对应文件作为视觉覆盖层；例如技术教程、AI 工具、SaaS 文档、产品更新、安装指南优先读 `references/styles/tech-card-green.md`；AI 新闻、模型发布、真假信号拆解、行业快讯解读优先读 `references/styles/ai-news-signal-green.md`。
- 文章内容写在 `article-template.html` 的 `<!-- ARTICLE HTML START -->` / `<!-- ARTICLE HTML END -->` 之间。
- 全程内联样式，不用 class/id。一篇挑 3–5 种组件循环用。

### Step 3 — 配图（和「套模板」拉开差距的关键）

每个核心论点尽量配一张图，四种来源按优先级：

1. **内联 SVG 信息图**（首选）：数据可视化（对比、时间线、飞轮、曲线）直接用 SVG 代码画进正文。微信支持、矢量清晰、不占图片配额。模板见 components.md。
2. **HTML 做图**：复杂图表用 HTML/CSS 写好，Chrome headless 截图成 PNG：
   ```bash
   "/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" --headless --disable-gpu \
     --hide-scrollbars --force-device-scale-factor=2 --default-background-color=ffffffff \
     --screenshot="out.png" --window-size=<宽>,<高> "file://<绝对路径>"
   ```
   窗口高度要贴合内容（可先用 playwright `evaluate` 取元素 `offsetHeight`），多余留白用 `sips` 裁掉。
3. **网图抓取 → base64 内嵌**：需要真实照片/实拍/产品图/截图时，**主动联网搜图**（用 WebSearch / WebFetch / 浏览器找到一张可用图片的**直链**），再用工具转成可内嵌的 base64：
   ```bash
   cd scripts && npx tsx img2base64.ts "<图片直链>" --max-kb 980
   ```
   - 它会**下载 → 校验确实是图片 → 超 1MB 自动压缩 → 输出 `data:image/...;base64,...`**。把这串直接填进 `<img src="...">`。复制预览交付时，base64 图会随「复制渲染内容」自动进公众号。
   - 抓到的不是真图（被反爬返回网页 / 404）脚本会**报错退出**——换一张图或换直链，**绝不编造图片网址**（会裂图）。
   - 优先选**可商用 / 无版权风险**的图源（Unsplash、Pexels、维基共享、用户自有素材），别直接扒有版权的图。
   - 实在搜不到合适的真图，再退回浅灰占位块 + 图注写建议搜索词（见 wechat-format-prompt.md）。
4. **gpt-image-2 生图**：插画、封面、meme 等非数据类配图。`imagegen.ts` 已封装，默认模型 `gpt-image-2`。

正文图单张 ≤1MB（微信 `media/uploadimg` 限制），超了用 `sips` 压缩转 jpg；`img2base64.ts` 已内置这步。

### Step 6 — 发布（HTML 输入，仅当用户选择一键入库）

```bash
npx tsx publish.ts <文章.html> --title "标题" --author "作者" \
  [--digest "摘要"] [--cover <封面图> | --gen-cover]
```

- `.html` 输入：脚本提取 `ARTICLE HTML` 标记之间的内容，**不做 markdown 渲染**，原样作为正文。
- 标题/作者/摘要通过 CLI 传；不传 `--title` 时取文章里第一个 `<h1>`。
- 封面 `news` 文章必需：`--cover <路径>` 或 `--gen-cover`（需 `OPENAI_API_KEY`）。
- 正文里的 `<img>`（本地路径 / 远程 URL / base64）会自动上传微信并替换 src；内联 `<svg>` 原样保留。

---

## 模式 B：Markdown 快速排版（兜底）

```bash
npx tsx publish.ts <文章.md> [--title ..] [--author ..] [--cover <路径> | --gen-cover]
```

`.md` 走 `render.ts` 固定主题（正文 15px / 行高 1.8 / 微信绿）。支持 frontmatter：`title` / `author` / `description` / `cover`。仅用于不在意排版质感的场景。

---

## Step 7 — 完成报告

输出草稿 `media_id`，提示用户：登录 `https://mp.weixin.qq.com` → 内容管理 → 草稿箱 → 预览 → 发布。

## 文件

| 路径 | 作用 |
|------|------|
| `references/components.md` | 组件库 + 设计 token + SVG 信息图模板（写 HTML 前必读） |
| `references/styles/tech-card-green.md` | 绿色技术卡片风预设，适合技术教程、AI 工具、SaaS 文档、产品更新 |
| `references/styles/ai-news-signal-green.md` | AI 新闻信号卡风预设，适合模型发布、真假消息拆解、行业快讯解读 |
| `references/article-template.html` | 带手机预览框的文章骨架，复制它开工 |
| `references/wechat-html-spec.md` | 微信 HTML/CSS 支持与过滤规范 |
| `scripts/publish.ts` | 发布入口，支持 `.html` 和 `.md` |
| `scripts/render.ts` | Markdown 固定主题渲染（仅模式 B） |
| `scripts/imagegen.ts` | gpt-image-2 生图 |

## 限制

- 不含 SVG 互动 —— 互动动画走 `draft/add` 会被过滤，见 `references/wechat-html-spec.md`。
- 正文图片单张 ≤1MB。

## 常见错误

| 报错 | 处理 |
|------|------|
| `Missing WECHAT_APP_ID` | 配置 `.env` |
| `errcode=40164` / IP 限制 | 把运行机器公网 IP 加入公众号后台 IP 白名单 |
| `errcode=48001` 接口未授权 | 公众号未认证或无 `draft/add` 权限 |
| `HTML file looks like a full document` | 给文章内容加上 `ARTICLE HTML START/END` 标记 |
| `Body image too large` | 正文图超 1MB，`sips` 压缩后重试 |
| `No title found` | 传 `--title` 或在 HTML 里放一个 `<h1>` |

---

> 作者 / Author：**sakuraoxo（喂鱼）** · 公众号：**喂鱼** · MIT License
> 原始仓库 / Original repo：https://github.com/sakuraoxo-clio/wechat-publisher
> 二创/再分发须保留本署名与原始仓库链接，不得暗示修改版为原作者官方发布。
> Derivatives must retain this attribution and the original repo link, and must not imply they are official releases by the original author.
