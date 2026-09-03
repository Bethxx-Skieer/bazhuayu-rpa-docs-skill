# 执行流程（拿到新 URL 后的 SOP）

1. **读取并解析**：
   ```bash
   # 读 MDX
   cat guides/<category>/<page>.mdx
   # 拉源文档比对（如用户给了源 URL）
   python -c "import urllib.request,ssl,re; ..."  # 参照 _fetch_xpath_9NSd7N.py 模式
   ```
2. **逐项过 8 条审计清单**（见 `audit-checklist.md`），列差异点（哪些有/哪些无/哪些部分有）。
3. **修复**：用 `Edit` 或 `Write` 工具改 MDX，注意：
   - 表格图 → 移出 + 加粗小标题
   - `<strong>` 相邻 → 拆开
   - `<br/>` 插 H2 前
   - description 重复 → 按决策删一边
   - 动图误标 → git mv + 真 GIF 覆盖 + MDX 同步
4. **校验**：
   ```bash
   node _mdxcheck.cjs <file1> <file2> ...  # 必跑
   python -c "import json; json.load(open('docs.json'))"  # 改了导航才跑
   ```
5. **提交推送**（参照 working memory 注意事项）：
   - `git add` 指定文件（**不要** `git add .`，会混入 untracked 临时脚本）
   - `git commit -m "fix(<category>): ..."`
   - `git push origin main`（需 `dangerouslyDisableSandbox:true`）
6. **线上轮询**（参照 working memory 「404 检查必须走 Python urllib」）：
   ```python
   import urllib.request, ssl, time
   ctx = ssl.create_default_context(); ctx.check_hostname=False; ctx.verify_mode=ssl.CERT_NONE
   UA = {"User-Agent":"Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/124.0 Safari/537.36"}
   url = "https://bazhuayu-rpa-docs.mintlify.app/<path>"
   for i in range(20):
       html = urllib.request.urlopen(urllib.request.Request(url, headers=UA), timeout=20, context=ctx).read().decode('utf-8','ignore')
       if "Page not found" not in html: break
       time.sleep(5)
   ```
7. **关键内容核验**：grep 关键字符串（新增/删除/修改的特征词）确认部署成功。

---

# 输出格式

完成审计 + 修复后，**必须**给用户一份清单：

| 页面 | 粗体 | 表格图 | 空行 | description | GIF | 层级 | 重复段 | 内联图 | 备注 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| xxx | OK/修 | OK/修 | OK/修 | OK/修 | OK/修 | OK/修 | OK/修 | OK/修 |  |

并附 commit hash + 线上 preview URL。
