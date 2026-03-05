# PTO Docs

<div class="home-hero" markdown>

PTO intrinsic / IR 笔记与设计文档集合，侧重可落地的语义解释与边界约束。

[从 VecTile 开始](vectile-design.md){ .md-button .md-button--primary }
[vbrcb 指令指南](vbrcb-instruction-guide.md){ .md-button }
[PTO IR Manual](pto-ir-manual.md){ .md-button }

</div>

## 核心入口

<div class="grid cards" markdown>

-   ### VecTile 设计

    A2/A3 后端的 tiling 抽象：调度与指令映射解耦、Lowering 规则、repeat/stride 边界。

    **适合**：想快速建立 PTO-ISA “在做什么 / 为什么这样做” 的整体直觉  
    **关键词**：`VecTile` / `VecIssue` / `RowRepeat`

    [打开文档](vectile-design.md){ .md-button .md-button--primary }

-   ### vbrcb 指令指南

    `vbrcb` 的参数语义、布局转换、典型广播场景，以及 repeat 上限时的拆分策略。

    **适合**：需要写/看 UB 侧广播预处理代码  
    **关键词**：`repeat` / `dst_rep_stride` / `pipe_barrier`

    [打开文档](vbrcb-instruction-guide.md){ .md-button }

-   ### PTO IR Manual

    PTO IR 的字段语义、约束与约定用法，作为实现和调试时的快速查表。

    **适合**：做 IR 相关开发、review、排查问题  
    **关键词**：IR 结构 / memory / event

    [打开手册](pto-ir-manual.md){ .md-button }

</div>

## 推荐阅读顺序

!!! tip "建议从上到下读一遍"
    1. `VecTile`：先建立整体抽象和 lowering 的直觉
    2. `vbrcb`：再看具体指令在 UB 上怎么落地
    3. `PTO IR Manual`：最后用手册补齐细节和约束

## 本地预览

```bash
cd /data/sunwenbo/pto/tech-lab-notes
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
mkdocs serve
```

```text
http://127.0.0.1:8000
```

## 线上地址

```text
https://wenbocodes.github.io/tech-lab-notes/
```

## 约定

- 新增页面：放在 `docs/`，文件名用 kebab-case，并同步更新 `mkdocs.yml` 的 `nav`。
- 合并前自检：`mkdocs build --strict`。
