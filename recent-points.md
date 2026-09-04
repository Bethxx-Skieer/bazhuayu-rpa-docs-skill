# 近期实战补充要点（2026-09）

以下为 2026-09-03 前后「企业版功能页批量修订 + 首页重写 + 清理 + 部署排查」中沉淀的新要点，作为前 8 项审计清单与注意事项的补充。

## A. 大分类用标题而非引用块

企业版 API 页（如 `webhook-trigger`）常把大分类写成 blockquote：`> 一、功能介绍` / `> 二、使用前提`。Mintlify 渲染成引用样式，不符合"大分类作为标题"的要求。
- 修复：改为 `## 一、功能介绍` / `### 1.1 使用前提`，二级小节用 `### N.N`。

## B. 表格图片可点击放大 + 垂直居中（HTML 模板）

需求："表格里的图点不开 + 说明要垂直居中"。Markdown `| ![](images/xx.png) |` 无法放大且难控制对齐。改用 HTML 表格：

```jsx
<table style={{ borderCollapse: 'collapse' }}>
  <thead>
    <tr>
      <th style={{ border: '1px solid #ddd', padding: '8px', textAlign: 'left' }}>参数</th>
      <th style={{ border: '1px solid #ddd', padding: '8px', textAlign: 'left' }}>说明</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td style={{ border: '1px solid #ddd', padding: '8px', verticalAlign: 'middle' }}>sign</td>
      <td style={{ border: '1px solid #ddd', padding: '8px', verticalAlign: 'middle' }}>
        签名参数，生成方式见下图<br/>
        <img src="/features/enterprise/api/images/start-app-01.png" alt="sign 参数" style={{ maxWidth: '400px', marginTop: '8px', display: 'block' }} />
      </td>
    </tr>
  </tbody>
</table>
```

要点：
- `verticalAlign: 'middle'` 实现单元格垂直居中。
- `<img>`（而非 `![]()`）放在 td 内可触发 Mintlify lightbox 放大。
- `src` 用仓库绝对路径 `/features/.../images/xx.png` 比相对路径 `images/xx.png` 更稳（规避 Mintlify 相对路径缓存不加载问题，见 D）。

## C. 加粗没显示的两种根因

1. `** 文本**`（加粗内容内部前导/尾随空格）→ MDX 不渲染加粗。例：`** configs[i]字段：**` 应改为 `#### configs[i]字段`（用标题替代），或去掉空格变 `**configs[i]字段：**`（内容含特殊字符仍建议用标题）。
2. blockquote 内的 `**描述：**` 被 `>` 引用包裹导致加粗不显。修复：去掉行首 `>`，改为普通加粗行 `**描述：** 不知道如何搭建流程？...`。

## D. 图片相对路径不加载 → 改绝对路径

`app-development` 总览图 `![...](images/app-development-overview.png)` 线上未加载。排查：文件存在、magic bytes 为合法 PNG（`\x89PNG`）、尺寸正常 → 非损坏，疑为相对路径 + Mintlify 缓存。
- 修复：改为仓库绝对路径 `![...](/features/app-development/images/app-development-overview.png)`，并用 PIL 重新 `img.save(p, format='PNG')` 确保无损。
- 原则：图片引用优先用 `/分类/子分类/images/xx.png` 绝对路径，少踩相对路径坑。

## E. 旧文档站链接替换（整站扫描）

用户要求扫全站 `https://rpa.bazhuayu.com/helpcenter/docs` 旧链接并替换。
- 用 `grep -rIn --include=*.mdx "rpa.bazhuayu.com/helpcenter/docs"` 全仓扫（排除 `_html/`、`_parsed/` 等未跟踪临时目录——它们不随站点发布）。
- 替换为站内相对链接：如 `[RPA 机器人](/features/enterprise/bot)`、`[成员管理](/features/enterprise/member-management)`、`[本地触发任务](/features/app-management/triggers)`。
- `.mintlifysite.com` 全站链接也统一改为站内相对 `/路径`（避免外链依赖域名）。
- 同页已有内容 → 改为页内锚点：`extractdatacommand` 的 `[批量采集&逐条采集](https://rpa.bazhuayu.com/helpcenter/docs/gIuxnds3)` 改为 `[批量采集&逐条采集](#两种采集模式)`（该小节就在下方）。

## F. 首页按实际结构重写（修 404 死链）

旧 `index.mdx` 是通用模板，含 `/features`、`/guides/workflow` 等不存在的页面（404）。
- 重写前先 `ls` / `find` 确认每个目标页真实存在。
- 按 `docs.json` 的 `navigation` 实际分区建卡片：快速开始 / 功能说明 / 指令概述 / 专题文档 / 常见问题 / 学院 / 更新日志。
- 无 index 落地页的分区（如 guides、features）指向最具代表性的真实页（如 `/guides/ai-write-flow`、`/features/app-management`）。

## G. 清理：瘦身但严禁误删生产资产

- 可放心删的临时/缓存（不随站点发布）：`_html/`、`_vhtml/`、`_newcat/`、`_parsed/`、`_vcheck/`、`_srclmgs/`、`__pycache__/`、仓库根 `tmp_*.png`、`repo_tmp_orig*.png`、本次会话剪贴板截图（`~/.workbuddy/clipboard-images/clipboard-*.png`）。实测瘦身：1.1G → 477M。
- **严禁批量删图片**：
  - safe-delete 守卫在 ≥50 文件时拦停并等待确认，批量脚本会被中断。
  - 更严重：扫描"未引用图片"的脚本会把 `docs.json` 引用的 `assets/brand/favicon.png`、`logo.png` 也列为"未引用"而误删 → 实测一次误删 49 文件（含 favicon/logo），需立即 `git restore $(git diff --name-only --diff-filter=D)` 全量恢复。
  - 对策：未确认用途的生产图片（`commands/**/images/`、`guides/**/images/` 下约 100+ 张）**不要批量删**，单独与用户确认再处理。
- 临时脚本（`_*.py`）多为 untracked，删用 `rm`（而非 `git rm`，避免级联）。

## H. 部署失败排查：先确认失败 commit 是不是你的

Mintlify Dashboard 显示某次 Deployment failed，未必是你这边的问题。
- 案例：失败记录是 `Bethxx-Skieer` 推送的 merge commit（`c09b14e`「merge: integrate yangyifan local documentation updates」，改 44 文件 commands/ai），报错 `Encountered syntax error(s). Deployment not updated.`——是他人 merge 带进的 MDX 语法错误，与你的账号无关。
- 你自己的 push 仍显示 Successful。
- 排查：`git fetch` → `git log --oneline -8` 看失败 commit 的时间/作者/内容；`git show --stat <sha>` 看改了哪些文件。
- 若怀疑某 commit 的 MDX 有问题：把其文件列表导出，逐个跑 `_mdxcheck.cjs` 定位语法错误页。

## I. 推送被拒 → rebase 而非强推

`git push` 报远程有新提交（rejected）→ 不要 `git push --force`。
- `git fetch origin` → `git pull --rebase origin main` → 再 `git push origin main`。
- 若无冲突直接成功；rebase 后 commit hash 会变（非强推，历史线性）。

## J. 域名：`.mintlifysite.com` vs `.mintlify.app`

- 沙箱/部分网络下 `.mintlifysite.com` DNS 解析失败（getaddrinfo failed），`.mintlify.app` 正常（内容完全一致）。
- 线上核验统一用 `https://bazhuayu-rpa-docs.mintlify.app/<path>`（带 Chrome UA 的 urllib HEAD 请求）。
- 用户本地浏览器访问 `.mintlifysite.com` 一般正常；若打不开说明该域名已切到 `.app`。

## K. JPEG 误标 PNG（沿用底层规范）

源站 `.jpg` 常被存成 `.png` 文件名但内容仍是 JPEG（magic bytes `FF D8 FF`），浏览器按 PNG 解码破图。
- 批量扫描 `commands/**/images/*.png` + `guides/**/images/*.png`：读前 8 字节，`b[:3]==b'\xff\xd8\xff'` 即 JPEG 误标。
- 修复（venv PIL）：`Image.open(p).save(p, format="PNG")` 原地转真 PNG（文件名不变，线上 URL 不变）。

## L. docs.json redirects 注入（解决客户端旧链接迁移）

**场景**：客户端每条指令链接到 `https://rpa.bazhuayu.com/helpcenter/docs/{slug}`，开发者只能把 `/docs/*` 服务端统一 302 到 `/commands/*`，但新文档站实际是 `/commands/{分类}/{slug}`（多一层分类），跳转后会 404。

**方案**：用 Mintlify docs.json 顶层 `redirects` 字段，把每个具体指令页加一条：
```json
{ "source": "/commands/{slug}", "destination": "/commands/{category}/{slug}" }
```
Mintlify 自动 308 跳转，无需后端配合。

**实施**：
1. 扫描 `commands/{category}/*.mdx` 列出所有具体指令 slug（跳过 `commands/{category}.mdx` 总览）。
2. Python 合并入 docs.json（dict 去重 + 校验 destination 全部在 navigation 里存在）。
3. 全链路：`/docs/{slug}` → 服务端 → `/commands/{slug}` → docs.json 308 → `/commands/{category}/{slug}`（200）。

**2026-09-04 实战**：`bazhuayu-rpa-docs` 21 个分类、330 条 redirect 注入 docs.json，commit `2d989b5`，每分类抽样 1 个共 20 个验证全部 308 OK。

## M. 线上核验必须先看 sitemap.xml 确认真实 host

**坑**：本仓库的预览域名 `bazhuayu-rpa-docs.mintlify.app` 与生产域名 `rpa.bazhuayu.com/helpcenter/` 内容一致但路由缓存时序不同——同一次部署后，预览域可能首页 200 + 具体页 404，但生产域是正常的；反之亦然。

**对策**：
- 核验任何 URL 前先请求 `https://<base>/sitemap.xml`，正则提取所有 `<loc>` 标签，**第一个出现的 host** 就是生产域（或当前 CDN 实际命中的 host）。
- 再用那个 host 做所有 200 / 404 / 跳转核验。
- 不要预设 `.mintlify.app` 就是生产域。
