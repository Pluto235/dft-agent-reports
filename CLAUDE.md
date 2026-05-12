# CLAUDE.md

DFT-agent bench 报告归档站。每份报告是一个**独立 self-contained HTML 文件**，
通过 GitHub Pages 静态托管，分享 URL 永久有效。

## URLs

- 首页：https://pluto235.github.io/dft-agent-reports/
- 单份报告：https://pluto235.github.io/dft-agent-reports/<filename>.html

## Git remote

```
origin: git@github.com:Pluto235/dft-agent-reports.git  (main branch)
```

无 upstream、无 fork。这是 user 自己的归档仓。

## 仓库布局

```
.
├── .nojekyll                              # 关 Jekyll，防止下划线前缀文件被忽略
├── index.html                             # 首页 — 手维护的报告列表（最新在上）
└── <report-name>-YYYY-MM-DD.html          # 一份报告一个 HTML，self-contained
```

文件命名约定：`<topic>-YYYY-MM-DD.html`（topic 描述实验类型，日期是报告生成日）。

## 加新报告流程（**重要 — 每次都要做这 4 步**）

```bash
cd /home/shijie/dft-agent-reports

# 1. 把新生成的 HTML 拷进来（命名带日期）
cp /path/to/new-report-2026-MM-DD.html .

# 2. 编辑 index.html，在 <ul> 顶部加一行 <li>（保持最新在上）
#    填三个字段：标题、url、small 描述（日期 + 题数 + 模型 + 任何 highlight）

# 3. 提交 + 推
git add new-report-2026-MM-DD.html index.html
git commit -m "report: <一句话主题>"
git push origin main

# 4. 等 1-2 min Pages 自动重建。验证：
curl -sI https://pluto235.github.io/dft-agent-reports/new-report-2026-MM-DD.html | head -1
# 期待 HTTP/2 200
```

**坑提醒**：

- 必须改 `index.html`，否则首页不显示新报告（虽然 URL 直接访问能看到）
- 文件名不要带空格 / 中文 / 特殊字符 — 用 ASCII 减号分隔
- 不要传**外部资源依赖**的 HTML（要求 self-contained：CSS / JS / 图片全部 inline 或 data: URI）
- 仓库是 **public**，会被搜索引擎索引 — 不要发包含敏感数据 / API key 的报告

## GitHub Pages 配置

- 已启用 2026-05-12，模式 **Deploy from a branch**（`main` / `/ root`）
- 入口：https://github.com/Pluto235/dft-agent-reports/settings/pages
- 重建机制：push 到 main 触发，1-2 min 完成。无需手工操作。

## 何时需要重新考虑架构

当前是 **手维护 index.html** + 单层目录结构。报告数 ≤ 10-15 都好用。当：

- 报告数 > 20 → 考虑加 GitHub Actions workflow：扫描 `*.html` 按日期 sort 自动重生成 index
- 报告分多个系列（baseline / solver-v3 / solver-v4 / ablations）→ 按子目录组织
- 想要 tag / search → 转 Jekyll 或 11ty 等 SSG（届时要删掉 `.nojekyll`）

目前 ≤ 5 份不需要折腾。
