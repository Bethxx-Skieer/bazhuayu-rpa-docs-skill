# 八爪鱼 RPA 文档站（bazhuayu-rpa-docs）工作流 Skill

本仓库是 **八爪鱼 RPA 帮助文档站**（`https://bazhuayu-rpa-docs.mintlify.app`）的维护工作流沉淀，可作为 WorkBuddy 的 Skill 使用，也可直接当作操作手册阅读。

当换电脑或新开环境时，拉取本仓库即可恢复完整的文档审计 / 修复 / 提交 / 部署核验流程，无需从头摸索。

---

## 文件说明

主文档 `SKILL.md` 刻意控制在 100 行内，作为**调度入口**：按场景用 `Read` 加载对应的子文档。

| 文件 | 作用 |
| --- | --- |
| `SKILL.md` | **调度入口（≤100 行）**。触发条件、子文档索引表、标准动作摘要、输出承诺。 |
| `audit-checklist.md` | 审计 8 项必查清单 + 第二批典型案例。拿到新 URL 时先读。 |
| `execution-sop.md` | 完整 SOP（读取→审计→修复→校验→推送→线上核验）+ 9 列输出表格格式。 |
| `pitfalls.md` | 踩坑注意事项：删文件 / revert / 脚本 / iframe / mintcdn / 图片清理纪律。 |
| `site-404-troubleshoot.md` | push 后整站 404、部署失败的排查流程（路由层 vs 构建层判定）。 |
| `recent-points.md` | 2026-09 近期实战补充要点 A–K（大分类标题化、表格图放大居中、加粗失效、旧链接替换、首页重写、清理、部署失败排查、rebase、域名差异、JPEG 误标 PNG）。 |
| `_mdxcheck.cjs` | MDX 语法校验脚本。批量编译 `.mdx` 文件，捕捉会导致单页 404 的语法错误。每次改完文档必跑。 |

> 所有子文档与主文档同目录，WorkBuddy 加载 `SKILL.md` 后按索引 `Read` 对应文件即可。

---

## 作为 WorkBuddy Skill 安装

1. 将本仓库内容放到本地 skill 目录：
   - 用户级：`~/.workbuddy/skills/rpa-doc-audit-workflow/`
   - 项目级：`<workspace>/.workbuddy/skills/rpa-doc-audit-workflow/`
2. 确保 `_mdxcheck.cjs` 同目录或文档仓库根目录可用。
3. 在对话中提"审查某个文档 URL / 修复某页排版"等，Skill 会自动按流程触发。

> 仅阅读手册时忽略上述安装步骤，直接看本目录下各 `.md` 文件即可（`SKILL.md` 为入口索引，按需跳到对应子文档）。

---

## 前置依赖

- **Node + `@mdx-js/mdx`**（供 `_mdxcheck.cjs`）：
  ```bash
  # managed node workspace 下
  cd ~/.workbuddy/binaries/node/workspace
  npm install @mdx-js/mdx
  ```
  脚本内 `createRequire` 指向该路径，按需改成你环境的 `@mdx-js/mdx` 实际位置。
- **Python 3 + PIL**（图片真伪校验 / 转格式）：
  ```bash
  python -m venv ~/.workbuddy/binaries/python/envs/default
  ~/.workbuddy/binaries/python/envs/default/bin/pip install pillow
  ```
- **文档仓库本体**：`https://github.com/Bethxx-Skieer/bazhuayu-rpa-docs`（本 Skill 只含工作流，不含文档站源码）。

---

## 标准工作流（每次改文档都走一遍）

1. **读取 + 解析**目标 MDX，对照 `SKILL.md` 的 8 项审计清单逐条过。
2. **修复**：表格图移出 / 改 HTML 表格、`<strong>` 相邻拆开、`H2` 前插 `<br/>`、description 去重、动图误标改 GIF、层级错位升 H2、重复段删除、内联小图标改文字。
3. **校验**：`node _mdxcheck.cjs <改的文件...>` 必须全 `ALL_OK`；改了 `docs.json` 才跑 JSON 合法性校验。
4. **提交推送**：
   - `git add` 指定文件（**绝不** `git add .`，会混入 untracked 临时脚本）。
   - `git commit -m "fix(<分类>): ..."`
   - `git push origin main`（网络受限时加 `dangerouslyDisableSandbox`）。
   - push 被拒 → `git fetch` + `git pull --rebase origin main` + 再 push，**不强行 force**。
5. **线上核验**：Python urllib（带 Chrome UA）请求 `https://bazhuayu-rpa-docs.mintlify.app/<path>`，轮询至内容生效；grep 关键特征串确认部署成功。

---

## 关键纪律（踩坑精华）

- **清理只删未跟踪缓存**（`_html/`、`_vhtml/`、`_newcat/`、`_parsed/`、`_vcheck/`、`_srclmgs/`、`__pycache__/`、剪贴板截图）。**严禁批量删 `images/` 下的生产图片** —— 扫描脚本会把 `docs.json` 引用的 `favicon.png`/`logo.png` 误判为"未引用"而误删，一旦误删立即 `git restore $(git diff --name-only --diff-filter=D)`。
- **部署失败先看失败 commit 是不是自己的**：别人 merge 带进语法错误的 MDX 也会让 Dashboard 显示 failed，与你的账号无关。
- **`.mintlifysite.com` 与 `.mintlify.app` 内容一致**，沙箱里前者常 DNS 解析失败，核验统一用 `.app`。
- **不要 `git rm` 删仓库内文件**（触发 safe-delete 级联拦截 + git index 错位），用 `rm` 删文件系统 + `git add -A` 暂存。
- **`git revert` 会 reapply 原始 commit 的所有改动**，可能覆盖中间改进；revert 后立刻 `git checkout <新commit> -- <受影响文件>` 恢复。

更完整的案例与判定方法见 `SKILL.md`。
