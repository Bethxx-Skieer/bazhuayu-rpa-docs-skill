---
name: rpa-doc-audit-workflow
description: 八爪鱼 RPA 文档站（bazhuayu-rpa-docs）所有 MDX 文章的审计与修复标准流程。用户每给一个文章 URL 或一组链接时自动触发。检查项：粗体标错、表格内图片无法点击放大、大分类之间空行太小、frontmatter description 与正文重复、动图误标 PNG、层级错位、重复标题/intro 段、废弃内联小图标。每项都给出可执行的修复方法、对应 mdxcheck 与 git 提交流程。适用于 guides/、commands/、getting-started/ 全部子目录的 MDX 文章，不用于全新写作或未圈定范围的全局治理。
---

# RPA 文档站文章审计标准流程

## 触发条件

用户给出**任意**一个或一组 Mintlify 文档站（`bazhuayu-rpa-docs.mintlify.app`）的文章 URL、MDX 路径、或在线预览截图时，**必须**按本流程走一遍审计 + 修复，不限于用户当时指出的问题。若用户说「这个问题在 X 分类下所有文章都存在」，**立即扩展到该分类（或 docs.json 中同一 group）下全部 MDX 同步自查 + 修复**。

适用目录：`guides/**`、`commands/**`、`getting-started/**`。`docs.json` 不动（除非用户明确要求改导航）。

## 子文档（按需用 Read 加载）

主文档精简到此，详细规范拆到以下文件，**按场景用 Read 工具加载对应文件**：

| 文件 | 何时读 |
| --- | --- |
| `audit-checklist.md` | 拿到新 URL，逐项过 8 条审计清单前 |
| `execution-sop.md` | 要执行「读取→审计→修复→校验→推送→线上核验」完整 SOP，含输出表格格式 |
| `pitfalls.md` | 准备删文件 / revert / 写脚本 / iframe / mintcdn 等踩坑前 |
| `site-404-troubleshoot.md` | push 后整站或全部页面 404、部署失败时 |
| `recent-points.md` | 处理企业版功能页大分类标题化、表格图片垂直居中、旧链接替换、首页重写、清理、部署排查时 |

校验工具：`_mdxcheck.cjs`（与本文件同目录，随 Skill 附带，对 `**/*.mdx` 逐文件 `compile()` 校验）。

## 标准动作（摘要）

1. **读取并解析**目标 MDX +（可选）源文档比对。
2. **逐项过审计清单**（先读 `audit-checklist.md`）。
3. **修复**：表格图移出 + 加粗小标题；`<strong>` 相邻拆开；`<br/>` 插 H2 前；description 重复按决策删一边；动图误标 `git mv` + 真 GIF。
4. **校验**：`node _mdxcheck.cjs <files...>`（必跑）；改导航才跑 `python -c "import json; json.load(open('docs.json'))"`。
5. **提交推送**：`git add` 指定文件（**不要** `git add .`）；`git commit -m "fix(<category>): ..."`；`git push origin main`（`dangerouslyDisableSandbox:true`）。
6. **线上核验**：Python urllib（带 Chrome UA）轮询 `.mintlify.app`，确认 200 且关键串已上线。

> 详细 SOP、踩坑、404 排查、近期实战要点见对应子文档。

## 输出承诺

完成审计 + 修复后，**必须**给用户一份 9 列清单（页面 / 粗体 / 表格图 / 空行 / description / GIF / 层级 / 重复段 / 内联图 / 备注），并附 commit hash + 线上 preview URL。
