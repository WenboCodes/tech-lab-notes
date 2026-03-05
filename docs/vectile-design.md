# A2/A3 VecTile 设计文档

## 1. 背景与 PTO-ISA 本质分析

在 A2/A3 后端架构中，**PTO-ISA**（特指 `pto-isa` 仓）的核心职能是实现从 **UB/L1 存储至向量计算单元 (Vector Length, VL)** 的自动化分块 (Tiling)。

**VecTile 的设计初衷在于实现“调度算法”与“指令映射”的深度解耦：**

* **调度与执行的解耦**：现有的 PTO 实现旨在将分块调度算法 (Tiling Scheduling) 与底层硬件指令的映射 (Mapping) 逻辑剥离。调度器不再直接操作底层硬件寄存器，而是面向一套抽象接口进行逻辑编排。
* **虚拟指令集抽象**：本文档引入了 **VecTile** 的概念。通过将底层硬件指令接口封装为虚拟指令，为上层调度器提供了一致的语义视图。
* **架构收益**：这种设计使得调度策略的优化（如循环展开、掩码管理）能够独立于底层硬件指令集的物理细节，极大地提升了算子库的可维护性与跨架构迁移能力。

### 1.1 模块关系总览（ASCII）

```text
┌────────────────────────────────────────────────────┐
│                      PTO-ISA                       │
│  ┌──────────────────────────────────────────────┐  │
│  │ Scheduler [TBinOp.hpp, TBinSOp.hpp]         │  │
│  └──────────────────────────────────────────────┘  │
│                     |                              │
│                     v                              │
│               VecTile / VecIssue                  │
│                     |                              │
│                     v                              │
│  ┌──────────────────────────────────────────────┐  │
│  │ Wrapper [vtile_common.hpp, vtile_bin.hpp]   │  │
│  └──────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────┘
                       |
                       v
      CCE Intrinsics (vadd / vsub / vmul / vdiv)
```

---

## 2. 核心概念

### 什么是 VecTile？

`VecTile` 是一个语义化的 **2D 视图**，用于描述 UB 内存中的数据布局。它采用行主序（Row-Major），包含逻辑形状和步长信息。

```cpp
template <typename T>
struct VecTile {
    __ubuf__ T *ptr;    // UB 基地址
    uint16_t rows;      // 行数
    uint16_t cols;      // 列数
    uint16_t ld;        // 行步长 (Leading Dimension，以元素数为单位)
};

```

**关键约束：**

* 必须满足 $ld \ge cols$。
* 物理对齐要求：$ld \times sizeof(T)$ 必须满足 **32B 对齐**。

### 什么是 VecTileDesc？

`VecTileDesc` 是底层的 **硬件视图**，直接对应 CCE 指令的步长（Stride）语义。

```cpp
template <typename T>
struct VecTileDesc {
    __ubuf__ T *ptr;
    uint8_t blk_stride;     // 块步长 (以 32B Block 为单位)
    uint16_t rep_stride;    // 重复步长 (以 32B Block 为单位)
};

```

**映射逻辑：**

* 步长单位由元素转换为 32B 块。
* 当前实现中 `blk_stride` 固定为 1。
* 计算公式：$rep\_stride = (ld \times sizeof(T)) / 32$。

### 什么是 VecIssue？

`VecIssue` 描述单次指令发射的执行模式。

```cpp
struct VecIssue {
    uint8_t repeats;    // 重复次数
    bool count_mode;    // 是否开启 count-mode
};

```

**支持模式：**

* `Repeat(r)`：普通重复模式，执行 $r$ 次迭代。
* `CountMode()`：元素计数模式，指令的 `repeats` 参数将被映射为 0。

---

## 3. TADD 完整流程

### 3.1 调用链概览

从用户侧逻辑到最终硬件指令的完整路径如下：

```text
用户代码
  |
  v
TADD(dstTile, src0Tile, src1Tile, event...)  <-- 公开 API 入口
  |
  v
MAP_INSTR_IMPL(TADD, ...)                     <-- 宏展开与静态分发
  |
  v
TADD_IMPL(...)                                <-- 类型/形状预校验
  |
  v
TAdd<...>                                     <-- UB 地址计算与解析
  |
  v
BinaryInstr<AddOp<...>>                       <-- Scheduler 执行策略决策
  |
  v
VecTile + VecIssue                            <-- 构建虚拟发射描述
  |
  v
AddOp::BinInstr                               <-- 转发至指令 Wrapper
  |
  v
vtile::VAdd                                   <-- Wrapper 语义解析
  |
  v
LowerRowRepeat(VecTile) -> VecTileDesc        <-- Lowering：逻辑视图转硬件视图
  |
  v
vadd(...)                                     <-- 发射 CCE 指令/Intrinsic

```

### 3.2 VecTile 到 VecTileDesc 的 Lowering

当前系统主要支持 **RowRepeat** 模式（按行重复）。

**Lowering 规则示意图：**

```text
VecTile (逻辑视图)              VecTileDesc (硬件视图)
┌─────────────────┐             ┌─────────────────┐
│ ptr: 0x1000     │             │ ptr: 0x1000     │
│ rows: 16        │    ───>     │ blk_stride: 1   │
│ cols: 64        │             │ rep_stride: 8   │
│ ld: 64          │             └─────────────────┘
└─────────────────┘
                                rep_stride = (64 * 4) / 32 = 8

```

**关键常量定义：**

* `REPEAT_BYTE = 256`：单次 repeat 处理的字节容量。
* `BLOCK_BYTE_SIZE = 32`：硬件基本块大小。
* `REPEAT_MAX = 255`：硬件支持的最大迭代次数。
* `REPEAT_STRIDE_MAX = 255`：硬件支持的最大重复步长。
* `VL = REPEAT_BYTE / sizeof(T)`：各类型对应的向量长度。

**Bin 编程约束（由硬件边界推导）：**

| 硬件约束 | 约束公式 | 软件编程约束 | 软件处理策略 |
| --- | --- | --- | --- |
| `REPEAT_STRIDE_MAX = 255` | `rep_stride = (ld * sizeof(T)) / 32 <= 255` | 必须保证 `ld >= cols` 且 `ld * sizeof(T)` 为 32B 对齐，并满足 `ld <= (255 * 32) / sizeof(T)` | 入口参数校验；超限属于非法输入，不能靠普通 repeat 绕过（wrapper 会在 `AssertRepeatStride` 处拦截） |
| `REPEAT_MAX = 255` | 单条指令 `repeats <= 255` | norm-mode 单次发射不能超过 255 次 repeat | 当 `totalRepeats > 255` 时拆分循环，或切换 count-mode（`repeats = 0` + `SetVectorCount`） |
| `REPEAT_BYTE = 256` | `VL = 256 / sizeof(T)` | `validCol` 非 VL 对齐时不能直接全量按 norm-mode 发射 | 使用 tail mask（`SetContMask*`）处理尾部，或切换 count-mode |

**按数据类型的 `ld` 上限（由 `REPEAT_STRIDE_MAX` 推导）：**

* `b32`: `ld <= 2040`
* `b16`: `ld <= 4080`
* `b8`: `ld <= 8160`

---

## 4. Scheduler 和 Wrapper 的职责划分

### 4.1 Scheduler (调度层)

位于 `TBinOp.hpp`，负责 **“如何分块”** 的全局调度逻辑：

1. **生命周期控制**：管理 Count-mode 的上下文切换，负责 Mask 的保存与恢复。
2. **执行策略决策**：基于形状、对齐及步长特征，选择最优执行路径（Fast Path vs. Tail Path）。
3. **循环编排**：处理 `Repeat > 255` 的拆分、非连续数据的行间循环及尾部逻辑处理。

### 4.2 Wrapper (指令包装层)

位于 `vtile_common.hpp` 和 `vtile_bin.hpp`，负责 **“如何映射”** 底层指令：

1. **视图降低 (Lowering)**：将 `VecTile` 转化为 `VecTileDesc`。
2. **参数映射**：将抽象的 `VecIssue` 语义映射为指令级的 `repeats`（例如：`count_mode -> repeats = 0`）。
3. **约束校验**：在发射前执行 `AssertRepeatStride` 等硬件边界检查。
4. **硬件接口调用**：封装底层 CCE 指令集（`vadd`, `vsub`, `vmul` 等）。

> **原则**：Wrapper 不参与 Mask/Count 的状态管理，也不执行任何循环拆解工作。

---

## 5. TADD Scheduler 调度策略

### 相同步长路径 (Same-Stride Path)

当算子的所有操作数行步长相同时，调度策略如下：

```text
开始
  |
  v
是否为小形状？ (rows <= 255 && cols < VL)
  |
  ├─ 是 ──> Bin1LNormModeSmall (单层循环小形状优化)
  |
  └─ 否 ──> 是否满足编译期连续？ (Cols == ValidCol || Rows == 1)
              |
              ├─ 是 ──> BinaryInstrFastPath (连续空间快速路径)
              |
              └─ 否 ──> 运行时判断
                          |
                          v
                        是否运行时连续？
                          |
                          ├─ 是 ──> 需要 count-mode？
                          |           |
                          |           ├─ 是 ──> Bin1LCountMode
                          |           └─ 否 ──> Bin1LNormMode
                          |
                          └─ 否 ──> 选择二层循环策略
                                      |
                                      ├─ Bin2LCountMode
                                      ├─ Bin2LNormModeColVLAlign
                                      └─ Bin2LNormModeRowRpt

```

**Count-mode 自动触发条件：**

* **非 VL 对齐**：长行且 `validCol % VL != 0`。
* **量级限制**：总迭代次数 `totalRepeats > REPEAT_MAX`。

### 5.2 Unary 相对 Bin 的调度差异点

在调度目标上，Unary 与 Bin 都是“优先连续 fast path，必要时切换 count-mode 或 2L 路径”；但实现拆分和分支命名不同：

1. **分支框架不同**：Unary 走 `Unary1LNormMode/Unary1LCountMode/Unary2LProcess`，Bin 走 `Bin1L*/Bin2L*`。
2. **小形状优化命名不同**：Bin 有显式 `Bin1LNormModeSmall`，Unary 没有对应命名分支。
3. **路径决策参数不同**：Unary 主要围绕 `isCombined`、`totalRepeats` 和 `validCol` 对齐状态切换 `Unary2LCountMode/Unary2LNormModeColVLAlign/Unary2LNormModeRowRpt`。
4. **wrapper count-mode 分支行为不同**：Unary 在 `vtile_unary.hpp` 中存在 `if (is.count_mode)` 的 direct intrinsic 分支；Bin 统一走 `PrepareBinaryIssue` 后发射。

---

## 6. Count-Mode 使用指南

Count-mode 是一种特殊的硬件执行模式，专门用于处理 **非对齐数据** 或 **超大重复次数**。

### 使用范式

在 Count-mode 下，`repeats` 参数失效（置 0），硬件依据 `SetVectorCount` 设置的值进行处理，并需配合特定的 Mask 模式。

```cpp
using namespace pto::npu::a2a3::vtile;

// 定义逻辑 Tile
VecTile<float> dst{dstPtr, 1, cols, ld};
VecTile<float> src0{src0Ptr, 1, cols, ld};
VecTile<float> src1{src1Ptr, 1, cols, ld};

BeginCountMode(validCount);          // 开启上下文：切换 Mask 并设置 Count
VAdd(CountMode(), dst, src0, src1);  // 虚拟指令发射
EndCountMode<float>();               // 恢复上下文

```

---

## 7. 参数流转与典型场景

### 7.1 流转阶段

1. **Stage 1 (Tile API)**: 解析 UB 指针及有效形状。
2. **Stage 2 (Scheduler Issue Build)**: 构建 `VecTile` 逻辑描述。
3. **Stage 3 (Scheduler Issue Mode)**: 确定发射模式 (`Repeat` or `CountMode`)。
4. **Stage 4 (Wrapper Lowering)**: 计算 `rep_stride` 生成 `VecTileDesc`。
5. **Stage 5 (Wrapper Repeat Map)**: 转换硬件 `repeats` 参数。
6. **Stage 6 (Wrapper Call)**: 最终发射硬件指令。

### 7.2 场景示例 (以 TADD FP32 为例)

| 场景 | 特征描述 | 执行路径 | 关键参数映射 |
| --- | --- | --- | --- |
| **S1: 标准全量 Tile** | 64x64，完全连续 | `Bin1LNormMode` | `rpt=64`, `rep_stride=8` |
| **S2: FP16 大行** | 16x256，行内连续 | `Bin1LNormMode` | `rpt=32`, `rep_stride=8` |
| **S3: 包含 Tail 分支** | 16x31，非对齐且有剩余 | `Bin2LNormModeRowRpt` | `rpt=16`, `mask=31`, `rep_stride=8` |
| **S4: 异构行步长** | dst/src0 步长不同 | `Bin2LNormModeRowRpt` | `rpt=16`, `dst_rep=4, src0_rep=8` |
| **S5: 计数触发** | 16x96，非对齐长行 | `Bin1LCountMode` | `rpt=0`, `vcount=1536`, `rep_stride=12` |

---

## 8. Expand / Reduce 相对 Bin 的差异设计

### 8.1 差异总表（以 Bin 为基线）

| 对象 | 基础数据结构 | 处理流程 | 编程约束 |
| --- | --- | --- | --- |
| `Bin` | `VecTile + VecIssue`，3 输入统一走 `PrepareBinaryIssue` | `Bin1L* / Bin2L*` 路径，tail 与 count-mode 统一调度 | `ld` 需 32B 对齐，`rep_stride <= 255` |
| `Expand(算术)` | 仍用 `VecTile + VecIssue`，但广播输入走扩展语义；wrapper 用 `PrepareScalarIssue` 仅降低 `dst/src0` | `TRowExpandBinOp` / `TColExpandBinOp` 调度到 `vtile_expand.hpp` | 受广播轴、`vbrcb` 预展开与 count-mode 切换条件约束 |
| `Expand(纯扩展)` | 不走 `VecIssue`，直接以 tile 指针与 valid shape 进行复制/扩展 | `TRowExpand.hpp`（`vector_dup`/`vbrcb`）或 `TColExpand.hpp`（`copy_ubuf_to_ubuf`） | 快路径依赖静态 shape 与 dtype；不经过 `vtile_expand` |
| `RowReduce` | 不以 `VecTile + VecIssue` 为主；采用 `TRowReduceOp` + `ReduceInstrImpl/GroupReduceInstrImpl/BinInstrImpl` | 先尝试 FP32 特化，否则 `FillTmp -> TmpProc -> FinalReduce` | 输入类型仅 `half/float`；输出布局需 ND 或 DN 且 `col=1` |
| `ColReduce` | 不以 `VecTile + VecIssue` 为主；采用 `TColReduceOp` 与 `ColReduceInstrByMode` | 先拷首行，再按 full-repeat 与 remain-tail 两段规约 | 输入输出都需 ND row-major，且 `SrcValidCol == DstValidCol` |

### 8.1.1 五类对象端到端示例（Tile 如何 lower 到 VecTile）

1. **`Bin` 示例：`TADD(dst, src0, src1)`**

```text
Tile API: TADD(dstTile, src0Tile, src1Tile)
  -> TADD_IMPL / TAdd (TAdd.hpp)
  -> BinaryInstr (TBinOp.hpp)
  -> MakeVecTile(dst/src0/src1) + VecIssue
  -> vtile::VAdd (vtile_bin.hpp)
  -> PrepareBinaryIssue + LowerRowRepeat (vtile_common.hpp)
  -> VecTileDesc(dst/src0/src1) -> vadd(...)
```

2. **`Expand(算术)` 示例：`TROWEXPANDADD(dst, src0, src1)`**

```text
Tile API: TROWEXPANDADD(dstTile, src0Tile, src1Tile)
  -> TROWEXPANDADD_IMPL (TRowExpandAdd.hpp)
  -> TRowExpandBin / EmitRowExpand (TRowExpandBinOp.hpp)
  -> MakeVecTile(dst/src0/src1) + VecIssue
  -> vtile::VRowExpandAdd (vtile_expand.hpp)
  -> PrepareScalarIssue + LowerRowRepeat(dst/src0) (vtile_common.hpp)
  -> broadcast src1 + vadd(...)
```

说明：算术 Expand 也会 lower 到 `VecTile`，但 `src1` 是广播语义，不按 Bin 第三输入做同形 lowering。

3. **`Expand(纯扩展)` 示例：`TROWEXPAND(dst, src)`**

```text
Tile API: TROWEXPAND(dstTile, srcTile)
  -> TROWEXPAND_IMPL (TRowExpand.hpp)
  -> TRowExpand / TRowExpandBrcb
  -> 直接解析 tile 指针与 validRow/validCol
  -> vector_dup(...) 或 vbrcb(...)
```

说明：纯扩展路径**不经过 `VecTile/VecIssue`**，因此不存在 `Tile -> VecTileDesc` 这一 lowering。

4. **`RowReduce` 示例：`TROWSUM(dst, src, tmp)`**

```text
Tile API: TROWSUM(dstTile, srcTile, tmpTile)
  -> TROWSUM_IMPL / TRowSum (TRowSum.hpp)
  -> TRowReduceInstr (TRowReduceOps.hpp)
  -> FillTmp -> TmpProc -> FinalReduce
  -> vcadd / vcgadd / vadd ...
```

说明：RowReduce 使用专用规约框架，不经过 `VecTile` wrapper lowering。

5. **`ColReduce` 示例：`TCOLSUM(dst, src)`**

```text
Tile API: TCOLSUM(dstTile, srcTile)
  -> TCOLSUM_IMPL / TColSum (TColSum.hpp)
  -> copy 首行到 dst
  -> BinarySum 或 SequentialSum
  -> vadd / copy_ubuf_to_ubuf ...
```

说明：ColReduce 同样走专用框架，不经过 `VecTile` wrapper lowering。

### 8.2 Expand 相对 Bin 的差异

#### 8.2.1 基础数据结构差异

1. **算术 Expand (`TRowExpandBinOp.hpp` / `TColExpandBinOp.hpp`)**：仍构造 `VecTile` 与 `VecIssue`，但第三输入 `src1` 是广播源，不再等价于 Bin 的“同形第三输入”。
2. RowExpand 中 `src1` 的 tile 形状/步长可与 `dst/src0` 不同（例如仅保留广播宽度），与 Bin 第三输入语义不同。
3. ColExpand 中 `src1` 被视为 `1 x cols` 广播源。
4. `vtile_expand.hpp` 的 wrapper 通过 `PrepareScalarIssue` 只降低 `dst/src0`，`src1` 按广播参数参与 intrinsic。
5. **纯扩展 (`TRowExpand.hpp` / `TColExpand.hpp`)**：不使用 `VecIssue`，直接使用 tile 指针、validRow、validCol 驱动复制/扩展。

#### 8.2.2 处理流程差异

```text
Expand 输入
  -> 路径选择: 算术Expand or 纯扩展
  -> 算术Expand: TRowExpandBinOp/TColExpandBinOp
  -> (可选) RowExpand 对 src1 先做 vbrcb 到 tmp UB
  -> VecTile + VecIssue -> vtile_expand(VRowExpand*/VColExpand*)
  -> 根据 stride/代价切换 norm-mode 或 count-mode
  -> 纯扩展: TRowExpand.hpp(vector_dup/vbrcb) 或 TColExpand.hpp(copy_ubuf_to_ubuf)
```

#### 8.2.2.1 vbrcb 在行向广播中的关键作用 ⭐

**问题背景**：实现 `dst[i,j] = src0[i,j] + src1[i]`（行向广播加法）

- 硬件vadd指令不支持"每行用一个标量"
- vadd的stride单位是32B块（8个FP32元素）
- 需要将240个标量转换为适合vadd的格式

**vbrcb的官方定义**：

> 获取src中8个b16/b32元素，**将每个元素单独广播成一个32B的block**，然后将8个block连续写入dst。
> - 对于b32（FP32）：每个元素广播为8个相同值（8×4B = 32B）
> - 对于b16（FP16）：每个元素广播为16个相同值（16×2B = 32B）

**转换过程示例（FP32）**：

```
输入 src1（240个标量）：
[a0, a1, a2, a3, a4, a5, a6, a7, a8, ..., a239]

vbrcb(tmpPtr, src1, 1, 8, 30);  // 30个repeat

Repeat 0 (src1[0:7]):
  a0 → [a0, a0, a0, a0, a0, a0, a0, a0]  (32B block)
  a1 → [a1, a1, a1, a1, a1, a1, a1, a1]  (32B block)
  ...
  a7 → [a7, a7, a7, a7, a7, a7, a7, a7]  (32B block)
  → tmpPtr[0:63]

Repeat 1 (src1[8:15]):
  a8 → [a8×8], a9 → [a9×8], ..., a15 → [a15×8]
  → tmpPtr[64:127]

...

总计：240个标量 → 1920个元素（240 × 8）
```

**tmpPtr的内存布局**：

```
偏移0-7:    [a0, a0, a0, a0, a0, a0, a0, a0]   ← a0的block
偏移8-15:   [a1, a1, a1, a1, a1, a1, a1, a1]   ← a1的block
偏移16-23:  [a2, a2, a2, a2, a2, a2, a2, a2]   ← a2的block
...
偏移56-63:  [a7, a7, a7, a7, a7, a7, a7, a7]   ← a7的block
偏移64-71:  [a8, a8, a8, a8, a8, a8, a8, a8]   ← a8的block
...
```

**vadd如何使用tmpPtr**：

```
vadd(dst, src0, tmpPtr, rpt=240, ..., src1_rep_stride=1)

Repeat 0: 读取tmpPtr[0:7] = [a0×8]，使用a0广播到第0行64列
Repeat 1: 读取tmpPtr[8:15] = [a1×8]，使用a1广播到第1行64列
...
Repeat 239: 读取tmpPtr[1912:1919] = [a239×8]，使用a239广播到第239行64列
```

**为什么需要vbrcb？**

1. **硬件限制**：vadd的stride单位是32B块，最小步进是8个FP32元素
2. **完美适配**：vbrcb将每个标量扩展为一个32B块（8个相同值）
3. **stride对齐**：vadd每次前进8个元素（1个32B块），读取到的8个元素都相同
4. **实现行向广播**：每行使用一个标量，完美实现 `dst[i,:] = src0[i,:] + src1[i]`

**关键理解**：vbrcb是**元素级广播**，不是块级重复！
- ❌ 错误：`[a0, a1, a2, a3, a4, a5, a6, a7]` 整体重复8次
- ✅ 正确：每个元素单独广播为一个32B block

#### 8.2.3 编程约束差异

1. **算术 Expand 约束**：`TRowExpandBinOp` 在 `repeatStrideOverflow` 或”列向 repeat 开销大于按行 count”时优先切到 count-mode。
2. 非 row-major 的广播输入会进入 `vbrcb` 预展开分支（使用 tmp UB）。
   - **vbrcb作用**：将每个标量扩展为一个32B block（8个相同值），使得vadd可以以32B为单位步进
   - **内存需求**：8KB临时UB，可存放32个repeat（每个repeat 256B）
   - **分批处理**：通常每批处理30个repeat（240行），避免临时UB溢出
3. `TColExpandBinOp` 在连续场景走 norm-mode，非连续场景按行循环走 count-mode。
4. **纯扩展约束**：`TRowExpand.hpp` 的广播快路径依赖静态 shape 与 dtype（b16/b32）条件。
5. `TColExpand.hpp` 要求输入输出 ND row-major 且 `SrcValidCol == DstValidCol`。
6. 纯扩展路径不经过 `BeginCountMode/EndCountMode` + `VecIssue` 这套接口层。

### 8.3 Reduce 相对 Bin 的差异

#### 8.3.1 基础数据结构差异

1. Reduce 不以 `VecTile + VecIssue` 作为主调度载体，而是走专用框架。
2. `TRowReduceOps.hpp`：`TRowReduceOp` 聚合 `ReduceInstrImpl + GroupReduceInstrImpl + BinInstrImpl`。
3. `TColReduceOps.hpp`：`TColReduceOp` 通过 `ColReduceInstrByMode` 执行列向分段规约。
4. 这与 Bin 的“统一 issue + wrapper lowering”模型存在结构性差异。

#### 8.3.2 处理流程差异

```text
Reduce 输入Tile
  -> RowReduce? (TRowReduceOps.hpp)
      -> TryOptimizeFP32Reduce 命中: 走特定shape优化链
      -> 否则: FillTmp -> TmpProc -> FinalReduce
      -> 内部混合 ReduceInstrImpl / GroupReduceInstrImpl / BinInstrImpl
  -> ColReduce? (TColReduceOps.hpp)
      -> copy首行到dst
      -> ColReduceInstrByMode(full-repeat)
      -> ColReduceInstrByMode(remain-tail + mask)
```

#### 8.3.3 编程约束差异

1. **RowReduce 约束**：输入类型限制为 `half/float`。
2. 输出布局要求 ND 或 DN 且 `col=1`。
3. `validRow` 必须与输出 `validRow` 对齐；`validRow/validCol` 不能为 0。
4. **ColReduce 约束**：输入类型支持 `half/float/int16/int32`。
5. 输入输出必须为 ND row-major。
6. 必须满足 `SrcValidCol == DstValidCol`，否则不满足接口前置条件。

---

## 9. 开发注意事项

### 资源状态管理

* **Mask 管理**：执行 `SetContMask` 后必须对应执行 `SetFullVecMask` 以清理状态。
* **Count 模式管理**：`BeginCountMode` 与 `EndCountMode` 必须成对出现，防止 Mask 污染。

### 硬件约束校验

* **对齐底线**：`VecTile.ld` 物理地址必须满足 32B 对齐。
* **步长上限**：确保 $rep\_stride \le 255$，若超出此限制，必须在 Scheduler 层拆解为多层循环。

## 10. 总结

本设计通过 **语义化视图 (VecTile)** 与 **物理视图 (VecTileDesc)** 的分层，构建了 A2/A3 后端高效、解耦的向量计算框架：

1. **VecTile** 作为逻辑纽带，解除了算法对硬件细节的依赖。
2. **Scheduler** 专注于全局效率优化与策略分发。
3. **Wrapper** 确保指令发射的规范性与安全性。

---
