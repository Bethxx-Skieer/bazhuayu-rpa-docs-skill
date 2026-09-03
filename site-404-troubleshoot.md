# 整站 404 排查流程（push 后所有页面都打不开时）

**不要一上来就 revert**。先按下面顺序定位——绝大多数"整站 404"其实是平台侧问题，而非仓库内容问题。revert 往往白做。

## 第 1 步：判断是"内容没构建"还是"路由挂了"

关键诊断点——对比请求**无后缀**与**带 `.md` 后缀**的 URL：

```python
u = "https://bazhuayu-rpa-docs.mintlify.app/guides/elements/element"
# 分别请求 u 和 u + ".md"，比较结果
```

| 无后缀 | `.md` 后缀 | 结论 |
| --- | --- | --- |
| 404 | 200 | **平台路由故障**（内容已构建，rewrite 失效）→ 走第 4 步 |
| 404 | 404 | 内容未构建 / 构建失败 → 走第 2 步 |
| 200 | 200 | 已恢复，只是 CDN 缓存 → 等缓存过期 |

## 第 2 步：验证内容是否真的构建成功

```python
for path in ["/llms.txt", "/llms-full.txt", "/sitemap.xml"]:
    # 返回真实内容且体积正常 = 构建成功
    # 参考量级：llms.txt 数万字节，llms-full.txt 数十万字节，sitemap.xml 与页面数成正比
```

三个都返回完整内容 → 构建没问题，故障在路由层。

## 第 3 步：查本地仓库自洽性

```python
import json, subprocess
d = json.loads(subprocess.check_output(['git','show','HEAD:docs.json'], text=True))
# 递归收集 navigation 下所有 pages 字符串条目
# 检查每个条目对应的 .mdx / .md 是否存在
```

**缺失页面数 = 0** 且 `json.load(open('docs.json'))` 通过 → 本地无问题。
同时确认 docs.json 里没有 `redirects` / `rewrites` / `basePath` / `trailingSlash` 等干扰路由的字段（本仓库顶层键只有 `$schema / theme / name / description / colors / logo / favicon / navigation`）。

## 第 4 步：读响应头定位

```python
# 重点关注
X-Matched-Path: /_sites/[subdomain]/[[...slug]]   # 动态子域路由，出现即说明站点元数据已加载
X-Nextjs-Prerender: 1                              # 预渲染
Server: Vercel                                     # Mintlify 底层部署在 Vercel
X-Vercel-Cache: HIT / MISS
Age: 586                                           # 边缘缓存秒数
X-Version: dpl_xxx                                 # 部署 ID，可用于对比是否换了新部署
```

`X-Matched-Path` 出现 `_sites/[subdomain]` → 站点元数据已加载，故障在 slug 匹配环节，属平台侧。

## 第 5 步：排除"爬虫被区别对待"

用完整浏览器头复测（UA + `Accept` + `Accept-Language` + `Sec-Fetch-*`）。若仍 404，排除 UA 因素，确认真故障。

## 第 6 步：平台侧处理

确认为平台故障后：

1. **Mintlify Dashboard** → 项目 → **Deployments**，看最新部署是否 Success、有无 build error 日志（最权威信息源，本地无法替代）。
2. 确认 Mintlify ↔ GitHub 连接状态（token 是否过期、webhook 是否仍触发）。
3. 平台无报错但路由仍失效 → 提 Mintlify 工单，附 `X-Version: dpl_xxx` 部署 ID + "无后缀 404 / `.md` 200" 对比证据。
4. **不要反复 push 试探**——不会修复问题，只会让部署队列更长。

## ⚠️ Dashboard 显示 "Successful" ≠ 路由一定正常

**关键点**：Mintlify Dashboard 的 "Successful" 指的是 **构建/部署**成功（产物生成、CDN 推送），不保证 **路由层**（Vercel Next.js rewrite）正确工作。

整站 404 时务必 **同时验证**：

| 检查点 | 怎么看 | 含义 |
| --- | --- | --- |
| Dashboard Status | 点进 deployment 详情 | 成功/失败 |
| 页面 `<title>` | 抓首页或任意页 HTML 里的 `<title>` | 显示最新标题 → 构建层用了新代码 |
| `.md` 后缀 URL | `curl /guides/elements/element.md` | 200 → 构建产物完整 |
| 无后缀 URL | `curl /guides/elements/element` | 404 → 路由层坏了 |

**典型故障组合**：Dashboard ✅ Successful + title 显示新代码 + `.md` 200 + 无后缀 404 = **构建完好，路由层（Vercel rewrite）失效**。此时 Dashboard 找不出问题，必须提工单附部署 ID + `X-Version: dpl_xxx` + Vercel 项目 ID（响应头 `X-Vercel-Project-Id: prj_xxx`）。

2026-09-01 本仓库案例实测：连续 4 个 commit（a848327 / 60db4e8 / cf22b2d / 461480b）Dashboard 全 Successful，但路由层从某个时刻起全 404，故障精确锁在 Vercel rewrite。

## 故障期间验证内容

平台故障期间，可用 `.md` 后缀 URL 验证内容确实已上线（如 `/guides/elements/element.md`）。llms.txt / sitemap.xml 也可作为内容已构建的证据。

## 故障期间的纪律

- 保持工作区干净（`git status` 无非 untracked 改动），避免与平台问题混淆、便于后续对比。
- 本地该做的修复照常做、照常 commit push——构建环节是好的，push 的内容会正确进入产物，平台恢复后立即可见。
- 恢复后重新跑一遍线上轮询 + 特征串核验。

## 本仓库实测案例（2026-09-01）

连续 3 个 commit（60db4e8 / cf22b2d / 461480b）push 后整站 404。排查结论：
- 无后缀 URL 全部 404，`.md` 后缀全部 200 → 平台路由故障
- llms.txt 43KB / llms-full.txt 554KB / sitemap.xml 67KB 全部正常 → 内容构建成功
- docs.json 引用 423 个页面，缺失 0 → 仓库自洽
- docs.json 无 redirects/rewrites 类配置 → 本地无干扰
- 响应头 `X-Matched-Path: /_sites/[subdomain]/[[...slug]]` → 元数据已加载，slug 匹配失效

结论：**内容全部正常上线，无需 revert，等平台修复路由即可**。
