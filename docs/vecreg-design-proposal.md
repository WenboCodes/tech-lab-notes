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

    // 辅助方法
    uint16_t TotalLanes() const {
        return num_regs * LanesPerReg<T>;
    }

    uint16_t RegStrideBytes() const {
        return reg_stride_blocks * BLOCK_BYTES;
    }

    uint16_t RegStrideElements() const {
        return reg_stride_blocks * LanesPerBlock<T>;
    }
};
```

**语义说明：**
- `ptr`：第一个VecReg的数据指针
- `num_regs`：寄存器个数，**直接映射硬件repeat次数**（1-255）
- `active_lanes`：每个寄存器中激活的lane数（≤ LanesPerReg）
- `reg_stride_blocks`：相邻寄存器之间的步长，以Block（32B）为单位

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
```

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

### 3.4 LaneMask：Lane级别的掩码

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

### 4.2 普通二元操作（Bin）

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

### 4.3 行向广播（Row Expand）

```cpp
// 行向广播加法（支持硬件repeat）
template <typename T>
void VRowExpandAdd(
    VecReg<T> dst,
    VecReg<T> src0,
    BroadcastReg<T> src1
);

// 语义：dst[reg_i][lane_j] = src0[reg_i][lane_j] + src1.scalars[reg_i]
//       src1.scalars[reg_i] 被广播到第reg_i个VecReg的所有lanes
// 处理：num_regs个VecReg，执行1次vadd（硬件repeat=num_regs）
// 前提：需要vbrcb预处理标量数组
```

**使用示例：**
```cpp
// 示例1：240行的行向广播（充分利用硬件repeat）
VecReg<float> dst{dstPtr, 240, 64, 8};
VecReg<float> src0{src0Ptr, 240, 64, 8};
BroadcastReg<float> src1{src1Ptr, 240, 1};  // 240个标量

VRowExpandAdd(dst, src0, src1);
// 内部：vbrcb预处理240个标量（分批，每批30个）
// 然后：1次vadd调用，硬件repeat=240
// 性能优于240次单独调用！

// 示例2：单行的行向广播
VRowExpandAdd(
    VecReg<float>{dstPtr, 1, 64, 8},
    VecReg<float>{src0Ptr, 1, 64, 8},
    BroadcastReg<float>{src1Ptr, 1, 1}
);
// 内部：vbrcb处理1个标量
// 然后：1次vadd调用，硬件repeat=1

// 示例3：1000行的行向广播（Scheduler负责拆分）
for (int i = 0; i < 1000; i += 255) {
    int batch = std::min(255, 1000 - i);
    uint32_t offset = i * 64;
    VRowExpandAdd(
        VecReg<float>{dstPtr + offset, batch, 64, 8},
        VecReg<float>{src0Ptr + offset, batch, 64, 8},
        BroadcastReg<float>{src1Ptr + i, batch, 1}
    );
}
```

### 4.4 列向广播（Col Expand）

```cpp
// 列向广播加法（单次操作）
template <typename T>
void VColExpandAdd(
    VecReg<T> dst,
    VecReg<T> src0,
    __ubuf__ T *src1_row  // 一行标量（LanesPerReg个）
);

// 语义：dst[lane_j] = src0[lane_j] + src1_row[lane_j]
//       src1_row 是一行标量，与dst/src0的lanes一一对应
// 处理：1个VecReg（256B），执行1次vadd
```

**使用示例：**
```cpp
// 单次操作：一行标量与一行数据相加
VecReg<float> dst{dstPtr, 64, 8};
VecReg<float> src0{src0Ptr, 64, 8};
__ubuf__ float *src1_row = src1Ptr;  // 64个标量

VColExpandAdd(dst, src0, src1_row);
// 硬件执行：1次vadd，src1_row与dst/src0逐lane相加

// 240行的列向广播：Scheduler负责循环
for (int i = 0; i < 240; i++) {
    uint32_t offset = i * 8 * LanesPerBlock<float>;
    VColExpandAdd(
        VecReg<float>{dstPtr + offset, 64, 8},
        VecReg<float>{src0Ptr + offset, 64, 8},
        src1Ptr  // 所有行共享同一行标量
    );
}
```

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
         1, 1, 0,                    // 标准模式
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

### 5.2 VRowExpandAdd的内部实现（利用硬件repeat）

```cpp
template <typename T>
void VRowExpandAdd(
    VecReg<T> dst,
    VecReg<T> src0,
    BroadcastReg<T> src1
) {
    // 前置检查
    PTO_ASSERT(dst.num_regs == src0.num_regs);
    PTO_ASSERT(dst.num_regs == src1.num_scalars);
    PTO_ASSERT(dst.num_regs <= 255);  // 硬件repeat上限
    PTO_ASSERT(dst.active_lanes == src0.active_lanes);
    PTO_ASSERT(dst.active_lanes <= LanesPerReg<T>);

    // 1. 使用vbrcb将标量数组扩展到Block数组
    //    每个标量 → 1个Block（LanesPerBlock<T>个相同值）
    __ubuf__ T *expanded = AllocTempUB<T>(src1.num_scalars * LanesPerBlock<T>);

    uint32_t num_vbrcb_repeats = (src1.num_scalars + 7) / 8;  // 每8个标量一个vbrcb repeat

    // 分批处理vbrcb，避免临时UB溢出（每批最多30个repeat）
    for (uint32_t i = 0; i < num_vbrcb_repeats; i += 30) {
        uint32_t batch_size = std::min(30u, num_vbrcb_repeats - i);

        vbrcb(expanded + i * 8 * LanesPerBlock<T>,
              src1.scalars + i * 8 * src1.scalar_stride,
              1,                    // blk_stride
              BLOCKS_PER_REG,       // rep_stride = 8
              batch_size);          // repeat

        pipe_barrier(PIPE_V);
    }

    // 2. 执行单次vadd（硬件repeat=num_regs）
    //    expanded的stride = 1 block，硬件自动将每个Block重复8次填满VecReg
    vadd(dst.ptr, src0.ptr, expanded,
         dst.num_regs,                    // repeat = num_regs（1-255）
         1, 1, 0,                         // 行向广播模式
         dst.reg_stride_blocks,           // dst stride (blocks)
         src0.reg_stride_blocks,          // src0 stride (blocks)
         1);                              // src1 stride = 1 block
}
```

**关键点：**
1. **vbrcb分批处理**：每批最多30个repeat（避免临时UB溢出）
2. **vadd repeat = num_regs**：充分利用硬件repeat能力（1-255）
3. **无软件循环**：vadd只调用1次，不循环
4. **性能优势**：240行广播只需1次vadd调用，而不是240次

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
    VRowExpandAdd(
        VecReg<float>{dst, rows, cols, 8},
        VecReg<float>{src0, rows, cols, 8},
        BroadcastReg<float>{src1_scalars, rows, 1}
    );
    // 1次VRowExpandAdd调用，硬件repeat=240
    // 内部vbrcb分批处理，vadd利用硬件repeat
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

        VRowExpandAdd(
            VecReg<float>{dst + offset, batch, cols, 8},
            VecReg<float>{src0 + offset, batch, cols, 8},
            BroadcastReg<float>{src1_scalars + i, batch, 1}
        );
    }
    // 循环4次，每次VRowExpandAdd利用硬件repeat
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

### 6.1 直观的硬件映射 ✅

```cpp
// 用户思维模型：
VecReg = 256B寄存器 = 8个Block = 64个Lane (FP32)

// 硬件执行：
repeat = 处理一个VecReg
block = 32B的基本单位
lane = 单个数据元素

// 完美对应！
```

### 6.2 清晰的广播语义 ✅

```cpp
// 旧设计：语义混乱
VecTile<float> src1{ptr, 240, 8, 8};
// 这是什么？240×8矩阵？为什么是8？

// 新设计：语义清晰
BroadcastReg<float> src1{ptr, 240, BroadcastAxis::ROW};
// 240个标量，行向广播，一目了然！
```

### 6.3 封装硬件细节 ✅

```cpp
// 旧设计：vbrcb暴露给用户
vbrcb(tmpPtr, src1Ptr, 1, 8, 30);
pipe_barrier(PIPE_V);
VRowExpandAdd(..., tmpPtr, ...);

// 新设计：vbrcb在内部
VRowExpandAdd(dst, src0, BroadcastReg{src1Ptr, 240, ROW});
// 用户完全不需要知道vbrcb的存在
```

### 6.4 类型安全 ✅

```cpp
// 编译期区分不同的操作
void VAdd(VecReg, VecReg, VecReg);                  // 普通加法
void VRowExpandAdd(VecReg, VecReg, BroadcastReg);   // 行向广播
void VColExpandAdd(VecReg, VecReg, BroadcastReg);   // 列向广播

// 不会混淆！编译器会检查类型
VAdd(dst, src0, BroadcastReg{...});  // 编译错误！
```

### 6.5 易于理解和教学 ✅

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
| **stride对齐** | 必须32B对齐 | `stride_blocks * 32` | 硬件错误 |
| **指针对齐** | 建议32B对齐 | UB地址对齐 | 性能下降 |
| **active_lanes** | ≤ LanesPerReg | 不能超过VecReg容量 | 硬件错误 |
| **stride上限** | stride_blocks ≤ 255 | 8-bit限制 | 硬件错误 |

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
   stride_blocks * 32 % 32 == 0  // 自动满足
   // 但stride_blocks必须 <= 255
   ```

4. **Stride上限**（按数据类型）：
   ```cpp
   // FP32 (4B):
   stride_blocks <= 255
   // 对应最大stride: 255 * 32 = 8160B = 2040个float

   // FP16 (2B):
   stride_blocks <= 255
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
VAdd(VecReg<float>{ptr, 64, 8}, ...);

// ✅ 合法：31 lanes + mask
VAdd(VecReg<float>{ptr, 31, 8}, ..., LaneMask<float>::FromCount(31));

// ❌ 非法：超过64 lanes
VAdd(VecReg<float>{ptr, 128, 8}, ...);  // FP32最多64 lanes

// ❌ 非法：stride超限
VAdd(VecReg<float>{ptr, 64, 256}, ...);  // stride_blocks > 255

// ❌ 非法：lane数不一致
VAdd(VecReg<float>{dstPtr, 64, 8},
     VecReg<float>{src0Ptr, 32, 8},  // 32 != 64
     VecReg<float>{src1Ptr, 64, 8});
```

#### 7.2.2 VRowExpandAdd约束

```cpp
void VRowExpandAdd(VecReg<T> dst, VecReg<T> src0, BroadcastReg<T> src1);
```

**约束条件：**

1. **基本约束**（继承自VAdd）：
   - dst.active_lanes == src0.active_lanes
   - active_lanes <= LanesPerReg<T>
   - stride_blocks <= 255

2. **vbrcb约束**：
   ```cpp
   // vbrcb需要临时buffer
   // FP32: 8个元素 = 32B
   // FP16: 16个元素 = 32B
   // 通常在栈上分配，无特殊限制
   ```

3. **标量指针对齐**：
   ```cpp
   // src1.scalar建议4B对齐（FP32）或2B对齐（FP16）
   // 不强制，但对齐性能更好
   ```

**示例：合法与非法用法**

```cpp
// ✅ 合法：标准行向广播
VRowExpandAdd(
    VecReg<float>{dstPtr, 64, 8},
    VecReg<float>{src0Ptr, 64, 8},
    BroadcastReg<float>{scalarPtr, 1}
);

// ✅ 合法：非对齐lanes + mask
VRowExpandAdd(
    VecReg<float>{dstPtr, 31, 8},
    VecReg<float>{src0Ptr, 31, 8},
    BroadcastReg<float>{scalarPtr, 1}
);

// ❌ 非法：dst和src0 lanes不一致
VRowExpandAdd(
    VecReg<float>{dstPtr, 64, 8},
    VecReg<float>{src0Ptr, 32, 8},  // 不一致
    BroadcastReg<float>{scalarPtr, 1}
);
```

#### 7.2.3 VColExpandAdd约束

```cpp
void VColExpandAdd(VecReg<T> dst, VecReg<T> src0, __ubuf__ T *src1_row);
```

**约束条件：**

1. **基本约束**（继承自VAdd）：
   - dst.active_lanes == src0.active_lanes
   - active_lanes <= LanesPerReg<T>
   - stride_blocks <= 255

2. **src1_row长度**：
   ```cpp
   // src1_row必须至少包含active_lanes个元素
   // 例如：active_lanes=64，src1_row至少64个float
   ```

3. **src1_row对齐**：
   ```cpp
   // 建议32B对齐，性能更好
   // 不强制，但非对齐可能导致性能下降
   ```

**示例：合法与非法用法**

```cpp
// ✅ 合法：标准列向广播
__ubuf__ float src1_row[64];  // 64个标量
VColExpandAdd(
    VecReg<float>{dstPtr, 64, 8},
    VecReg<float>{src0Ptr, 64, 8},
    src1_row
);

// ✅ 合法：非对齐lanes
__ubuf__ float src1_row[31];  // 31个标量
VColExpandAdd(
    VecReg<float>{dstPtr, 31, 8},
    VecReg<float>{src0Ptr, 31, 8},
    src1_row
);

// ❌ 非法：src1_row长度不足
__ubuf__ float src1_row[32];  // 只有32个
VColExpandAdd(
    VecReg<float>{dstPtr, 64, 8},  // 需要64个
    VecReg<float>{src0Ptr, 64, 8},
    src1_row  // 长度不足！
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
    VAdd(VecReg<float>{ptr + offset, 64, 8}, ...);
}

// ✅ 合法：不规则访问
for (int i = 0; i < rows; i++) {
    uint32_t offset = index_array[i];  // 不规则
    VAdd(VecReg<float>{ptr + offset, 64, 8}, ...);
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
VAdd(VecReg<float>{ptr, 64, 8}, ...);

// 可接受：自然对齐（4B for float）
__ubuf__ float *ptr = malloc(size);
VAdd(VecReg<float>{ptr, 64, 8}, ...);  // 性能略降

// 避免：未对齐
__ubuf__ float *ptr = (float*)((char*)base + 1);
VAdd(VecReg<float>{ptr, 64, 8}, ...);  // 性能显著下降
```

#### 7.5.2 Stride建议

```cpp
// 最佳：stride = LanesPerReg（连续访问）
VAdd(VecReg<float>{ptr, 64, 8}, ...);  // stride = 64 elements

// 可接受：stride > LanesPerReg（有间隔）
VAdd(VecReg<float>{ptr, 64, 16}, ...);  // stride = 128 elements

// 避免：stride过大（缓存不友好）
VAdd(VecReg<float>{ptr, 64, 255}, ...);  // stride = 2040 elements
```

#### 7.5.3 循环展开建议

```cpp
// 好：适度展开（2-4次）
for (int i = 0; i < rows; i += 2) {
    VAdd(VecReg<float>{ptr + i*64, 64, 8}, ...);
    VAdd(VecReg<float>{ptr + (i+1)*64, 64, 8}, ...);
}

// 避免：过度展开（>8次）
// 可能导致寄存器压力过大
```

### 7.6 常见错误和调试

#### 7.6.1 常见错误

```cpp
// 错误1：stride超限
VecReg<float> reg{ptr, 64, 300};  // ❌ 300 > 255
// 解决：拆分为多个小stride的操作

// 错误2：lanes超限
VecReg<float> reg{ptr, 128, 8};  // ❌ FP32最多64 lanes
// 解决：拆分为多次操作

// 错误3：lanes不一致
VAdd(VecReg<float>{dst, 64, 8},
     VecReg<float>{src0, 32, 8},  // ❌ 32 != 64
     VecReg<float>{src1, 64, 8});
// 解决：确保所有操作数lanes一致

// 错误4：忘记mask
VAdd(VecReg<float>{ptr, 31, 8}, ...);  // ⚠️ 缺少mask
// 解决：添加mask
VAdd(VecReg<float>{ptr, 31, 8}, ..., LaneMask<float>::FromCount(31));

// 错误5：指针越界
for (int i = 0; i < rows; i++) {
    VAdd(VecReg<float>{ptr + i*64, 64, 8}, ...);
}
// 如果rows*64 > buffer_size，会越界
// 解决：检查边界条件
```

#### 7.6.2 调试技巧

```cpp
// 技巧1：添加断言
void MyScheduler(...) {
    PTO_ASSERT(rows * 64 <= buffer_size);  // 检查越界
    PTO_ASSERT(stride_blocks <= 255);      // 检查stride

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
VAdd(VecReg<float>{ptr, 64, 8}, ...);  // 单次测试

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
| active_lanes | ≤ LanesPerReg | 运行时 | 硬件错误 |
| stride_blocks | ≤ 255 | 运行时 | 硬件错误 |
| stride对齐 | 32B对齐 | 运行时 | 硬件错误 |
| lanes一致性 | dst == src0 == src1 | 运行时 | 未定义行为 |
| 指针对齐 | 建议32B | 运行时 | 性能下降 |
| UB空间 | 芯片相关（256KB-512KB） | 运行时 | 内存错误 |
| 循环次数 | 无限制 | N/A | N/A |

---

## 8. 迁移策略

### 7.1 渐进式迁移

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
    void VAdd(VecReg<T>, VecReg<T>, VecReg<T>);
    void VRowExpandAdd(VecReg<T>, VecReg<T>, BroadcastReg<T>);
}

// 旧API保持不变
namespace vtile {
    void VAdd(VecTile<T>, VecTile<T>, VecTile<T>);
}
```

**阶段3：提供转换工具**
```cpp
// VecTile → VecReg 转换
template <typename T>
VecReg<T> ToVecReg(const VecTile<T>& tile) {
    return VecReg<T>{
        tile.ptr,
        tile.rows,
        tile.cols,
        static_cast<uint16_t>((tile.ld * sizeof(T)) / BLOCK_BYTES)
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

### 7.2 性能验证

在迁移前需要验证：
1. 新API的性能与旧API相同
2. vbrcb封装没有引入额外开销
3. 编译后的汇编代码一致

---

## 8. 示例代码

### 8.1 矩阵加法

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

### 8.2 行向广播

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
    BroadcastReg<float> src1_reg{src1_scalars, rows, BroadcastAxis::ROW};

    VRowExpandAdd(dst_reg, src0_reg, src1_reg);
    // vbrcb在内部自动处理
}
```

### 8.3 列向广播

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
    BroadcastReg<float> src1_reg{src1_scalars, cols, BroadcastAxis::COL};

    VColExpandAdd(dst_reg, src0_reg, src1_reg);
}
```

### 8.4 非对齐数据

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

## 9. 文档和教学

### 9.1 用户文档大纲

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

### 9.2 教学示例

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
BroadcastReg<float> scalars{ptr, 240, BroadcastAxis::ROW};
// 240个标量，每个广播到一行
```

---

## 10. 总结

### 10.1 设计优势

1. **直观性**：Reg-Block-Lane模型与硬件结构完美对应
2. **清晰性**：BroadcastReg明确表达广播语义
3. **封装性**：vbrcb等硬件细节对用户透明
4. **安全性**：类型系统区分不同操作
5. **可扩展性**：易于添加新的广播模式和操作

### 10.2 实施建议

1. **原型验证**：在新命名空间实现VecReg/BroadcastReg
2. **性能测试**：确保无性能退化
3. **文档先行**：详细解释Reg-Block-Lane模型
4. **渐进迁移**：保持向后兼容，逐步迁移
5. **用户反馈**：收集使用体验，持续改进

### 10.3 预期收益

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
