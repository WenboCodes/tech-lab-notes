# PTO Docs (GitHub Pages)

这个目录已经整理成可直接发布到 GitHub Pages 的 MkDocs 项目。

## 目录结构

- `docs/`: 文档正文
- `mkdocs.yml`: 站点配置与导航
- `requirements.txt`: 构建依赖
- `.github/workflows/deploy-docs.yml`: 自动发布工作流

## 本地预览

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
mkdocs serve
```

默认访问 `http://127.0.0.1:8000`。

## GitHub Pages 发布

1. 把当前目录内容提交到 GitHub 仓库根目录（默认分支 `main`）。
2. 在仓库设置中打开 `Settings -> Pages`。
3. 在 `Build and deployment` 里将 `Source` 设为 `GitHub Actions`。
4. 推送到 `main` 后，工作流 `deploy-docs` 会自动构建并发布。

## 文档入口

站点首页由 `docs/README.md` 提供，导航由 `mkdocs.yml` 管理。
