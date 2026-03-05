# vbrcb 指令详细使用指南

## 1. 指令概述

### 1.1 指令原型

```cpp
vbrcb(__ubuf__ T *dst, __ubuf__ T *src,
      uint8_t dst_blk_stride,
      uint8_t dst_rep_stride,
      uint8_t repeat)
```

### 1.2 参数说明

| 参数 | 类型 | 说明 |
|-----|------|------|
| `dst` | `__ubuf__ T *` | 目标UB地址（输出） |
| `src` | `__ubuf__ T *` | 源UB地址（输入） |
| `dst_blk_stride` | `uint8_t` | 块内步长（通常为1，表示连续） |
| `dst_rep_stride` | `uint8_t` | repeat间步长（以32B块为单位） |
| `repeat` | `uint8_t` | 重复次数（1-255） |

### 1.3 功能说明

**vbrcb = Vector Broadcast Block**

vbrcb指令的核心功能是**将8个连续元素广播复制成一个256B的块**（一个repeat）。

**工作原理：**
- 每次从src读取8个元素
- 将这8个元素广播扩展为256B（一个完整的repeat）
- 重复执行repeat次

**内存布局转换示意：**
```
输入 src:  [a0, a1, a2, a3, a4, a5, a6, a7, b0, b1, b2, b3, b4, b5, b6, b7, ...]
                    ↓ vbrcb(dst, src, 1, 8, 2)
输出 dst:  [a0, a1, a2, a3, a4, a5, a6, a7, a0, a1, a2, a3, a4, a5, a6, a7, ...] (重复32次，共256B)
           [b0, b1, b2, b3, b4, b5, b6, b7, b0, b1, b2, b3, b4, b5, b6, b7, ...] (重复32次，共256B)
```

---

## 2. 基础使用示例

### 2.1 示例1：基础用法（FP32）

```cpp
// 场景：将8个float32元素广播为64个元素（256B）
__ubuf__ float *src = ...;  // [a, b, c, d, e, f, g, h]
__ubuf__ float *dst = ...;

// 参数计算
constexpr int elemPerRepeat = 256 / sizeof(float);  // 64
constexpr int vbrcbElem = 8;  // 每次读取8个元素

// 执行vbrcb
vbrcb(dst, src, 1, 8, 1);

// dst输出: [a,b,c,d,e,f,g,h, a,b,c,d,e,f,g,h, ... (重复8次)] = 64个元素
```

**参数解释：**
- `dst_blk_stride = 1`：块内连续写入
- `dst_rep_stride = 8`：每个repeat占8个32B块（8 × 32B = 256B）
- `repeat = 1`：执行1次广播

**内存布局详解（FP32）：**
```
src[0:7]:     [a, b, c, d, e, f, g, h]  (8个元素 = 32B)

dst[0:63]:    [a,b,c,d,e,f,g,h] × 8次  (64个元素 = 256B)
              ↑ 一个repeat的输出
```

### 2.2 示例2：多repeat（FP16）

```cpp
// 场景：将16个half元素广播（2个repeat）
__ubuf__ half *src = ...;  // 16个元素
__ubuf__ half *dst = ...;

// 参数计算
constexpr int elemPerRepeat = 256 / sizeof(half);  // 128
constexpr int vbrcbElem = 8;

// 执行vbrcb
vbrcb(dst, src, 1, 8, 2);

// 结果：
// repeat 0: src[0:7] 广播为 dst[0:127]     (128个元素)
// repeat 1: src[8:15] 广播为 dst[128:255]  (128个元素)
```

**内存布局详解（FP16）：**
```
src[0:15]:    [a0,a1,a2,a3,a4,a5,a6,a7, b0,b1,b2,b3,b4,b5,b6,b7]

dst[0:127]:   [a0,a1,a2,a3,a4,a5,a6,a7] × 16次  (repeat 0)
dst[128:255]: [b0,b1,b2,b3,b4,b5,b6,b7] × 16次  (repeat 1)
```

---

## 3. 实际应用场景

### 3.1 场景1：行向广播预处理

**问题：** 列主序的src1需要转换为行主序用于行向Expand操作

```cpp
// 场景：将列向量[240, 1]广播为[240, 64]（FP32）
__ubuf__ uint32_t *src1Ptr = ...;  // 240个元素（列向量）
__ubuf__ float *tmpPtr = ...;      // 临时缓冲区

// 计算repeat次数
unsigned validRow = 240;
unsigned repeatTimes = CeilDivision(validRow, 8);  // 240/8 = 30

// 执行vbrcb
vbrcb(tmpPtr, src1Ptr, 1, 8, 30);
pipe_barrier(PIPE_V);  // 等待完成

// 结果：
// src1[0:7]     → tmpPtr[0:63]       (第1个repeat)
// src1[8:15]    → tmpPtr[64:127]     (第2个repeat)
// ...
// src1[232:239] → tmpPtr[1856:1919]  (第30个repeat)
```

**使用流程：**
```
列主序src1 → vbrcb → 行主序tmpPtr → 用于算术运算
```

### 3.2 场景2：纯扩展操作

**问题：** 将一维向量扩展为二维矩阵

```cpp
// 场景：[1, M] → [M, 8]（M必须是8的倍数）
template <typename T>
void RowExpandBrcb(__ubuf__ T *dst, __ubuf__ T *src, int srcNumel) {
    constexpr int vbrcbElem = 8;
    constexpr int elemPerRepeat = 256 / sizeof(T);

    int repeat = srcNumel / vbrcbElem;  // 计算repeat次数

    // 执行vbrcb
    if (repeat <= 254) {
        vbrcb(dst, src, 1, 8, repeat);
    } else {
        // 处理超过254 repeat的情况（见下节）
    }
}
```

**示例：**
```cpp
// FP32: [1, 64] → [64, 8]
float src[64] = {a0, a1, ..., a63};
float dst[64 * 64];

vbrcb(dst, src, 1, 8, 8);  // 8个repeat

// 结果：
// dst[0:63]   = [a0,a1,a2,a3,a4,a5,a6,a7] × 8
// dst[64:127] = [a8,a9,a10,a11,a12,a13,a14,a15] × 8
// ...
```

---

## 4. 高级用法

### 4.1 处理大规模数据（超过255 repeat）

**问题：** repeat参数最大为255，如何处理更大的数据？

**解决方案：** 分批处理

```cpp
template <typename T>
void LargeVbrcb(__ubuf__ T *dst, __ubuf__ T *src, unsigned totalRows) {
    constexpr unsigned BRCB_REPEAT_MAX = 30;  // 每批最多30个repeat
    constexpr unsigned MAX_ROW = 240;         // 30 * 8 = 240行
    constexpr unsigned elemPerRepeat = 256 / sizeof(T);

    unsigned repeatTimes = CeilDivision(totalRows, 8);
    unsigned numLoop = repeatTimes / BRCB_REPEAT_MAX;
    unsigned numRemain = repeatTimes % BRCB_REPEAT_MAX;

    // 分批处理
    for (unsigned i = 0; i < numLoop; i++) {
        vbrcb(dst, src, 1, 8, BRCB_REPEAT_MAX);
        pipe_barrier(PIPE_V);

        dst += MAX_ROW * elemPerRepeat;  // 移动目标指针
        src += MAX_ROW;                   // 移动源指针
    }

    // 处理剩余
    if (numRemain > 0) {
        vbrcb(dst, src, 1, 8, numRemain);
        pipe_barrier(PIPE_V);
    }
}
```

**为什么使用30而不是255？**
- 临时UB缓冲区通常只有8KB
- 8KB = 8192B，可以存放32个repeat（32 × 256B）
- 留出余量，使用30个repeat（7680B）更安全

### 4.2 避免对齐问题

**问题：** 连续处理254个repeat时，偏移量可能不是32B对齐

```cpp
// 来自TRowExpand.hpp的实现
template <typename T>
void SafeVbrcb(__ubuf__ T *dst, __ubuf__ T *src, int totalRepeat) {
    constexpr int elemPerRepeat = 256 / sizeof(T);
    constexpr int vbrcbElem = 8;

    // 使用254而不是255，避免对齐问题
    constexpr int loop = totalRepeat / (REPEAT_MAX - 1);  // 254
    constexpr int remain = totalRepeat % (REPEAT_MAX - 1);

    for (int i = 0; i < loop; i++) {
        vbrcb(dst + i * (REPEAT_MAX - 1) * elemPerRepeat,
              src + i * (REPEAT_MAX - 1) * vbrcbElem,
              1, 8, REPEAT_MAX - 1);
    }

    if (remain > 0) {
        vbrcb(dst + loop * (REPEAT_MAX - 1) * elemPerRepeat,
              src + loop * (REPEAT_MAX - 1) * vbrcbElem,
              1, 8, remain);
    }
}
```

**关键点：**
- `REPEAT_MAX * vbrcbElem` 可能不是32B对齐
- 使用 `(REPEAT_MAX - 1)` 确保每次偏移都是32B对齐

---

## 5. 完整应用示例

### 5.1 行向Expand with vbrcb

```cpp
// 完整流程：dst[i,j] = src0[i,j] + src1[i]（src1为列主序）
void RowExpandAddWithVbrcb(__ubuf__ float *dst,
                           __ubuf__ float *src0,
                           __ubuf__ uint32_t *src1_col_major,
                           unsigned validRow,
                           unsigned validCol) {
    constexpr unsigned blockSizeElem = 8;  // FP32
    constexpr unsigned elemPerRepeat = 64;

    // 1. 计算vbrcb参数
    unsigned repeatTimes = CeilDivision(validRow, 8);

    // 2. 预展开src1到临时UB
    __ubuf__ float *tmpPtr = (__ubuf__ float *)(TMP_UB_OFFSET);
    __ubuf__ uint32_t *tmpPtr_ = (__ubuf__ uint32_t *)(TMP_UB_OFFSET);

    vbrcb(tmpPtr_, src1_col_major, 1, 8, repeatTimes);
    pipe_barrier(PIPE_V);

    // 3. 使用tmpPtr作为row-major的src1进行运算
    auto dstTile = MakeVecTile(dst, validRow, validCol, validCol);
    auto src0Tile = MakeVecTile(src0, validRow, validCol, validCol);
    auto src1Tile = MakeVecTile(tmpPtr, validRow, blockSizeElem, blockSizeElem);

    // 4. 执行行向广播加法
    auto is = Repeat(validRow);
    vtile::VRowExpandAdd(is, dstTile, src0Tile, src1Tile);
}
```

### 5.2 来自TRowExpandBinOp.hpp的实际代码

```cpp
// 处理validRow < 256的情况
if (validRow < 256) {
    unsigned repeatTimes = CeilDivision(validRow, 8);

    // vbrcb预展开
    vbrcb(tmpPtr_, src1Ptr, 1, 8, repeatTimes);
    pipe_barrier(PIPE_V);

    // 选择执行模式
    if (useCountMode) {
        TRowExpandBinaryCountMode<Op, T, blockSizeElem, DstRowStride, Src0RowStride>(
            dstPtr, src0Ptr, tmpPtr, validRow, validCol);
    } else {
        TRowExpandBinaryNormModeTail<Op, T, elementsPerRepeat, blockSizeElem,
                                     DstRowStride, Src0RowStride>(
            dstPtr, src0Ptr, tmpPtr, validRow, validCol);
    }
}
```

### 5.3 处理validRow >= 256的情况

```cpp
// 大于256行时分批处理
constexpr unsigned BRCB_REPEAT_MAX = 30;
constexpr unsigned MAX_ROW = 240;  // 30 * 8

unsigned repeatTimes = CeilDivision(validRow, 8);
unsigned numLoop = repeatTimes / BRCB_REPEAT_MAX;
unsigned numRemain = repeatTimes % BRCB_REPEAT_MAX;

for (unsigned i = 0; i < numLoop; i++) {
    // 每批处理240行
    vbrcb(tmpPtr_, src1Ptr, 1, 8, BRCB_REPEAT_MAX);
    pipe_barrier(PIPE_V);

    TRowExpandBinaryNormModeTail<Op, T, elementsPerRepeat, blockSizeElem,
                                 DstRowStride, Src0RowStride>(
        dstPtr, src0Ptr, tmpPtr, MAX_ROW, validCol);
    pipe_barrier(PIPE_V);

    dstPtr += MAX_ROW * DstRowStride;
    src0Ptr += MAX_ROW * Src0RowStride;
    src1Ptr += MAX_ROW;
}

// 处理剩余行
if (numRemain > 0) {
    vbrcb(tmpPtr_, src1Ptr, 1, 8, numRemain);
    pipe_barrier(PIPE_V);

    TRowExpandBinaryNormModeTail<Op, T, elementsPerRepeat, blockSizeElem,
                                 DstRowStride, Src0RowStride>(
        dstPtr, src0Ptr, tmpPtr, validRow % MAX_ROW, validCol);
}
```

---

## 6. 关键约束与注意事项

### 6.1 对齐要求

| 约束项 | 要求 | 说明 |
|-------|------|------|
| src地址 | 32B对齐 | 必须满足，否则硬件错误 |
| 读取单位 | 8个元素 | 固定值，不可更改 |
| 元素大小 | `8 * sizeof(T)` 需32B对齐 | FP32: 32B ✓, FP16: 16B ✗ |

**FP16特殊处理：**
```cpp
// FP16需要16个元素才能32B对齐
// 因此实际使用时需要特殊处理
using BrcbType = std::conditional_t<sizeof(T) == sizeof(uint16_t), uint16_t,
                 std::conditional_t<sizeof(T) == sizeof(uint32_t), uint32_t, T>>;
```

### 6.2 数据类型支持

| 类型 | sizeof | 8个元素 | 是否直接支持 |
|-----|--------|---------|------------|
| float (b32) | 4B | 32B | ✓ |
| half (b16) | 2B | 16B | 需特殊处理 |
| int32 | 4B | 32B | ✓ |
| int16 | 2B | 16B | 需特殊处理 |

### 6.3 repeat限制

- **硬件限制**：单次最多255个repeat
- **实际使用**：建议不超过30-50个repeat（受临时UB大小限制）
- **超限处理**：分批执行，每批之间需要`pipe_barrier(PIPE_V)`

### 6.4 临时缓冲区

```cpp
// 通常使用8KB临时UB
#define TMP_UB_OFFSET 0x...  // 8KB临时缓冲区地址

// 容量计算
constexpr int TMP_UB_SIZE = 8192;  // 8KB
constexpr int MAX_REPEATS = TMP_UB_SIZE / 256;  // 32个repeat
```

**注意事项：**
- 临时UB可能被多个操作共享
- 使用前确保没有冲突
- 使用后可以被其他操作复用

### 6.5 Pipeline同步

```cpp
// vbrcb是异步操作，必须同步
vbrcb(dst, src, 1, 8, repeat);
pipe_barrier(PIPE_V);  // 等待向量计算单元完成

// 然后才能使用dst的数据
```

**同步点：**
- vbrcb执行后
- 使用dst数据前
- 切换到其他pipeline操作前

---

## 7. 性能优化建议

### 7.1 避免不必要的vbrcb

```cpp
// 优先使用row-major布局，避免vbrcb
if constexpr (TileDataSrc1::isRowMajor) {
    // 直接使用，无需vbrcb
    TRowExpandBinaryInstr32B(...);
} else {
    // 需要vbrcb预展开
    vbrcb(...);
    TRowExpandBinaryInstr(...);
}
```

### 7.2 批量处理

```cpp
// 好：一次vbrcb处理更多数据
vbrcb(dst, src, 1, 8, 30);  // 处理240行

// 差：多次vbrcb处理少量数据
for (int i = 0; i < 30; i++) {
    vbrcb(dst + i * 256, src + i * 8, 1, 8, 1);
    pipe_barrier(PIPE_V);
}
```

### 7.3 临时缓冲区复用

```cpp
// 同一个临时UB可以在不同操作间复用
__ubuf__ float *tmpPtr = (__ubuf__ float *)(TMP_UB_OFFSET);

// 操作1
vbrcb(tmpPtr, src1, 1, 8, repeat1);
pipe_barrier(PIPE_V);
// 使用tmpPtr...

// 操作2（可以复用同一个tmpPtr）
vbrcb(tmpPtr, src2, 1, 8, repeat2);
pipe_barrier(PIPE_V);
// 使用tmpPtr...
```

### 7.4 编译期优化

```cpp
// 利用编译期常量选择最优路径
if constexpr (isBroadcastSupportType && isStaticShape && isBroadcast) {
    // 快路径：使用vbrcb
    TRowExpandBrcb<T, TileDataDst, TileDataSrc>(dst, src);
} else {
    // 通用路径：使用vector_dup
    TRowExpand<TileDataDst, TileDataSrc>(dst, src, validRow, validCol);
}
```

---

## 8. 常见问题

### Q1: vbrcb和vector_dup的区别？

**vbrcb:**
- 批量广播，一次处理8个元素
- 输出256B（一个repeat）
- 适合大规模数据转换
- 需要32B对齐

**vector_dup:**
- 单个标量广播
- 逐行处理
- 适合小规模或动态数据
- 对齐要求较低

### Q2: 为什么vbrcb固定读取8个元素？

这是硬件设计决定的：
- 8个元素 × 4B (FP32) = 32B（一个block）
- 硬件以32B为基本单位处理
- 保证对齐和效率

### Q3: 如何选择vbrcb还是其他方法？

```cpp
// 决策树
if (数据量大 && src是32B对齐 && 支持的数据类型) {
    使用vbrcb;  // 高效批量处理
} else if (数据量小 || 动态形状) {
    使用vector_dup;  // 灵活但较慢
} else {
    使用copy_ubuf_to_ubuf;  // 简单复制
}
```

### Q4: vbrcb失败的常见原因？

1. **src地址未32B对齐**
2. **repeat超过255**
3. **临时缓冲区不足**
4. **缺少pipe_barrier同步**
5. **数据类型不匹配**

---

## 9. 总结

### 9.1 核心要点

1. **功能**：8个元素 → 256B广播块
2. **参数**：`vbrcb(dst, src, 1, 8, repeat)`
3. **约束**：32B对齐，repeat ≤ 255
4. **同步**：必须`pipe_barrier(PIPE_V)`

### 9.2 典型使用模式

```cpp
// 标准模式
vbrcb(tmpPtr, src, 1, 8, repeatTimes);
pipe_barrier(PIPE_V);
// 使用tmpPtr...

// 分批模式（大数据）
for (batch) {
    vbrcb(dst, src, 1, 8, BATCH_SIZE);
    pipe_barrier(PIPE_V);
    // 处理这批数据...
    dst += offset;
    src += offset;
}
```

### 9.3 最佳实践

1. **优先row-major布局**，避免vbrcb开销
2. **批量处理**，减少vbrcb调用次数
3. **注意对齐**，确保src地址32B对齐
4. **及时同步**，vbrcb后立即pipe_barrier
5. **复用缓冲区**，节省UB空间

vbrcb是实现高效广播操作的关键硬件指令，正确使用可以显著提升性能。
