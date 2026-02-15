# A3 版 TROWMAX 详细分析（面向小白到进阶）

## 0. 这份文档讲什么
这份文档只讲 **A3（A2/A3 共用后端）** 的 `TROWMAX`。

目标是把这件事讲清楚：
1. `TROWMAX` 在数学上做什么。
2. 在 A3 上为什么实现复杂。
3. PTO 代码里到底是怎么调度 `vcmax / vmax / vcgmax` 的。
4. 哪些路径是性能关键（尤其是静态 shape 快路径）。
5. 它和你正在做的 Bin Intrinsic 思路有什么共性。

---

## 1. 一句话理解 TROWMAX
`TROWMAX` 就是：**每一行取最大值**。

输入（`m x n`）：

```text
1  7  3
4  2  9
```

输出（`m x 1`）：

```text
7
9
```

---

## 2. 为什么实现“看起来很重”
算法简单，但 A3 是向量硬件，需要按硬件粒度发射指令：

1. 一次 repeat 对应固定 256B 数据窗口。
2. repeat 次数有编码上限（255）。
3. 行长度经常不是 256B 整数倍，尾部要 mask。
4. stride 也有编码限制，超限时要降级成更细粒度循环。
5. 行规约不是一次指令完成，通常是“分段规约 + 分段合并 + 最终收敛”。

所以复杂度来自“调度硬件”，不是“max 这个数学操作”。

---

## 3. A3 端代码调用链（从 API 到 intrinsic）

1. 用户接口：
`TROWMAX(dst, src, tmp)`  
来源：`include/pto/common/pto_instr.hpp:714`

2. A3 入口：
`TROWMAX_IMPL`  
来源：`include/pto/npu/a2a3/TRowMax.hpp:55`

3. 规约调度核心：
`TRowReduceInstr<TRowMaxOp,...>`  
来源：`include/pto/npu/a2a3/TRowMax.hpp:51`

4. 实际底层指令映射：
- `ReduceInstrImpl -> vcmax`（段内规约）  
  `include/pto/npu/a2a3/TRowMax.hpp:30`
- `BinInstrImpl -> vmax`（段间合并）  
  `include/pto/npu/a2a3/TRowMax.hpp:20`
- `GroupReduceInstrImpl -> vcgmax`（分组规约）  
  `include/pto/npu/a2a3/TRowMax.hpp:35`

---

## 4. 关键参数（A3 路径里真正控制调度的量）
在 `TRowReduceInstr` 中，核心参数定义如下：
来源：`include/pto/npu/a2a3/TRowReduceOps.hpp:272`

1. `elemPerBlock = 32 / sizeof(T)`
2. `elemPerRpt = 256 / sizeof(T)`
3. `srcRptPerRow = validCol / elemPerRpt`
4. `remain = validCol % elemPerRpt`
5. `rowRptTimes = validRow / REPEAT_MAX`

直观理解：
- `srcRptPerRow`：每行有多少个完整 repeat 块
- `remain`：每行尾部不满一个 repeat 的元素数
- `rowRptTimes`：行数是否超过 255，需要按块拆分发射

---

## 5. 约束检查（为什么有些 Tile 会直接报错）
来源：`include/pto/npu/a2a3/TRowReduceOps.hpp:176`

A3 `TROWMAX` 主要限制：
1. `src/dst` 必须是 `Vec Tile`
2. `src` 必须是 ND（row major + NoneBox）
3. `dst` 必须是 ND 或 DN(col=1)
4. dtype 只支持 `half/float`
5. `src.validRow == dst.validRow`
6. `validRow/validCol` 不能为 0

这也是你在业务侧看到“看起来合法但跑不通”的主要来源。

---

## 6. A3 通用算法路径（按场景拆解）
核心函数：`TRowReduceInstr`  
来源：`include/pto/npu/a2a3/TRowReduceOps.hpp:267`

### 场景 A：`validCol <= elemPerRpt`
走 `OneRepeatProc`：
来源：`include/pto/npu/a2a3/TRowReduceOps.hpp:198`

- 如果 `validCol == elemPerRpt`：直接走 count 模式规约，批量处理行。
- 如果 `validCol < elemPerRpt`：先设置连续 mask，再按行块（最多 255 行一组）做规约。

这部分就是“只有 1 个 repeat（或不足 1 个 repeat）”的处理。

### 场景 B：`elemPerRpt < validCol < 2*elemPerRpt`
来源：`include/pto/npu/a2a3/TRowReduceOps.hpp:288`

- 先把前一段完整 repeat 复制到 `tmp`
- 后续再用 tail/mask 和 `tmp` 合并

### 场景 C：`validCol >= 2*elemPerRpt`
来源：`include/pto/npu/a2a3/TRowReduceOps.hpp:299`

- `FillTmp`: 先比较前两段 repeat（初始化 tmp）
- `TmpProc`: 把第 3 段到最后一段逐次并入 tmp
- 最后对 tmp 再做一次规约写 dst

### 场景 D：存在 `remain`（尾块）
来源：`include/pto/npu/a2a3/TRowReduceOps.hpp:303`

- 先 `SetContinuousMask(remain)`
- 对每行尾段和 tmp 做 `vmax` 合并
- 恢复 full mask

---

## 7. A3 的 stride/repeat 超限处理机制
在 `ReduceInstrByMode / BinInstrByMode` 内，做了“超编码范围降级”：
来源：`include/pto/npu/a2a3/TRowReduceOps.hpp:120` 与 `:137`

1. 如果 stride 超限：
- 不再一次发 repeat
- 改成 `repeat=1` 的显式循环（每次手动更新指针）

2. 如果开启 count mode：
- `set_mask_count`
- `set_vector_mask` 指定总元素数
- intrinsic 用 `repeat=0` 发射

这套机制是 A3 通用路径稳健性的关键，也是代码看起来“分支很多”的根本原因。

---

## 8. 为什么需要 `tmp`
A3 的行规约不是一次 `vcmax` 就结束：
1. 先得到分段局部 max
2. 段与段继续合并
3. 最后收敛成行 max

`tmp` 就是中间合并缓冲区，避免每次都回写 dst 并重复读回。

在 A3 里它是实际参与计算的，不是“形式参数”。

---

## 9. 静态 shape 快路径（A3 性能核心）
函数：`TryOptimizeFP32Reduce`  
来源：`include/pto/npu/a2a3/TRowReduceOps.hpp:223`

触发条件：
1. `float`
2. `dst` 为 DN 1 列
3. shape 精确命中以下之一：
- `64x128`
- `32x256`
- `16x512`
- `8x1024`

对应函数：
- `ReduceOptFP32_64x128`（`:34`）
- `ReduceOptFP32_32x256`（`:54`）
- `ReduceOptFP32_16x512`（`:77`）
- `ReduceOptFP32_8x1024`（`:97`）

这些快路径会用更短的规约树（`vcgmax + vmax` 组合）直接收敛，显著减少通用循环开销。

---

## 10. 测试用例与路径覆盖关系（A3）
A3 `trowmax` 测试专门覆盖了：
来源：`tests/npu/a2a3/src/st/testcase/trowmax/gen_data.py:48`

1. 不满 repeat / 恰好 repeat / 多 repeat / 有 remainder
2. 大行数和非满有效行
3. DN 输出场景
4. 快路径 shape：
- case15: `64x128`
- case16: `32x256`
- case17: `16x512`
- case18: `8x1024`

这说明实现不是只做了“能跑”，而是按调度分支系统覆盖。

---

## 11. 和 Bin Intrinsic 设计的共性（你当前工作的连接点）
你现在做的 Bin Intrinsic 核心思想是：
**把最小可控粒度暴露出来（repeat/stride/mask），让上层调度自己决定循环。**

A3 的 `TROWMAX` 在思想上是同源的：
1. 底层也在围绕 repeat/stride/mask 做调度
2. 只是当前 `TROWMAX` 把调度逻辑包在 high-level 实现里了
3. 若未来要做 reduce low-level intrinsic API，本质上可复用同一套“参数契约 + 单次发射语义”

---

## 12. 小结（给产品/算法同学看的版本）
如果只记 4 句话：

1. `TROWMAX` = 每行取最大值。
2. A3 实现复杂是因为硬件发射规则（repeat/mask/stride），不是因为算法。
3. A3 通过 `vcmax + vmax + vcgmax + tmp` 完成分段规约与收敛。
4. A3 性能好的关键是：通用路径 + 4 个热点静态 shape 快路径。


---

## 13. case15~18 逐步执行示例（按快路径拆解）
先给统一结论：这 4 个 case 都命中 `TryOptimizeFP32Reduce` 的静态快路径（`float + DN输出 + 满shape`），所以不会走通用的 `for` 循环分块逻辑。  
来源：`include/pto/npu/a2a3/TRowReduceOps.hpp:223`

### case15: `64x128`（`ReduceOptFP32_64x128`）
来源：`include/pto/npu/a2a3/TRowReduceOps.hpp:34`

触发条件：`Rows=64, ValidRow=64, Cols=128, ValidCol=128`

| 阶段 | intrinsic | 关键参数 | 形状演进 |
|---|---|---|---|
| 1 | `vcgmax` | `repeat=ValidRow*2=128` | `64x128 -> 64x16` |
| 2 | `vmax` | `repeat=ValidRow/8=8` | `64x16 -> 64x8` |
| 3 | `vcgmax` | `repeat=ValidRow/8=8` | `64x8 -> 64x1` |

为什么快：
1. 固定 3 段规约树，无通用路径的行循环和尾块判断。
2. `repeat` 均在编码范围内，不触发降级发射。
3. 没有 remainder，不需要 mask 切换。

### case16: `32x256`（`ReduceOptFP32_32x256`）
来源：`include/pto/npu/a2a3/TRowReduceOps.hpp:54`

触发条件：`Rows=32, ValidRow=32, Cols=256, ValidCol=256`

| 阶段 | intrinsic | 关键参数 | 形状演进 |
|---|---|---|---|
| 1 | `vcgmax` | `repeat=ValidRow*4=128` | `32x256 -> 32x32` |
| 2 | `vmax` | `repeat=ValidRow/4=8` | `32x32 -> 32x16` |
| 3 | `vmax` | `repeat=ValidRow/8=4` | `32x16 -> 32x8` |
| 4 | `vcgmax` | `repeat=ValidRow/8=4` | `32x8 -> 32x1` |

为什么快：
1. 比通用路径少了“先填 tmp、再按段循环并入 tmp”的动态调度。
2. 两次 `vmax` 是固定位置合并，访存模式连续，流水更稳定。

### case17: `16x512`（`ReduceOptFP32_16x512`）
来源：`include/pto/npu/a2a3/TRowReduceOps.hpp:77`

触发条件：`Rows=16, ValidRow=16, Cols=512, ValidCol=512`

| 阶段 | intrinsic | 关键参数 | 形状演进 |
|---|---|---|---|
| 1 | `vcgmax` | `repeat=ValidRow*8=128` | `16x512 -> 16x64` |
| 2 | `vcgmax` | `repeat=ValidRow=16` | `16x64 -> 16x8` |
| 3 | `vcgmax` | `repeat=ValidRow/8=2` | `16x8 -> 16x1` |

为什么快：
1. 全程 `vcgmax` 规约树，步骤固定，控制流非常短。
2. 不走按 `srcRptPerRow` 的循环归并，减少指令调度和 barrier 负担。

### case18: `8x1024`（`ReduceOptFP32_8x1024`）
来源：`include/pto/npu/a2a3/TRowReduceOps.hpp:97`

触发条件：`Rows=8, ValidRow=8, Cols=1024, ValidCol=1024`

| 阶段 | intrinsic | 关键参数 | 形状演进 |
|---|---|---|---|
| 1 | `vcgmax` | `repeat=ValidRow*16=128` | `8x1024 -> 8x128` |
| 2 | `vcgmax` | `repeat=ValidRow*2=16` | `8x128 -> 8x16` |
| 3 | `vmax` | `repeat=ValidRow/8=1` | `8x16 -> 8x8` |
| 4 | `vcgmax` | `repeat=ValidRow/8=1` | `8x8 -> 8x1` |

为什么快：
1. 先用两段 `vcgmax` 快速“砍列数”，最后只做一次 `vmax` 对折。
2. `repeat=1` 的尾段是快路径内固定步骤，不是通用路径那种“被迫降级”的单次发射。

### 这 4 个 case 的共同性能原因（A3）
1. 编译期 shape 已知，运行时不需要分支判断 `repeat/remain/stride over`。
2. 不涉及 `SetContinuousMask` 的动态切换。
3. 指令序列短且固定，`pipe_barrier(PIPE_V)` 位置可预测，流水更稳定。
4. 测试用例专门绑定这 4 个 shape（`case15~18`），验证了快路径逻辑与结果正确性。  
来源：`tests/npu/a2a3/src/st/testcase/trowmax/gen_data.py:62` 与 `tests/npu/a2a3/src/st/testcase/trowmax/trowmax_kernel.cpp:146`
