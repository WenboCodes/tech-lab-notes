# Wiki: 手动修改首页并发布上线

这份文档用于记录 `tech-lab-notes` 的首页修改与发布流程，适合日常手动维护。

## 1. 进入仓库

```bash
cd /data/sunwenbo/pto/tech-lab-notes
```

## 2. 修改首页文件

站点首页是 `docs/README.md`（不是仓库根目录 `README.md`）。

```bash
vim docs/README.md
```

或：

```bash
nano docs/README.md
```

如果你同时调整导航顺序，需要再编辑：

```bash
vim mkdocs.yml
```

## 3. 本地预览（推荐）

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
mkdocs serve
```

预览地址：

```text
http://127.0.0.1:8000
```

## 4. 提交并推送

```bash
git add docs/README.md mkdocs.yml
git commit -m "docs: optimize homepage"
git push origin main
```

## 5. 等待自动部署

仓库已配置 GitHub Actions 工作流 `deploy-docs`，推送到 `main` 会自动构建并发布。

查看最近一次运行：

```bash
gh run list -R WenboCodes/tech-lab-notes --workflow deploy-docs --limit 1
```

查看指定运行详情：

```bash
gh run view <run_id> -R WenboCodes/tech-lab-notes
```

## 6. 线上访问地址

```text
https://wenbocodes.github.io/tech-lab-notes/
```

## 7. 常见问题

1. 页面没更新  
先看 Actions 是否成功；其次检查是否推到了 `main` 分支。

2. 首页改了但导航没变  
确认 `mkdocs.yml` 的 `nav` 是否同步更新。

3. 本地预览失败  
确认虚拟环境已激活，并重新执行 `pip install -r requirements.txt`。
