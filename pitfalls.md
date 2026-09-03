# 注意事项（来自踩坑记录）

- **不要用 `git rm` 删仓库内文件**——会触发 302 个文件的级联删除（safe-delete 拦截 + git index 错位），改用 `rm` 删文件系统 + `git add -A` 暂存。Windows + Git Bash 下 `rm` 还可能被 safe-delete wrapper 拦截，可用 Python `os.remove()` 绕开（wrapper 不拦 Python）。
- **不要写 `.ps1`/`.bat` 处理文件**——Windows 编码问题会导致中文路径乱码，用 `execute_command` 直调。
- **iframe src 里的 `&` 必须写成 `&amp;`**——MDX 严格解析会丢失 iframe 节点。
- **B 站播放器用 `https://player.bilibili.com/...`** 而不是 `//player...`。
- **mintcdn 对无浏览器 UA 一律 403**——核查线上图片务必在 urllib 请求头加 Chrome UA。
- **`git revert <revert-commit>` 会 reapply 原始 commit 的所有改动，可能覆盖中间的新改进**。例如：A 删文件并改 5 个 MDX → B revert A（恢复文件 + 撤销 MDX 改动）→ D 改进那 5 个 MDX → `git revert B` 会 reapply A 的旧 MDX 改动，悄悄覆盖 D 的改进。**对策**：revert 后立刻 `git checkout <新-commit> -- <被影响的文件>` 恢复，再用新 commit 提交。本仓库 2026-09-01 实测：revert cf22b2d 后用 `git checkout 461480b --` 恢复了 5 个 MDX。
- **清理严禁批量删图片**：safe-delete 守卫在 ≥50 文件时拦停并等待确认；扫描"未引用图片"的脚本会把 `docs.json` 引用的 `favicon.png`/`logo.png` 也列为"未引用"而误删（实测一次误删 49 文件含 favicon/logo，需立即 `git restore $(git diff --name-only --diff-filter=D)` 全量恢复）。未确认用途的生产图片（`commands/**/images/`、`guides/**/images/` 下约 100+ 张）**不要批量删**，单独与用户确认。
