# 审计清单（8 项必查）

每拿到一个新 URL，先**全量**对照清单扫描一遍，再针对性修复。

## 1. 粗体标错 / 加粗位置不对

- **症状**：`<strong>...</strong><strong>...</strong>` 多个相邻导致加粗边界跑偏；`**...**` Markdown 风格在 MDX 解析后位置错位；`<strong>描述：xxx</strong>` 把"描述："也加粗了。
- **判别**：用 `grep -n '<strong>'` 找连续相邻；用 `grep -n '\*\*'` 找 Markdown 风格；通读确认加粗位置与源文档一致。
- **修复**：相邻的 `<strong>` 拆开，让文字在中间；`描述：` 这种引导词**不要**加粗（参照 capture-element 多次修复）；XPath 表达式、关键术语加粗（如「**text()='xxx'**」、「**//a[@class='...']**」）。
  - 典型错误：`在RPA中，XPath能精准解析复杂网页结构，实现对目标元素的<strong>高效定位与</strong><strong>数据提取，是自动化任务中不可或缺的强力工具。</strong>` → 修正为普通文本或仅加粗关键词。
  - 典型错误：`<strong>1、什么是iframe框架</strong>` 这种加粗当 H2 用 → 改为 `## 1、什么是iframe框架`。
  - 典型错误：`**总结：**当发现...` / `**注意：**如需...` / `**效果展示：**` 等引导词带中文冒号整体加粗 → Mintlify 可能保留星号不渲染，应改为 `**总结**：当发现...`、普通正文 `注意：如需...`、或真正标题 `### 效果展示`。

## 2. 表格内图片无法点击放大

- **症状**：`| ![](images/xx.png) |` 把图片塞进 Markdown 表格的 `<td>`，Mintlify 渲染时不会触发图片 lightbox/放大。
- **判别**：`grep -n '| !\['` 或读 MDX 看是否有 `![](...)` 在表格行内。
- **修复**：
  - 选项 A：表格只保留文字（场景/说明/参数），图片移到表格**下方**作为独立 `![](images/xx.png)` 段落，加粗小标题（如 `**结构定位样式：**` + 图）。
  - 选项 B：必须用表格且要居中行标题时，用 HTML `<table>` + `<td style={{ verticalAlign: 'middle' }}>`（参照 xpath-with-bazhuayu-rpa 新手示例表格）。图片改用 `<img src="images/xx.png" />` 放在 td 内，可触发 lightbox 放大。
- **禁忌**：不要把 `![](...)` 直接写进 Markdown 表格单元格（Mintlify 不会给这类图片加 lightbox）。

## 3. 大分类之间空行太小（第一个除外）

- **症状**：H2 段落之间紧贴，无视觉间隔；用户用「每个大分类之间的空行大一点，第一个分类除外」描述。
- **判别**：相邻 H2 之间无空行/`<br/>`/`---`。
- **修复**：在第二个及之后的 H2 标题**前**插一个 `<br/>`（MDX 允许多个空行折叠，`<br/>` 是最稳的视觉间距手段）。
- **适用范围**：H2 大分类（不是 H3/H4）。第一个 H2 前不加。

## 4. frontmatter description 与正文重复

- **症状**：页面顶部 Mintlify 渲染的副标题（来自 `description`）与正文第一段 `> **xxx**` blockquote 或 `**描述：**xxx` 完全相同。
- **判别**：对比 `description` 字段与正文首段/blockquote。
- **修复决策**（参照 2026-09-01 variable.mdx 批量删字段）：
  - **指令页**（commands/**）：保留 frontmatter `description` 删正文重复段（用于搜索/SEO）。
  - **概念/指南页**（guides/**）：保留正文 `> xxx` / `**描述：**xxx` 删 frontmatter `description`（首页副标题区可以留空）。若用户截图红框框住标题下方重复 intro，通常就是 description 与正文 blockquote 重复。
  - 用单引号包裹含内双引号的值（`key: '值 "内双引号" 继续'`）。

## 5. 动图误标 PNG（GIF 文件但扩展名是 .png）

- **症状**：源站是 GIF（动画展示），但迁移时下载成 `.png` 文件，浏览器/CDN 按 PNG 解码后破图或不动画。
- **判别**：用 PIL 或纯 stdlib 读 magic bytes：
  ```python
  from PIL import Image
  im = Image.open('images/xx-NN.png')  # 真格式可能是 GIF
  print(im.format)  # 应是 PNG；若是 GIF 需改
  ```
  或快速判别：PIL `im.format == 'GIF'` 但扩展名 `.png` → 误标。
- **修复**：
  - `git mv images/xx-NN.png images/xx-NN.gif`
  - 从源站重新下载真 GIF 覆盖（Python `urllib` + Chrome UA 绕过 mintcdn 403）
  - MDX 引用同步改 `.gif`
  - 批量发现可用 PIL 遍历 `commands/**/images/*.png` + `guides/**/images/*.png`

## 6. 层级错位（H3 当 H2 用 / 反之）

- **症状**：整页只有 H3 没有 H2（如 xpath-by-tag-attribute.mdx 整改前）；或子标题比父标题还深；或「**1、xxx**」加粗当 H2。
- **判别**：`grep -n '^##\|^###'` 看层级链。
- **修复**：
  - 子分类正确升 H2（参照 xpath-by-tag-attribute 全部 `###` → `##`）。
  - 全部加粗当 H2 的（如 iframe-handling 整改前）统一用 `## N、xxx`。
  - H3 是子节（如 `### 常用方法`、`### 使用示例`）保留 H3。

## 7. 重复标题 / intro 段

- **症状**：页面顶部 `## 功能说明` + `**描述：**xxx` 与 frontmatter `description` 重复（edit-element/reselect-element 整改前）；或 `>` blockquote 与 `description` 完全相同（xpath-with-bazhuayu-rpa 整改前）。
- **判别**：对比标题块与 frontmatter。
- **修复**：直接删除重复的 `## 功能说明` 段；提升下一个原 H3 到 H2 补位（参照 edit-element.mdx 「### 编辑元素的两个入口」→「## 编辑元素的两个入口」）。

## 8. 废弃内联小图标（v 形箭头 / 下拉图标）

- **症状**：「鼠标移动到这个 ⬇️ 图案上面」等小图标 PNG 嵌入正文段落，渲染时图片不能与文字同行，破坏排版。
- **判别**：看 MDX 是否有 `1.xxxxx\n![](...)\nxxxxx` 这种「文字-图-文字」模式。
- **修复**：把小图标 PNG 替换为文字 `V`（参照 edit-element.mdx v1.0/v2.0、reselect-element.mdx v2.0）：
  ```
  1. 鼠标移动到这个 V 图案上面...
  ```
  废弃的原 PNG 不删（用户偏好保留既有图片/资产），仅不引用。

---

## 第二批典型案例（2026-09-01）

| 页面 | 主要问题 | 修复方法 |
| --- | --- | --- |
| `guides/images/image-element.mdx` | 内容需按新截图重写，description 与正文不匹配 | 按「元素」/「图像」两张新图重写概念与对比表格；description 按新正文重新总结 |
| `guides/images/capture-image.mdx` | 分类标题像指令文档（捕获方法/捕获后的操作/使用示例） | 按指南重新梳理：简介 → 捕获方式 → 管理已捕获的图像 → 在指令中使用图像 → 示例 |
| `guides/python/environment-setup.mdx` | frontmatter description 与正文 blockquote 重复 | 删除 `description` 字段；正文保留注意 blockquote；确认「Python库安装工具」链接指向 `/guides/python/library-installer` |
| `guides/python/library-installer.mdx` | 顶部 blockquote 与 description 重复；「效果展示：」「使用小Tips：」像小标题 | 删除顶部 blockquote；「效果展示」「使用小Tips」改为正文普通文本（不加 ###/##） |
| `guides/custom-instructions.mdx` | description 概述与正文重复；9 处表格内图片无法放大 | 按全文总结重新写 description；9 个 Markdown 图片表格改为 HTML `<table>` + `<td style={{ verticalAlign: 'top', textAlign: 'center' }}>` + `<img>` |

**新增自检点**：拿到图片替换需求时，用 `md5sum` 对比确认新图与仓库已有文件是否一致，避免误以为没替换。
