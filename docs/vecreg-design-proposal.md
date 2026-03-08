# VecReg 设计提案：Reg-Block-Lane 抽象模型

## 1. 设计动机

### 1.1 当前VecTile设计的问题

当前的VecTile抽象在处理Expand（广播）操作时存在以下问题：

1. **语义不匹配**：VecTile是为"同形操作"设计的，但expand本质是"异形广播"
   ```cpp
   // Row Expand：src1是240个标量，但被表示为[240, 8]矩阵
   VecTile<float> src1{ptr, 240, 8, 8};  // 语义混乱！
   // 8不是真实的列数，而是为了适配32B对齐的填充
   ```

2. **抽象泄漏**：硬件细节（vbrcb、blockSizeElem）暴露到调度层
   ```cpp
   // Scheduler需要知道硬件相关的blockSizeElem
   VecTile<float> src1{ptr, 240, blockSizeElem, blockSizeElem};

   // Scheduler需要调用硬件指令vbrcb
   vbrcb(tmpPtr, src1Ptr, 1, 8, 30);
   ```

3. **概念复杂度高**：引入多个特殊概念（PrepareScalarIssue、vbrcb预处理、tmpPtr等）

4. **性能开销**：vbrcb导致8倍内存膨胀（240个标量 → 1920个元素）

5. **可读性差**：expand的实现逻辑分散在多个层次，难以理解

### 1.2 设计目标

提出一个新的抽象模型，使得：
- **虚拟指令集接口**：VecReg作为虚拟指令集暴露，每个操作对应一条虚拟指令
- **充分利用硬件能力**：VecReg操作直接映射硬件repeat能力（1-255次）
- **不做软件循环**：VecReg操作内部不做软件循环/tiling，只执行单次硬件指令
- **职责清晰分离**：Scheduler负责大规模循环（>255）和tiling策略，VecReg负责硬件指令映射
- 用户能够直观理解硬件结构
- 广播语义清晰明确
- 硬件细节被良好封装
- 类型安全，编译期区分不同操作
- 性能无退化

---

## 2. Reg-Block-Lane 抽象模型

### 2.1 核心概念

将256B的向量长度（VL）抽象为一个**向量寄存器（VecReg）**，类似CPU的SIMD寄存器：

```text
┌─────────────────────────────────────────────────────────┐
│                    VecReg (256B)                        │
├────────┬────────┬────────┬────────┬────────┬────────────┤
│ Block0 │ Block1 │ Block2 │ Block3 │ Block4 │ ... Block7 │
│  32B   │  32B   │  32B   │  32B   │  32B   │    32B     │
└────────┴────────┴────────┴────────┴────────┴────────────┘
    ↓
┌─────────────────────────────┐
│      Block (32B)            │
├──────┬──────┬──────┬────────┤
│Lane0 │Lane1 │Lane2 │ ...    │  FP32: 8 lanes
│ 4B   │ 4B   │ 4B   │        │  FP16: 16 lanes
└──────┴──────┴──────┴────────┘
```

**三层结构：**
1. **VecReg（向量寄存器）**：256B，单次repeat处理的数据量
2. **Block（块）**：32B，硬件stride的基本单位
3. **Lane（通道）**：单个数据元素，所有lane并行执行

### 2.2 虚拟指令集设计原则 ⭐

**核心原则：一条VecReg指令 = 一次硬件指令调用（可包含1-255次repeat）**

```cpp
// ✅ 正确：VecReg操作利用硬件repeat能力
VAdd(VecReg{ptr, 240, 64, 8}, ...);  // 执行1次vadd，硬件repeat=240

// ✅ 正确：VecReg操作处理单个VecReg
VAdd(VecReg{ptr, 1, 64, 8}, ...);    // 执行1次vadd，硬件repeat=1

// ❌ 错误：VecReg操作内部不应该有软件循环
void VAdd(...) {
    for (int i = 0; i < 240; i++) {  // ❌ 不应该在VecReg内部循环
        vadd(..., repeat=1, ...);
    }
}

// ✅ 正确：VecReg操作直接映射硬件repeat
void VAdd(VecReg reg, ...) {
    vadd(..., repeat=reg.num_regs, ...);  // ✓ 利用硬件repeat
}
```

**职责分离：**

| 层次 | 职责 | 示例 |
|-----|------|------|
| **Scheduler** | 大规模循环（>255）、tiling策略 | `for (i=0; i<1000; i+=255) VAdd(...)` |
| **VecReg Op** | 硬件指令映射（repeat ≤ 255） | `VAdd(240个VecReg, ...)` → vadd(repeat=240) |
| **Hardware** | 执行repeat | vadd intrinsic with repeat |

**设计理念：**
- VecReg是**虚拟指令集**，直接映射硬件能力
- 每个VecReg操作对应**一次硬件指令调用**
- 充分利用硬件repeat能力（1-255次）
- 当需要 > 255次时，Scheduler负责拆分
- 类似CPU SIMD：`_mm512_add_ps`处理1个寄存器，但可以在循环中调用多次

### 2.3 与现有概念的映射

| 新抽象 | 现有概念 | 说明 |
|--------|---------|------|
| **VecReg** | `REPEAT_BYTE = 256` | 单次repeat处理的数据量 |
| **Block** | `BLOCK_BYTE_SIZE = 32` | 硬件stride的基本单位 |
| **Lane** | 元素 | Block内的单个数据元素 |
| **LanesPerBlock** | `32 / sizeof(T)` | FP32: 8, FP16: 16 |
| **LanesPerReg** | `VL = 256 / sizeof(T)` | FP32: 64, FP16: 128 |
| **VecReg Op** | 单次vtile操作 | 虚拟指令 vs 可能包含循环的操作 |

---

## 3. 核心数据结构

### 3.1 基础常量

```cpp
namespace pto::npu::a2a3::vreg {

// 硬件常量
constexpr uint32_t VEC_REG_BYTES = 256;      // VecReg大小
constexpr uint32_t BLOCK_BYTES = 32;         // Block大小
constexpr uint32_t BLOCKS_PER_REG = 8;       // 每个VecReg有8个Block

// 类型相关常量
template <typename T>
constexpr uint32_t LanesPerBlock = BLOCK_BYTES / sizeof(T);

template <typename T>
constexpr uint32_t LanesPerReg = VEC_REG_BYTES / sizeof(T);

// 示例：
// FP32: LanesPerBlock = 8, LanesPerReg = 64
// FP16: LanesPerBlock = 16, LanesPerReg = 128

}  // namespace vreg
```

### 3.2 VecReg：向量寄存器（支持硬件repeat）

```cpp
template <typename T>
struct VecReg {
    __ubuf__ T *ptr;                // 数据指针
    uint16_t num_regs;              // 寄存器个数（对应硬件repeat次数）
    uint16_t active_lanes;          // 激活的lane数（每个寄存器）
    uint16_t reg_stride_blocks;     // 寄存器间步长（以Block为单位）
    uint16_t blk_stride = 1;        // 寄存器内Block间步长（默认1，表示连续dense布局）

    // 辅助方法
    uint32_t TotalLanes() const {
        return static_cast<uint32_t>(num_regs) * LanesPerReg<T>;
    }

    uint32_t RegStrideBytes() const {
        return static_cast<uint32_t>(reg_stride_blocks) * BLOCK_BYTES;
    }

    uint32_t RegStrideElements() const {
        return static_cast<uint32_t>(reg_stride_blocks) * LanesPerBlock<T>;
    }

    uint32_t BlockStrideBytes() const {
        return static_cast<uint32_t>(blk_stride) * BLOCK_BYTES;
    }
};
```

**语义说明：**
- `ptr`：第一个VecReg的数据指针
- `num_regs`：寄存器个数，**直接映射硬件repeat次数**（1-255）
- `active_lanes`：每个寄存器中激活的lane数（≤ LanesPerReg）
- `reg_stride_blocks`：相邻寄存器之间的步长，以Block（32B）为单位
- `blk_stride`：单个寄存器内相邻Block之间的步长，以Block（32B）为单位

**设计说明：**
- `reg_stride_blocks` 描述 **repeat维度** 的步长
- `blk_stride` 描述 **单次repeat内部** 的Block访问模式
- `blk_stride = 1` 是默认优化路径，覆盖当前dense/contiguous主场景
- 但 `blk_stride = 1` **不是唯一语义**；当底层指令支持时，可以显式表达 `blk_stride != 1`

**重要**：VecReg可以描述**多个寄存器**，充分利用硬件repeat能力！

**示例：**
```cpp
// 单个VecReg（1次repeat）
VecReg<float> reg{ptr, 1, 64, 8};
// 执行：vadd(..., repeat=1, ...)

// 240个VecReg（240次repeat）
VecReg<float> regs{ptr, 240, 64, 8};
// 执行：vadd(..., repeat=240, ...)
// 硬件自动处理240次repeat，无需软件循环！

// 255个VecReg（硬件上限）
VecReg<float> regs{ptr, 255, 64, 8};
// 执行：vadd(..., repeat=255, ...)

// 非连续Block布局（按需显式指定blk_stride）
VecReg<float> strided{ptr, 32, 64, 8, 2};
// 单个寄存器内每个Block间隔2个Block访问
```

**约束（VecReg不是任意memory view）：**
- `VecReg` 只描述 **CCE可编码** 的向量访问模式，不是通用tensor slice/view
- `num_regs` 直接映射硬件 `repeat`，取值范围必须是 `[1, 255]`
- `active_lanes` 必须满足 `1 <= active_lanes <= LanesPerReg<T>`
- `reg_stride_blocks` / `blk_stride` 的单位都是 `Block(32B)`，不是元素；这里使用 `uint16_t` 是为了容纳不同 CCE 向量指令族的 stride 编码上限
- 不同指令族的 stride 能力并不完全相同：例如经典三目向量指令 `vadd/vsub/vmul/vdiv/vmax/vmin/...` 的显式stride形态通常收敛在 `<=255`，而 `vadds/vmuls/vector_dup/vconv*` 等指令族可出现更宽的16-bit stride 参数；具体合法范围由对应底层intrinsic校验
- `reg_stride_blocks = 0` 或 `blk_stride = 0` 只在底层指令明确支持广播/复用语义时才合法，不能把 `0` 当成任意stride；例如 `src1.blk_stride = 0` 表示单个Block在一个VecReg内复用，`src1.reg_stride_blocks = 0` 表示同一行在repeat维度复用
- 默认dense主路径下，`dst/src0` 应保持 `reg_stride_blocks > 0` 且 `blk_stride = 1`
- `VecReg` 主要服务于**按BlockStride + RepeatStride编码**的向量算术/单目/部分转换指令族；`vcopy/vgather/vscatter/vtranspose/...` 这类拥有独立寻址模型的指令，应单独设计对应operand，而不是强行塞进同一个 `VecReg` 语义里
- `ptr` 不是任意地址：除满足 `T` 自身对齐外，按Block寻址的首版路径要求首Block满足 `32B` 对齐；若具体CCE指令有更严格要求，以该指令为准
- `LaneMask` 只是进一步收缩参与计算的lane范围，必须满足 `mask.active_lanes <= active_lanes`

### 3.3 BroadcastReg：广播寄存器（支持多个标量）

```cpp
template <typename T>
struct BroadcastReg {
    __ubuf__ T *scalars;       // 标量数组指针
    uint16_t num_scalars;      // 标量个数
    uint16_t scalar_stride;    // 标量间步长（以元素为单位）
};
```

**语义说明：**
- `scalars`：标量数组的起始指针
- `num_scalars`：标量个数，**对应需要广播的次数**
- `scalar_stride`：相邻标量之间的步长（通常为1）

**重要**：BroadcastReg可以描述**多个标量**，配合VecReg的num_regs使用！

**示例：**
```cpp
// 单个标量广播
BroadcastReg<float> scalar{ptr, 1, 1};
// 配合 VecReg{..., 1, ...} 使用

// 240个标量广播（行向广播）
BroadcastReg<float> scalars{ptr, 240, 1};
// 配合 VecReg{..., 240, ...} 使用
// 每个标量广播到对应的VecReg

// 非连续标量
BroadcastReg<float> scalars{ptr, 100, 2};
// 标量间隔为2（ptr[0], ptr[2], ptr[4], ...）
```

### 3.4 ScalarPack：标量包（Reduce输出）

```cpp
template <typename T>
struct ScalarPack {
    __ubuf__ T *ptr;         // 输出标量起始地址
    uint16_t count;          // 标量个数
    uint16_t stride;         // 标量间步长（以元素为单位）
};
```

**用途：**
- 表示 `num_regs` 个VecReg在规约后的标量结果
- 典型场景：`[num_regs, active_lanes] -> [num_regs]`
- 用于表达行内规约（row reduce）这类shape-changing操作的输出

**约束（ScalarPack不是任意scalar slice）：**
- `ScalarPack` 描述的是**逻辑上的标量结果布局**，不等价于“任意一条CCE指令都能直接把一个 `VecReg` 变成一个 `ScalarPack`”
- `count` 必须和产生该结果的语义一一对应；对当前 `VReduceRow*`，要求 `count == src.num_regs`，因此首版有效范围是 `[1, 255]`
- `stride` 的单位是**元素**，不是Block；`stride = 1` 表示标量连续写出
- `stride` 必须满足对应底层指令的目标scalar stride编码限制；当前A2/A3 reduce主路径按 `vcadd/vcmax/vcmin(mode=false)` 的目标stride约束处理，不把 `ScalarPack` 当成任意跨距view
- 首版约定 `stride >= 1`，不把 `stride = 0` 作为通用语义
- `ptr` 需要满足 `T` 的对齐要求；如果后续lowering经过临时block/vector buffer，再写回 `ScalarPack`，该tmp buffer的约束由具体lowering另行满足
- `vcgadd/vcgmax/vcgmin` 的直接输出是“每次迭代8个block得到8个block结果”，本质上仍是向量/block结果，不应直接等同于 `ScalarPack`

### 3.5 LaneMask：Lane级别的掩码

```cpp
template <typename T>
struct LaneMask {
    uint32_t active_lanes;  // 激活的lane数

    static LaneMask<T> Full() {
        return {LanesPerReg<T>};
    }

    static LaneMask<T> FromCount(uint32_t count) {
        return {count};
    }
};
```

**用途：**
- 处理非对齐的数据（如31列而不是64列）
- 对应硬件的mask机制

---

## 4. API设计（虚拟指令集）

### 4.1 设计原则 ⭐

**核心原则：每个VecReg操作 = 一次硬件指令调用（充分利用硬件repeat能力）**

```cpp
// VecReg操作直接映射硬件repeat（1-255）
VAdd(VecReg{ptr, 240, 64, 8}, ...);  // 1次vadd调用，硬件repeat=240

// 当需要 > 255次时，Scheduler负责拆分
for (int i = 0; i < 1000; i += 255) {
    int batch = min(255, 1000 - i);
    VAdd(VecReg{ptr + i*stride, batch, 64, 8}, ...);
}
```

**关键区别：**
- ❌ **不是**：VecReg操作内部做软件循环
- ✅ **而是**：VecReg操作利用硬件repeat能力
- ✅ **当超过255时**：Scheduler负责拆分

**补充说明：**
- 对于 `Bin/Unary/Expand` 这类逐元素操作，优先保持 **一条VecReg操作 = 一次硬件指令调用**
- 对于 `Reduce` 这类shape-changing操作，允许用 **固定的小序列** 映射硬件能力
- 即便是Reduce，VecReg层也 **不负责** `>255` 拆分、tile级循环或跨tile调度

#### 4.1.1 命名原则与向量指令覆盖

底层API命名应尽量贴近 CCE intrinsic 本名/本族，而不是为某个stride组合单独发明语义名：
- `VColExpandAdd` 不应作为底层API存在；它只是 `vadd` 在 `src1.reg_stride_blocks = 0` 时的一种用法
- `RowExpand*` 也不应作为底层API存在；它只是 `VBrcb + VAdd(..., src1.blk_stride = 0)` 的固定组合
- 如果未来要直接暴露reduce底层指令，更推荐 `VCAdd/VCMax/VCMin/VCGAdd/...` 这类和CCE一致的命名；`VReduceRow*` 适合作为上层组合语义接口

结合官方《向量算术指令》与本地 CANN `stub_fun.h` 可见，底层抽象应优先覆盖下列向量指令族：

| 指令族 | 代表CCE intrinsic | 建议底层API形态 | 说明 |
| --- | --- | --- | --- |
| 向量-向量二元 | `vadd/vsub/vmul/vdiv/vmax/vmin/vand/vor` | `Op(VecReg dst, VecReg src0, VecReg src1, LaneMask mask)` | 共享同一组 repeat/block stride 语义 |
| 向量-标量二元 | `vadds/vmuls/vmins/vmaxs` | `Op(VecReg dst, VecReg src, T scalar, LaneMask mask)` | 共享 `VecReg + scalar` 形态 |
| 向量单目数学 | `vabs/vexp/vln/vrec/vsqrt/vrsqrt/vrelu/vlrelu/vnot` | `Op(VecReg dst, VecReg src, LaneMask mask)` | 共享单src的 `VecReg` 形态 |
| 融合算术 | `vmadd/vmla/vaddrelu/vsubrelu/vmaddrelu` | 以具体intrinsic同名封装 | 不再额外发明语义名 |
| 广播/初始化 | `vbrcb/vector_dup` | `VBrcb(BroadcastReg)` / `VectorDup(VecReg, scalar)` | 行广播和整块填充分别对应不同CCE原语 |
| 规约 | `vcadd/vcmax/vcmin` | 专用reduce wrapper | 输出是标量流，不宜伪装成普通二元算子 |
| 分组规约 | `vcgadd/vcgmax/vcgmin` | faithful low-level wrapper | 输出仍是block/vector结果，不等价于 `ScalarPack` |
| 比较/选择 | `vcmp*/vsel` | 需要单独的谓词/比较结果抽象 | 不能只靠 `LaneMask` 覆盖 |
| 类型转换 | `vconv_*` | `Op(VecReg<Tdst> dst, VecReg<Tsrc> src, LaneMask mask)` | 与普通 `VecReg` 兼容，但需允许源/目的dtype不同 |
| 数据搬移/重排 | `vcopy/vgather/vscatter/vextract/vconcat/vtranspose/vpadding` | 单独设计专用operand | 这类指令的寻址模型不等于 block/repeat stride |

**和当前代码的对应关系：**
- 当前 `vtile::VAdd` 可直接对应未来 `vreg::VAdd/VSub/VMul/...` 这类二元向量指令族
- 当前 `vtile::VColExpandAdd` 应下沉为一个上层helper；底层只保留 `VAdd(..., src1.reg_stride_blocks = 0)`
- 当前 `TRowExpand*` 内部会循环调用 `vbrcb`；提案里不再把这种“带批处理的row expand”作为底层API，而是显式写成 `VBrcb + VAdd`
- 当前 `TRowSum/TRowMax/TRowMin` 属于上层组合算子；如果后续要补齐底层覆盖，应优先按 `VCAdd/VCMax/VCMin/VCG*` 的CCE命名补齐

### 4.2 向量-向量二元指令族（以 VAdd 为例）

```cpp
// 向量加法（支持硬件repeat）
template <typename T>
void VAdd(
    VecReg<T> dst,
    VecReg<T> src0,
    VecReg<T> src1,
    LaneMask<T> mask = LaneMask<T>::Full()
);

// 语义：dst[reg_i][lane_j] = src0[reg_i][lane_j] + src1[reg_i][lane_j]
// 处理：num_regs个VecReg，执行1次vadd（硬件repeat=num_regs）
```

**同族扩展：**
- 同一操作数模型可直接覆盖 `VSub/VMul/VDiv/VMax/VMin/VAnd/VOr` 等大部分向量-向量二元指令族
- 对这组经典三目向量指令，底层wrapper应按对应CCE intrinsic的限制校验 `blk_stride/reg_stride_blocks <= 255`

**使用示例：**
```cpp
// 示例1：240个VecReg（充分利用硬件repeat）
VecReg<float> dst{dstPtr, 240, 64, 8};
VecReg<float> src0{src0Ptr, 240, 64, 8};
VecReg<float> src1{src1Ptr, 240, 64, 8};

VAdd(dst, src0, src1);
// 硬件执行：1次vadd调用，硬件repeat=240
// 无软件循环！性能最优！

// 示例2：单个VecReg
VAdd(VecReg<float>{dstPtr, 1, 64, 8}, ...);
// 硬件执行：1次vadd调用，硬件repeat=1

// 示例3：1000个VecReg（Scheduler负责拆分）
for (int i = 0; i < 1000; i += 255) {
    int batch = std::min(255, 1000 - i);
    uint32_t offset = i * 8 * LanesPerBlock<float>;
    VAdd(
        VecReg<float>{dstPtr + offset, batch, 64, 8},
        VecReg<float>{src0Ptr + offset, batch, 64, 8},
        VecReg<float>{src1Ptr + offset, batch, 64, 8}
    );
}
// 循环4次：255 + 255 + 255 + 235
```

**非对齐示例：**
```cpp
// 16个VecReg，每个31 lanes（非64对齐）
VecReg<float> dst{dstPtr, 16, 31, 8};
VecReg<float> src0{src0Ptr, 16, 31, 8};
VecReg<float> src1{src1Ptr, 16, 31, 8};

LaneMask<float> mask = LaneMask<float>::FromCount(31);
VAdd(dst, src0, src1, mask);
// 硬件执行：1次vadd调用，硬件repeat=16，mask处理尾部
```

**补充说明：**
- `blk_stride = 1` 是默认快路径，可覆盖当前dense主场景
- `src.reg_stride_blocks = 0` 表示同一个VecReg在repeat维度复用；列向广播/共享一行权重不需要单独API
- `src.blk_stride = 0` 表示同一个Block在单次repeat内复用；这正是行向广播在 `vadd` 里的原生表达
- `src.reg_stride_blocks = 0 && src.blk_stride = 0` 表示同一个Block在整个repeat包内都被复用
- 首版普通dense `VAdd` 主路径仍保持 `blk_stride = 1`，扩展布局按底层intrinsic能力校验

### 4.3 块广播与行向广播（VBrcb + VAdd）

```cpp
// 标量 -> Block 的底层广播原语（1层CCE抽象）
template <typename T>
void VBrcb(
    __ubuf__ T *dst_blocks,
    BroadcastReg<T> src1
);

// VBrcb语义：src1.scalars[i] -> dst_blocks中第i个32B Block
//            每个Block内部的LanesPerBlock<T>个元素都等于该标量
// 处理：1次vbrcb（硬件repeat = CeilDivision(num_scalars, 8)）

// 行向广播不需要单独API：
// dst[reg_i][lane_j] = src0[reg_i][lane_j] + src1_blocks对应第reg_i个Block的广播值
// 可直接表达为：
// VAdd(dst, src0, VecReg<T>{src1_blocks, dst.num_regs, dst.active_lanes, 1, 0})
// 其中 src1 的 blk_stride = 0，表示同一个Block在单次repeat内广播到整个VecReg
```

**使用示例：**
```cpp
// 示例1：240行的行向广播（2条底层原语，各自映射1条硬件指令）
VecReg<float> dst{dstPtr, 240, 64, 8};
VecReg<float> src0{src0Ptr, 240, 64, 8};
BroadcastReg<float> src1{src1Ptr, 240, 1};
__ubuf__ float *src1_blocks = tmpPtr;  // 至少容纳240个Block = 240 * 8个float

VBrcb(src1_blocks, src1);  // 1次vbrcb
VAdd(dst, src0, VecReg<float>{src1_blocks, 240, 64, 1, 0});  // 1次vadd

// 示例2：单行的行向广播
VBrcb(tmpPtr, BroadcastReg<float>{src1Ptr, 1, 1});
VAdd(VecReg<float>{dstPtr, 1, 64, 8},
     VecReg<float>{src0Ptr, 1, 64, 8},
     VecReg<float>{tmpPtr, 1, 64, 1, 0});

// 示例3：1000行的行向广播（Scheduler负责拆分）
for (int i = 0; i < 1000; i += 255) {
    int batch = std::min(255, 1000 - i);
    uint32_t offset = i * 64;
    VBrcb(tmpPtr, BroadcastReg<float>{src1Ptr + i, batch, 1});
    VAdd(VecReg<float>{dstPtr + offset, batch, 64, 8},
         VecReg<float>{src0Ptr + offset, batch, 64, 8},
         VecReg<float>{tmpPtr, batch, 64, 1, 0});
}
```

### 4.4 重复维共享（由 VAdd 直接表达）

```cpp
// 列向广播 / repeat维共享不需要单独API：
// src1.reg_stride_blocks = 0 表示同一个VecReg在所有repeat之间复用
// 因此：dst[reg_i][lane_j] = src0[reg_i][lane_j] + src1_shared[lane_j]
```

**使用示例：**
```cpp
// 240行共享同一行标量（1条底层vadd，不再引入 VColExpandAdd 专名）
VecReg<float> dst{dstPtr, 240, 64, 8};
VecReg<float> src0{src0Ptr, 240, 64, 8};
VecReg<float> src1_shared{src1Ptr, 240, 64, 0, 1};
// 注意：num_regs = 240 表示逻辑上覆盖240个repeat；由于 reg_stride_blocks = 0，
//       实际物理上只复用同一行VecReg

VAdd(dst, src0, src1_shared);

// 单个标量扩到整块后，在repeat维和寄存器内同时复用
VBrcb(tmpPtr, BroadcastReg<float>{scalarPtr, 1, 1});
VAdd(dst, src0, VecReg<float>{tmpPtr, 240, 64, 0, 0});
```

**结论：**
- `VColExpandAdd` 更适合作为上层helper名字，而不是底层API名字
- 对底层API而言，列向广播只是 `vadd` 的一个合法stride配置，不应再额外造型

### 4.5 其他向量指令族的接口覆盖方向

在官方《向量算术指令》与本地 CANN 头文件中，除 `vadd` 类外，还能看到大量共享相似operand形态的向量intrinsic。底层API建议按“指令族”逐步补齐，而不是继续引入 `VColExpandAdd` 这类语义专名：
- **向量-标量二元**：`VAdds/VMuls/VMaxs/VMins`，统一采用 `VecReg + scalar + LaneMask`
- **向量单目数学**：`VAbs/VExp/VLn/VRec/VSqrt/VRsqrt/VRelu/VLRelu/VNot`，统一采用 `dst VecReg + src VecReg + LaneMask`
- **融合算术**：`VMadd/VMla/VAddRelu/VSubRelu/VMaddRelu`，直接沿用CCE名字建模
- **比较/选择**：`VCmp* / VSel`，需要引入额外的谓词结果抽象，不能只依赖 `LaneMask`
- **类型转换**：`VConv*`，需要允许 `dst/src` 的dtype不同，但 `VecReg` 的repeat/block布局抽象依然可复用
- **数据搬移/重排**：`VCopy/VGather/VScatter/VExtract/VConcat/VTranspose/VPadding`，需要单独的寻址/布局operand，不强行揉进 `VecReg` 同一套语义

### 4.6 行内规约（Row Reduce）

```cpp
template <typename T>
void VReduceRowSum(
    ScalarPack<T> dst,
    VecReg<T> src,
    LaneMask<T> mask = LaneMask<T>::Full()
);

template <typename T>
void VReduceRowMax(
    ScalarPack<T> dst,
    VecReg<T> src,
    LaneMask<T> mask = LaneMask<T>::Full()
);

template <typename T>
void VReduceRowMin(
    ScalarPack<T> dst,
    VecReg<T> src,
    LaneMask<T> mask = LaneMask<T>::Full()
);

// 语义：dst[reg_i] = Reduce_j(src[reg_i][lane_j])
//       即每个VecReg规约成一个标量
// 约束：dst.count == src.num_regs
```

**使用示例：**
```cpp
// 240行，每行64个float，规约为240个标量
VecReg<float> src{srcPtr, 240, 64, 8};
ScalarPack<float> dst{dstPtr, 240, 1};

VReduceRowSum(dst, src);

// 非对齐lane的规约
VecReg<float> tailSrc{srcPtr, 16, 31, 8};
VReduceRowMax(ScalarPack<float>{dstPtr, 16, 1}, tailSrc, LaneMask<float>::FromCount(31));
```

**设计定位：**
- `VReduceRow*` 是 **VecReg层的组合原语**，允许由固定的小序列实现
- 它表达的是“单个寄存器包内部的规约语义”，而不是完整的tile级reduce算法
- 在当前 `VecReg` 契约下（`active_lanes <= LanesPerReg<T>`），`VReduceRow*` 的首选lowering是
  `vcadd/vcmax/vcmin(mode=false)`：每个repeat对应1个VecReg输入，输出1个标量到 `ScalarPack`
- `vcgadd/vcgmax/vcgmin` 更适合作为上层宽行reduce的中间压缩原语：其语义是
  “8个Block输入 → 8个block级规约结果输出”，不作为首版 `VReduceRow*` 的必需实现路径
- 更复杂的shape特化、tmp管理和大规模调度，仍由上层 `TRowReduce*` 负责

### 4.7 列规约（Col Reduce）

列规约本质上是 **跨多个repeat/行的累加或比较**，更接近调度/算法层操作，而不是基础VecReg primitive。

因此建议：
- **行内规约** 进入VecReg层：`VReduceRowSum/Max/Min`
- **列规约** 保持在组合算子层：`TColSum/TColMax/TColMin/TColProd`
- VecReg层只提供实现这些组合算子所需的底层Bin/Expand/Mask能力

---

## 5. 实现细节

### 5.1 VAdd的内部实现（利用硬件repeat）

```cpp
template <typename T>
void VAdd(
    VecReg<T> dst,
    VecReg<T> src0,
    VecReg<T> src1,
    LaneMask<T> mask
) {
    // 前置检查
    PTO_ASSERT(dst.num_regs == src0.num_regs);
    PTO_ASSERT(dst.num_regs == src1.num_regs);
    PTO_ASSERT(dst.num_regs <= 255);  // 硬件repeat上限
    PTO_ASSERT(dst.active_lanes == src0.active_lanes);
    PTO_ASSERT(dst.active_lanes == src1.active_lanes);
    PTO_ASSERT(dst.active_lanes <= LanesPerReg<T>);

    // 设置mask（如果需要）
    if (mask.active_lanes < LanesPerReg<T>) {
        SetContMaskByDType<T>(mask.active_lanes);
    }

    // 执行vadd（1次硬件指令调用，硬件repeat=num_regs）
    vadd(dst.ptr, src0.ptr, src1.ptr,
         dst.num_regs,               // repeat = num_regs（1-255）
         dst.blk_stride,
         src0.blk_stride,
         src1.blk_stride,
         dst.reg_stride_blocks,      // dst stride
         src0.reg_stride_blocks,     // src0 stride
         src1.reg_stride_blocks);    // src1 stride

    // 恢复mask
    if (mask.active_lanes < LanesPerReg<T>) {
        SetFullVecMaskByDType<T>();
    }
}
```

**关键点：**
1. **repeat = num_regs**：直接使用硬件repeat能力（1-255）
2. **无软件循环**：不包含任何for循环
3. **1次硬件指令调用**：只调用1次vadd intrinsic，但硬件内部repeat num_regs次
4. **性能最优**：充分利用硬件并行能力

### 5.2 VBrcb 与 `VAdd(..., blk_stride=0)` 的底层表达（1层CCE抽象）

```cpp
template <typename T>
void VBrcb(
    __ubuf__ T *dst_blocks,
    BroadcastReg<T> src1
) {
    uint32_t repeats = CeilDivision(src1.num_scalars, 8u);
    PTO_ASSERT(repeats <= 255);
    PTO_ASSERT(src1.scalar_stride == 1);

    vbrcb(dst_blocks,
          src1.scalars,
          1,              // blk_stride
          BLOCKS_PER_REG, // rep_stride = 8
          repeats);       // repeat
}
```

**关键点：**
1. **`VBrcb` 与 `VAdd` 各自都只映射1条硬件指令**
2. **底层VecReg API不做软件循环/tiling**：不在底层API中分批vbrcb
3. **行向广播直接由 `VAdd + src1.blk_stride=0` 表达**
4. **批量拆分、临时buffer规划**：由上层Scheduler/组合算子负责

### 5.3 vbrcb的作用详解

**官方定义：**
> 获取src中8个元素，将每个元素单独广播成一个32B的block，然后将8个block连续写入dst。

**单个标量的转换（FP32）：**
```text
输入：[a0]  (1个标量)

vbrcb处理（repeat=1）：
  a0 → [a0, a0, a0, a0, a0, a0, a0, a0]  (Block 0, 32B)

输出：8个元素（32B），都是a0
```

**为什么需要vbrcb？**
- vadd的stride单位是Block（32B），最小步进是8个FP32元素
- 如果直接使用标量，vadd无法实现"广播到整个VecReg"
- vbrcb将标量扩展为一个Block，vadd的stride=0可以将这个Block重复8次
- 完美实现"单个标量广播到64 lanes"的语义

---

## 6. Scheduler如何使用VecReg ⭐

### 6.1 职责分离

| 层次 | 职责 | 不负责 |
|-----|------|--------|
| **Scheduler** | 大规模循环（>255）、tiling策略、数据分块 | 硬件指令映射、mask管理、vbrcb处理 |
| **VecReg Op** | 硬件指令映射（repeat ≤ 255）、mask管理 | 大规模循环、tiling |

### 6.2 示例：240×64矩阵加法（无需Scheduler循环）

```cpp
// Scheduler层代码
void MatrixAdd_Scheduler(
    __ubuf__ float *dst,
    __ubuf__ float *src0,
    __ubuf__ float *src1,
    uint32_t rows,  // 240
    uint32_t cols   // 64
) {
    // 240 <= 255，无需Scheduler循环！
    // 直接调用VecReg，利用硬件repeat
    VAdd(
        VecReg<float>{dst, rows, cols, 8},
        VecReg<float>{src0, rows, cols, 8},
        VecReg<float>{src1, rows, cols, 8}
    );
    // 1次VAdd调用，硬件repeat=240
    // 性能最优！
}
```

### 6.3 示例：1000×64矩阵加法（Scheduler负责拆分）

```cpp
// Scheduler层代码
void LargeMatrixAdd_Scheduler(
    __ubuf__ float *dst,
    __ubuf__ float *src0,
    __ubuf__ float *src1,
    uint32_t rows,  // 1000
    uint32_t cols   // 64
) {
    // 1000 > 255，Scheduler负责拆分
    for (uint32_t i = 0; i < rows; i += 255) {
        uint32_t batch = std::min(255u, rows - i);
        uint32_t offset = i * cols;

        VAdd(
            VecReg<float>{dst + offset, batch, cols, 8},
            VecReg<float>{src0 + offset, batch, cols, 8},
            VecReg<float>{src1 + offset, batch, cols, 8}
        );
    }
    // 循环4次：255 + 255 + 255 + 235
    // 每次VAdd调用利用硬件repeat
}
```

### 6.4 示例：行向广播（无需Scheduler循环）

```cpp
// Scheduler层代码
void RowBroadcastAdd_Scheduler(
    __ubuf__ float *dst,
    __ubuf__ float *src0,
    __ubuf__ float *src1_scalars,
    uint32_t rows,  // 240
    uint32_t cols   // 64
) {
    // 240 <= 255，无需Scheduler循环！
    __ubuf__ float *src1_blocks = tmpPtr;

    VBrcb(src1_blocks, BroadcastReg<float>{src1_scalars, rows, 1});
    VAdd(VecReg<float>{dst, rows, cols, 8},
         VecReg<float>{src0, rows, cols, 8},
         VecReg<float>{src1_blocks, rows, cols, 1, 0});
    // 1次VBrcb + 1次VAdd
}
```

### 6.5 示例：大规模行向广播（Scheduler负责拆分）

```cpp
// Scheduler层代码
void LargeRowBroadcastAdd_Scheduler(
    __ubuf__ float *dst,
    __ubuf__ float *src0,
    __ubuf__ float *src1_scalars,
    uint32_t rows,  // 1000
    uint32_t cols   // 64
) {
    // 1000 > 255，Scheduler负责拆分
    for (uint32_t i = 0; i < rows; i += 255) {
        uint32_t batch = std::min(255u, rows - i);
        uint32_t offset = i * cols;

        VBrcb(tmpPtr, BroadcastReg<float>{src1_scalars + i, batch, 1});

        VAdd(VecReg<float>{dst + offset, batch, cols, 8},
             VecReg<float>{src0 + offset, batch, cols, 8},
             VecReg<float>{tmpPtr, batch, cols, 1, 0});
    }
    // 循环4次，每次：1次VBrcb + 1次VAdd
}
```

### 6.6 优势总结

**充分利用硬件能力：**
- VecReg操作直接映射硬件repeat（1-255）
- 240行矩阵加法：1次VAdd调用（而不是240次）
- 性能提升显著！

**清晰的职责分离：**
- Scheduler：只在需要时（>255）才循环
- VecReg：充分利用硬件repeat能力
- 大多数情况下（≤255），Scheduler无需循环

**易于优化：**
- Scheduler可以根据数据规模选择策略
- 小规模（≤255）：直接调用VecReg
- 大规模（>255）：拆分为多个batch

**性能对比：**
```cpp
// 旧设计（错误）：Scheduler循环240次
for (int i = 0; i < 240; i++) {
    VAdd(VecReg{..., 1, ...}, ...);  // 240次VAdd调用
}
// 性能：240次函数调用 + 240次vadd

// 新设计（正确）：利用硬件repeat
VAdd(VecReg{..., 240, ...}, ...);  // 1次VAdd调用
// 性能：1次函数调用 + 1次vadd（硬件repeat=240）
// 性能提升：~240倍！
```

### 6.7 直观的硬件映射 ✅

```cpp
// 用户思维模型：
VecReg = 256B寄存器 = 8个Block = 64个Lane (FP32)

// 硬件执行：
repeat = 处理一个VecReg
block = 32B的基本单位
lane = 单个数据元素

// 完美对应！
```

### 6.8 清晰的广播语义 ✅

```cpp
// 旧设计：语义混乱
VecTile<float> src1{ptr, 240, 8, 8};
// 这是什么？240×8矩阵？为什么是8？

// 新设计：语义清晰
BroadcastReg<float> src1{ptr, 240, 1};
// 240个标量，行向广播，一目了然！
```

### 6.9 封装硬件细节 ✅

```cpp
// 旧设计：高层直接操作CCE细节且容易混入tiling逻辑
vbrcb(tmpPtr, src1Ptr, 1, 8, 30);
pipe_barrier(PIPE_V);
vadd(dstPtr, src0Ptr, tmpPtr, ...);

// 新设计：底层vreg显式暴露 `VBrcb`；广播通过 `VAdd` 的stride语义表达
VBrcb(tmpPtr, BroadcastReg{src1Ptr, 240, 1});
VAdd(dst, src0, VecReg{tmpPtr, 240, 64, 1, 0});
// 如需进一步隐藏这两步，可在更高层组合算子中封装为组合接口
```

### 6.10 类型安全 ✅

```cpp
// 编译期区分底层原语 / operand 形态
void VAdd(VecReg, VecReg, VecReg);                  // 向量-向量二元
void VAdds(VecReg, VecReg, scalar);                 // 向量-标量二元
void VBrcb(__ubuf__ T*, BroadcastReg<T>);           // 标量 -> Block

// 不会混淆！编译器会检查类型
VAdd(dst, src0, BroadcastReg{...});  // 编译错误！
VAdds(dst, src0, src1_vec);          // 编译错误！
// 列向广播通过 VAdd + VecReg{..., reg_stride_blocks=0} 显式表达，不再需要专用API
// 行向广播通过 VBrcb + VAdd + VecReg{..., blk_stride=0} 显式表达
```

### 6.11 易于理解和教学 ✅

```cpp
// 新用户可以快速理解：
// 1. VecReg就像CPU的SIMD寄存器（如AVX512）
// 2. Block是硬件处理的基本单位
// 3. Lane是并行执行的数据通道
// 4. BroadcastReg明确表达广播语义

// 类比CPU SIMD：
__m512 reg;  // 512-bit寄存器，16个float
VecReg<float> reg;  // 256B寄存器，64个float
```

---

## 7. 编程约束和限制 ⚠️

### 7.1 硬件约束总览

| 约束项 | 限制值 | 说明 | 违反后果 |
|-------|--------|------|---------|
| **VecReg大小** | 256B | 固定值，不可更改 | N/A |
| **Block大小** | 32B | 硬件基本单位 | N/A |
| **LanesPerReg** | FP32: 64, FP16: 128 | 由数据类型决定 | N/A |
| **stride对齐** | 必须32B对齐 | `reg_stride_blocks * 32` | 硬件错误 |
| **指针对齐** | 建议32B对齐 | UB地址对齐 | 性能下降 |
| **active_lanes** | ≤ LanesPerReg | 不能超过VecReg容量 | 硬件错误 |
| **stride上限** | 随指令族变化；`VAdd`类显式stride常见 ≤ 255 | 不是统一常量 | 硬件错误 |

### 7.2 VecReg操作约束

#### 7.2.1 VAdd约束

```cpp
void VAdd(VecReg<T> dst, VecReg<T> src0, VecReg<T> src1, LaneMask<T> mask);
```

**约束条件：**

1. **Lane数一致**：
   ```cpp
   dst.active_lanes == src0.active_lanes == src1.active_lanes
   // 违反：硬件行为未定义
   ```

2. **Lane数上限**：
   ```cpp
   active_lanes <= LanesPerReg<T>
   // FP32: active_lanes <= 64
   // FP16: active_lanes <= 128
   // 违反：硬件错误
   ```

3. **Stride对齐**：
   ```cpp
   reg_stride_blocks * 32 % 32 == 0  // 自动满足
   // 但reg_stride_blocks必须 <= 255
   ```

4. **Stride上限**（按数据类型）：
   ```cpp
   // FP32 (4B):
   reg_stride_blocks <= 255
   // 对应最大stride: 255 * 32 = 8160B = 2040个float

   // FP16 (2B):
   reg_stride_blocks <= 255
   // 对应最大stride: 255 * 32 = 8160B = 4080个half
   ```

5. **Mask约束**：
   ```cpp
   mask.active_lanes <= active_lanes
   // 违反：部分lane未被mask覆盖
   ```

**示例：合法与非法用法**

```cpp
// ✅ 合法：标准64 lanes
VAdd(VecReg<float>{ptr, 1, 64, 8}, ...);

// ✅ 合法：31 lanes + mask
VAdd(VecReg<float>{ptr, 1, 31, 8}, ..., LaneMask<float>::FromCount(31));

// ❌ 非法：超过64 lanes
VAdd(VecReg<float>{ptr, 1, 128, 8}, ...);  // FP32最多64 lanes

// ❌ 非法：stride超限
VAdd(VecReg<float>{ptr, 1, 64, 256}, ...);  // reg_stride_blocks > 255

// ❌ 非法：lane数不一致
VAdd(VecReg<float>{dstPtr, 1, 64, 8},
     VecReg<float>{src0Ptr, 1, 32, 8},  // 32 != 64
     VecReg<float>{src1Ptr, 1, 64, 8});
```

#### 7.2.2 `VBrcb + VAdd` 表达的行向广播约束

```cpp
void VBrcb(__ubuf__ T *dst_blocks, BroadcastReg<T> src1);
// 行向广播通过下式表达：
VAdd(dst, src0, VecReg<T>{src1_blocks, dst.num_regs, dst.active_lanes, 1, 0});
```

**约束条件：**

1. **`VBrcb` 约束**：
   ```cpp
   CeilDivision(src1.num_scalars, 8) <= 255
   src1.scalar_stride == 1
   // dst_blocks 至少能容纳 src1.num_scalars 个Block
   // 即 src1.num_scalars * LanesPerBlock<T> 个元素
   ```

2. **参与 `VAdd` 的3个VecReg约束**：
   - dst.active_lanes == src0.active_lanes
   - src1_view.active_lanes == dst.active_lanes
   - src1_view.num_regs == dst.num_regs
   - active_lanes <= LanesPerReg<T>
   - reg_stride_blocks <= 255
   - 首版建议 dst/src0 的 blk_stride == 1
   - 行向广播要求 `src1_view.reg_stride_blocks == 1` 且 `src1_view.blk_stride == 0`

3. **`src1_blocks` / `src1_view` 布局约束**：
   ```cpp
   // src1_blocks 中每个repeat对应1个32B Block
   // 相邻repeat的Block连续排列（stride = 1 block）
   // 通常由 VBrcb 预先生成
   ```

4. **标量指针对齐**：
   ```cpp
   // src1.scalar建议4B对齐（FP32）或2B对齐（FP16）
   // 不强制，但对齐性能更好
   ```

**示例：合法与非法用法**

```cpp
// ✅ 合法：标准行向广播
VBrcb(tmpPtr, BroadcastReg<float>{scalarPtr, 1, 1});
VAdd(VecReg<float>{dstPtr, 1, 64, 8},
     VecReg<float>{src0Ptr, 1, 64, 8},
     VecReg<float>{tmpPtr, 1, 64, 1, 0});

// ✅ 合法：非对齐lanes + mask
VBrcb(tmpPtr, BroadcastReg<float>{scalarPtr, 1, 1});
VAdd(VecReg<float>{dstPtr, 1, 31, 8},
     VecReg<float>{src0Ptr, 1, 31, 8},
     VecReg<float>{tmpPtr, 1, 31, 1, 0},
     LaneMask<float>::FromCount(31));

// ❌ 非法：dst和src0 lanes不一致
VBrcb(tmpPtr, BroadcastReg<float>{scalarPtr, 1, 1});
VAdd(VecReg<float>{dstPtr, 1, 64, 8},
     VecReg<float>{src0Ptr, 1, 32, 8},  // 不一致
     VecReg<float>{tmpPtr, 1, 64, 1, 0});
```

#### 7.2.3 repeat维共享（由 `VAdd` 表达）约束

```cpp
// 列向广播 / repeat维共享通过下式表达：
VAdd(dst, src0, VecReg<T>{src1_ptr, dst.num_regs, dst.active_lanes, 0, 1});
```

**约束条件：**

1. **基本约束**（继承自VAdd）:
   - `dst.active_lanes == src0.active_lanes == src1_shared.active_lanes`
   - `active_lanes <= LanesPerReg<T>`
   - 对 `vadd` 这类经典三目向量指令，`reg_stride_blocks/blk_stride` 需满足对应intrinsic的stride上限（首版按 `<=255` 收敛）
   - 首版建议 `dst/src0` 的 `blk_stride == 1`

2. **共享语义约束**：
   ```cpp
   src1_shared.num_regs == dst.num_regs
   src1_shared.reg_stride_blocks == 0
   src1_shared.blk_stride == 1
   // 语义：同一个VecReg在所有repeat之间复用
   ```

3. **物理存储约束**：
   ```cpp
   // 当 reg_stride_blocks == 0 时，src1_shared.ptr 只需要提供“1个VecReg”的物理存储
   // num_regs 表示逻辑上覆盖的repeat个数，而不是必须准备num_regs份物理副本
   ```

4. **对齐约束**：
   ```cpp
   // 建议32B对齐，性能更好
   // 不强制，但非对齐可能导致性能下降或触发更严格的底层约束
   ```

**示例：合法与非法用法**

```cpp
// ✅ 合法：标准列向广播 / repeat维共享
VAdd(
    VecReg<float>{dstPtr, 240, 64, 8},
    VecReg<float>{src0Ptr, 240, 64, 8},
    VecReg<float>{src1Ptr, 240, 64, 0, 1}
);

// ✅ 合法：非对齐lanes
VAdd(
    VecReg<float>{dstPtr, 16, 31, 8},
    VecReg<float>{src0Ptr, 16, 31, 8},
    VecReg<float>{src1Ptr, 16, 31, 0, 1},
    LaneMask<float>::FromCount(31)
);

// ❌ 非法：共享源没有覆盖到和dst一致的逻辑repeat数
VAdd(
    VecReg<float>{dstPtr, 16, 64, 8},
    VecReg<float>{src0Ptr, 16, 64, 8},
    VecReg<float>{src1Ptr, 1, 64, 0, 1}  // src1_shared.num_regs != dst.num_regs
);
```

#### 7.2.4 VReduceRow*约束

```cpp
void VReduceRowSum(ScalarPack<T> dst, VecReg<T> src, LaneMask<T> mask);
void VReduceRowMax(ScalarPack<T> dst, VecReg<T> src, LaneMask<T> mask);
void VReduceRowMin(ScalarPack<T> dst, VecReg<T> src, LaneMask<T> mask);
```

**约束条件：**

1. **输出/输入形状约束**：
   ```cpp
   dst.count == src.num_regs
   // 每个VecReg规约成1个标量
   ```

2. **VecReg基本约束**：
   ```cpp
   src.num_regs <= 255
   src.active_lanes <= LanesPerReg<T>
   src.reg_stride_blocks <= 255
   // 首版建议 src.blk_stride == 1，先收敛dense主路径
   ```

3. **ScalarPack步长语义**：
   ```cpp
   // dst.stride 的单位是“元素”，不是Block
   // FP32: stride=1 表示相邻输出相差1个float
   // FP16: stride=1 表示相邻输出相差1个half
   ```

4. **Mask语义**：
   ```cpp
   mask.active_lanes <= src.active_lanes
   // mask作用在输入lane上，而不是输出标量上
   ```

5. **实现建议（首版）**：
   ```cpp
   // 当前VecReg语义下，优先直接lower到 vcadd/vcmax/vcmin(mode=false)
   // 也就是：每个repeat输入1个VecReg，输出1个标量到dst
   // 首版不依赖 mode=true / ACC_VAL，避免额外的累加寄存器状态管理
   // vcg* 主要用于更宽row-reduce的block级中间压缩，属于上层TRowReduce*优化路径
   // 若后续引入vcg*优化，需要注意：mask作用在输入元素上；若某个Block全被mask掉，
   // 对应的block级输出可能不会写回，因此不宜直接作为首版通用tail处理策略
   ```

**示例：合法与非法用法**

```cpp
// ✅ 合法：240个VecReg，各自规约为1个标量
VReduceRowSum(
    ScalarPack<float>{dstPtr, 240, 1},
    VecReg<float>{srcPtr, 240, 64, 8}
);

// ✅ 合法：尾部31 lane参与规约
VReduceRowMax(
    ScalarPack<float>{dstPtr, 16, 1},
    VecReg<float>{srcPtr, 16, 31, 8},
    LaneMask<float>::FromCount(31)
);

// ❌ 非法：输出标量个数与输入VecReg个数不一致
VReduceRowMin(
    ScalarPack<float>{dstPtr, 15, 1},
    VecReg<float>{srcPtr, 16, 64, 8}
);
```

### 7.3 Scheduler层约束

#### 7.3.1 循环次数限制

**无硬件限制**：Scheduler可以循环任意次数

```cpp
// ✅ 合法：循环240次
for (int i = 0; i < 240; i++) {
    VAdd(...);
}

// ✅ 合法：循环10000次
for (int i = 0; i < 10000; i++) {
    VAdd(...);
}

// ✅ 合法：嵌套循环
for (int i = 0; i < 100; i++) {
    for (int j = 0; j < 100; j++) {
        VAdd(...);
    }
}
```

**注意**：虽然无硬件限制，但需要考虑：
- UB空间大小（通常几百KB）
- 性能（过多循环可能导致性能下降）
- Pipeline效率

#### 7.3.2 指针计算约束

```cpp
// ✅ 合法：按stride移动指针
for (int i = 0; i < rows; i++) {
    uint32_t offset = i * stride_elements;
    VAdd(VecReg<float>{ptr + offset, 1, 64, 8}, ...);
}

// ✅ 合法：不规则访问
for (int i = 0; i < rows; i++) {
    uint32_t offset = index_array[i];  // 不规则
    VAdd(VecReg<float>{ptr + offset, 1, 64, 8}, ...);
}

// ⚠️ 注意：确保不越界
// offset + 64 <= total_elements
```

#### 7.3.3 临时缓冲区约束

**UB空间限制**：
```cpp
// 典型UB大小：256KB - 512KB（芯片相关）
// 需要合理规划UB使用

// ✅ 合法：小临时buffer
__ubuf__ float tmp[64];  // 256B

// ✅ 合法：中等临时buffer
__ubuf__ float tmp[1024];  // 4KB

// ⚠️ 注意：大临时buffer可能导致UB溢出
__ubuf__ float tmp[100000];  // 400KB，可能超限
```

### 7.4 数据类型约束

#### 7.4.1 支持的数据类型

| 类型 | sizeof | LanesPerBlock | LanesPerReg | 最大stride (blocks) | 最大stride (elements) |
|-----|--------|---------------|-------------|-------------------|---------------------|
| `float` (b32) | 4B | 8 | 64 | 255 | 2040 |
| `half` (b16) | 2B | 16 | 128 | 255 | 4080 |
| `int32_t` | 4B | 8 | 64 | 255 | 2040 |
| `int16_t` | 2B | 16 | 128 | 255 | 4080 |

**不支持的类型**：
```cpp
// ❌ 不支持：int8_t, uint8_t
VAdd(VecReg<int8_t>{...}, ...);  // 编译错误或硬件错误

// ❌ 不支持：double (b64)
VAdd(VecReg<double>{...}, ...);  // 不支持
```

#### 7.4.2 类型一致性

```cpp
// ✅ 合法：所有操作数类型一致
VAdd(VecReg<float>{...}, VecReg<float>{...}, VecReg<float>{...});

// ❌ 非法：类型不一致
VAdd(VecReg<float>{...}, VecReg<half>{...}, VecReg<float>{...});
```

### 7.5 性能相关约束

#### 7.5.1 对齐建议

```cpp
// 最佳性能：32B对齐
__ubuf__ float *ptr = AllocAligned(size, 32);
VAdd(VecReg<float>{ptr, 1, 64, 8}, ...);

// 可接受：自然对齐（4B for float）
__ubuf__ float *ptr = malloc(size);
VAdd(VecReg<float>{ptr, 1, 64, 8}, ...);  // 性能略降

// 避免：未对齐
__ubuf__ float *ptr = (float*)((char*)base + 1);
VAdd(VecReg<float>{ptr, 1, 64, 8}, ...);  // 性能显著下降
```

#### 7.5.2 Stride建议

```cpp
// 最佳：stride = LanesPerReg（连续访问）
VAdd(VecReg<float>{ptr, 1, 64, 8}, ...);  // stride = 64 elements

// 可接受：stride > LanesPerReg（有间隔）
VAdd(VecReg<float>{ptr, 1, 64, 16}, ...);  // stride = 128 elements

// 避免：stride过大（缓存不友好）
VAdd(VecReg<float>{ptr, 1, 64, 255}, ...);  // stride = 2040 elements
```

#### 7.5.3 循环展开建议

```cpp
// 好：适度展开（2-4次）
for (int i = 0; i < rows; i += 2) {
    VAdd(VecReg<float>{ptr + i*64, 1, 64, 8}, ...);
    VAdd(VecReg<float>{ptr + (i+1)*64, 1, 64, 8}, ...);
}

// 避免：过度展开（>8次）
// 可能导致寄存器压力过大
```

### 7.6 常见错误和调试

#### 7.6.1 常见错误

```cpp
// 错误1：stride超限
VecReg<float> reg{ptr, 1, 64, 300};  // ❌ 300 > 255
// 解决：拆分为多个小stride的操作

// 错误2：lanes超限
VecReg<float> reg{ptr, 1, 128, 8};  // ❌ FP32最多64 lanes
// 解决：拆分为多次操作

// 错误3：lanes不一致
VAdd(VecReg<float>{dst, 1, 64, 8},
     VecReg<float>{src0, 1, 32, 8},  // ❌ 32 != 64
     VecReg<float>{src1, 1, 64, 8});
// 解决：确保所有操作数lanes一致

// 错误4：忘记mask
VAdd(VecReg<float>{ptr, 1, 31, 8}, ...);  // ⚠️ 缺少mask
// 解决：添加mask
VAdd(VecReg<float>{ptr, 1, 31, 8}, ..., LaneMask<float>::FromCount(31));

// 错误5：指针越界
for (int i = 0; i < rows; i++) {
    VAdd(VecReg<float>{ptr + i*64, 1, 64, 8}, ...);
}
// 如果rows*64 > buffer_size，会越界
// 解决：检查边界条件
```

#### 7.6.2 调试技巧

```cpp
// 技巧1：添加断言
void MyScheduler(...) {
    PTO_ASSERT(rows * 64 <= buffer_size);  // 检查越界
    PTO_ASSERT(reg_stride_blocks <= 255);  // 检查stride

    for (int i = 0; i < rows; i++) {
        VAdd(...);
    }
}

// 技巧2：打印关键参数
void MyScheduler(...) {
    printf("rows=%d, cols=%d, stride=%d\n", rows, cols, stride);
    // 检查参数是否合理
}

// 技巧3：分阶段验证
// 先验证单次操作
VAdd(VecReg<float>{ptr, 1, 64, 8}, ...);  // 单次测试

// 再验证循环
for (int i = 0; i < 2; i++) {  // 先测试2次
    VAdd(...);
}

// 最后验证完整逻辑
for (int i = 0; i < rows; i++) {
    VAdd(...);
}
```

### 7.7 约束总结表

| 约束类型 | 约束值 | 检查时机 | 违反后果 |
|---------|--------|---------|---------|
| VecReg大小 | 256B | 编译期 | N/A（固定） |
| LanesPerReg | FP32: 64, FP16: 128 | 编译期 | N/A（固定） |
| num_regs | `1..255`，直接映射硬件repeat | 运行时 | 硬件错误或需上层拆分 |
| active_lanes | `1..LanesPerReg<T>` | 运行时 | 硬件错误 |
| reg_stride_blocks | 单位为Block；上限依指令族而定，`0` 仅在支持复用语义时合法 | 运行时 | 硬件错误或未定义行为 |
| blk_stride | 单位为Block；经典三目向量指令通常收敛到 `<=255`，部分指令族可更宽，`0` 仅用于广播类语义 | 运行时 | 硬件错误或未定义行为 |
| ScalarPack.count | row-reduce场景要求 `count == src.num_regs`，首版 `1..255` | 运行时 | 语义错误或lowering失败 |
| ScalarPack.stride | 单位为元素；首版 `>= 1` 且需满足目标指令dst stride限制 | 运行时 | 硬件错误或lowering失败 |
| stride对齐 | Block路径按32B对齐建模 | 运行时 | 硬件错误 |
| lanes一致性 | dst == src0 == src1（广播特例除外） | 运行时 | 未定义行为 |
| 指针对齐 | 建议32B；若底层指令更严格，以指令要求为准 | 运行时 | 性能下降或硬件错误 |
| UB空间 | 芯片相关（256KB-512KB） | 运行时 | 内存错误 |
| 循环次数 | 底层API单次调用不超过255；更大规模由Scheduler拆分 | N/A | N/A |

---

## 8. 迁移策略

### 8.1 渐进式迁移

**阶段1：添加新抽象，保持兼容**
```cpp
namespace pto::npu::a2a3 {
    // 保留旧的vtile命名空间
    namespace vtile { /* 现有代码 */ }

    // 添加新的vreg命名空间
    namespace vreg {
        template <typename T> struct VecReg { /*...*/ };
        template <typename T> struct BroadcastReg { /*...*/ };
        // ...
    }
}
```

**阶段2：实现新API**
```cpp
// 新API使用vreg
namespace vreg {
    // 底层原语：名字尽量与CCE intrinsic同名/同族
    void VAdd(VecReg<T>, VecReg<T>, VecReg<T>);
    void VSub(VecReg<T>, VecReg<T>, VecReg<T>);
    void VMul(VecReg<T>, VecReg<T>, VecReg<T>);
    void VDiv(VecReg<T>, VecReg<T>, VecReg<T>);
    void VMax(VecReg<T>, VecReg<T>, VecReg<T>);
    void VMin(VecReg<T>, VecReg<T>, VecReg<T>);

    void VAdds(VecReg<T>, VecReg<T>, T);
    void VMuls(VecReg<T>, VecReg<T>, T);
    void VectorDup(VecReg<T>, T);
    void VBrcb(__ubuf__ T*, BroadcastReg<T>);

    void VAbs(VecReg<T>, VecReg<T>);
    void VExp(VecReg<T>, VecReg<T>);
    void VLn(VecReg<T>, VecReg<T>);
    void VRec(VecReg<T>, VecReg<T>);
    void VSqrt(VecReg<T>, VecReg<T>);
    void VRsqrt(VecReg<T>, VecReg<T>);
    void VRelu(VecReg<T>, VecReg<T>);

    // 组合语义接口：如列广播helper、row-reduce helper，可建立在底层原语之上
    void VReduceRowSum(ScalarPack<T>, VecReg<T>);
    void VReduceRowMax(ScalarPack<T>, VecReg<T>);
    void VReduceRowMin(ScalarPack<T>, VecReg<T>);
}

// 旧API保持不变
namespace vtile {
    void VAdd(VecTile<T>, VecTile<T>, VecTile<T>);
}
```

**阶段3：提供转换工具**
```cpp
// VecTile → VecReg 转换
// 仅适用于：单次VecReg op可直接覆盖的dense row-major视图
// 即：tile.cols <= LanesPerReg<T>，tile.rows <= 255，且ld满足32B对齐
template <typename T>
VecReg<T> ToVecReg(const VecTile<T>& tile) {
    PTO_ASSERT(tile.rows <= 255);
    PTO_ASSERT(tile.cols <= LanesPerReg<T>);
    PTO_ASSERT((tile.ld * sizeof(T)) % BLOCK_BYTES == 0);

    return VecReg<T>{
        tile.ptr,
        tile.rows,
        tile.cols,
        static_cast<uint16_t>((tile.ld * sizeof(T)) / BLOCK_BYTES),
        1
    };
}
```

**阶段4：逐步迁移用户代码**
- 新算子使用vreg API
- 旧算子保持不变
- 提供迁移指南

**阶段5：废弃旧API（可选）**
- 标记vtile为deprecated
- 提供自动迁移工具
- 最终移除vtile

### 8.2 性能验证

在迁移前需要验证：
1. 新API的性能与旧API相同
2. vbrcb封装没有引入额外开销
3. 编译后的汇编代码一致

---

## 9. 示例代码

### 9.1 矩阵加法

```cpp
// 240×64矩阵加法（FP32）
void MatrixAdd(
    __ubuf__ float *dst,
    __ubuf__ float *src0,
    __ubuf__ float *src1,
    uint32_t rows,
    uint32_t cols
) {
    VecReg<float> dst_reg{dst, rows, cols, 8};
    VecReg<float> src0_reg{src0, rows, cols, 8};
    VecReg<float> src1_reg{src1, rows, cols, 8};

    VAdd(dst_reg, src0_reg, src1_reg);
}
```

### 9.2 行向广播

```cpp
// dst[i,j] = src0[i,j] + src1[i]
void RowBroadcastAdd(
    __ubuf__ float *dst,
    __ubuf__ float *src0,
    __ubuf__ float *src1_scalars,
    uint32_t rows,
    uint32_t cols
) {
    VecReg<float> dst_reg{dst, rows, cols, 8};
    VecReg<float> src0_reg{src0, rows, cols, 8};
    __ubuf__ float *src1_blocks = tmpPtr;

    VBrcb(src1_blocks, BroadcastReg<float>{src1_scalars, rows, 1});
    VAdd(dst_reg, src0_reg, VecReg<float>{src1_blocks, rows, cols, 1, 0});
}
```

### 9.3 列向广播

```cpp
// dst[i,j] = src0[i,j] + src1[j]
void ColBroadcastAdd(
    __ubuf__ float *dst,
    __ubuf__ float *src0,
    __ubuf__ float *src1_scalars,
    uint32_t rows,
    uint32_t cols
) {
    VecReg<float> dst_reg{dst, rows, cols, 8};
    VecReg<float> src0_reg{src0, rows, cols, 8};
    VecReg<float> src1_reg{src1_scalars, rows, cols,
                           0,
                           static_cast<uint16_t>(1)};

    VAdd(dst_reg, src0_reg, src1_reg);
}
```

### 9.4 非对齐数据

```cpp
// 16×31矩阵加法（31列，非64对齐）
void NonAlignedAdd(
    __ubuf__ float *dst,
    __ubuf__ float *src0,
    __ubuf__ float *src1
) {
    VecReg<float> dst_reg{dst, 16, 31, 8};
    VecReg<float> src0_reg{src0, 16, 31, 8};
    VecReg<float> src1_reg{src1, 16, 31, 8};

    LaneMask<float> mask = LaneMask<float>::FromCount(31);
    VAdd(dst_reg, src0_reg, src1_reg, mask);
}
```

---

## 10. 文档和教学

### 10.1 用户文档大纲

```markdown
# VecReg 编程指南

## 1. 核心概念
- VecReg：256B向量寄存器
- Block：32B基本单位
- Lane：数据通道

## 2. 基础操作
- 矩阵加法
- 矩阵乘法
- 元素级操作

## 3. 广播操作
- 行向广播
- 列向广播
- 标量广播

## 4. 高级特性
- Mask处理
- 非对齐数据
- 性能优化

## 5. 最佳实践
- 内存对齐
- 批量处理
- 资源管理
```

### 10.2 教学示例

```cpp
// 示例1：理解VecReg结构
// VecReg就像一个256B的寄存器，包含64个float（FP32）
VecReg<float> reg{ptr, 1, 64, 8};
// 1个寄存器，64个lane全激活，步长8个block

// 示例2：理解Block
// 每个Block是32B，包含8个float
// 硬件以Block为单位进行stride

// 示例3：理解广播
// BroadcastReg明确表达"这是标量数组，需要广播"
BroadcastReg<float> scalars{ptr, 240, 1};
// 240个标量，每个广播到一行
```

---

## 11. 总结

### 11.1 设计优势

1. **直观性**：Reg-Block-Lane模型与硬件结构完美对应
2. **清晰性**：BroadcastReg明确表达广播语义
3. **封装性**：vbrcb等硬件细节对用户透明
4. **安全性**：类型系统区分不同操作
5. **可扩展性**：易于添加新的广播模式和操作

### 11.2 实施建议

1. **原型验证**：在新命名空间实现VecReg/BroadcastReg
2. **性能测试**：确保无性能退化
3. **文档先行**：详细解释Reg-Block-Lane模型
4. **渐进迁移**：保持向后兼容，逐步迁移
5. **用户反馈**：收集使用体验，持续改进

### 11.3 预期收益

- **降低学习曲线**：新用户更容易理解
- **减少错误**：类型安全避免误用
- **提升可维护性**：代码更清晰易读
- **便于优化**：硬件细节封装便于后续优化

---

## 附录A：术语对照表

| 新术语 | 旧术语 | 说明 |
|-------|-------|------|
| VecReg | VecTile | 向量寄存器 vs 向量瓦片 |
| num_regs | rows | 寄存器个数 vs 行数 |
| active_lanes | cols | 激活lane数 vs 列数 |
| reg_stride_blocks | rep_stride | 寄存器步长 vs 重复步长 |
| BroadcastReg | VecTile (特殊用法) | 广播寄存器 vs 瓦片 |
| Lane | 元素 | 通道 vs 元素 |
| Block | 32B块 | 块 vs 块 |

## 附录B：参考资料

- `vectile_design.md`：当前VecTile设计文档
- `vbrcb_instruction_guide.md`：vbrcb指令详细说明
- `expand_usage_guide.md`：Expand操作使用指南
