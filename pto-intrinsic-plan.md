## A2/A3 TBinOp 无循环 Intrinsic 化改造方案（含公开 Low-Level API）

### 摘要
目标是把 `include/pto/npu/a2a3/TBinOp.hpp` 的“循环切分调度”与“单次 CCE intrinsic 发射”解耦，新增一个最小粒度（single-issue）的公开 API，直接映射 `vadd/vsub/vmul/vmax/vmin` 的 `repeat/blockStride/repeatStride` 参数，不在该 low-level API 内做任何循环。
同时保留现有高层 TADD/TMUL 等 API 兼容路径（可继续循环），实现 Both + Intrinsic-Only API。

### 参数语义
1. repeat: 发射 repeat 次数，范围 0..255；repeat=0 仅用于 count mode。
2. block stride（dst/src0/src1）: 同一 repeat 内相邻 block 的步长，单位 block(32B)。
3. repeat stride（dst/src0/src1）: 相邻 repeat 的步长，单位 block(32B)。
4. 单次 issue 粒度: 一次调用覆盖 repeat * REPEAT_BYTE 的逻辑窗口，尾部由 mask/count 裁剪。

### 公共接口与内部接口改造
1. 新增公开 low-level API（A2/A3 专用）:
   TBIN_INTRIN<T, OpTag>(dst, src0, src1, BinIssueDesc desc)
2. BinIssueDesc 字段:
   - mask_mode (NORMAL / COUNT)
   - vector_count (COUNT 模式)
   - tail_elements (NORMAL tail 模式)
   - repeat
   - dst/src0/src1 block stride
   - dst/src0/src1 repeat stride
   - repeat_stride_mode / stride_size_mode (默认 false)
3. 在 a2a3/TBinOp.hpp 增加 BinaryIssueOnce:
   - 仅参数校验 + mask 设置 + Op::BinInstr 一次调用
   - 不含循环
4. 在 TAdd/TSub/TMul/TMax/TMin 增加 IssueByDesc，复用现有 BinInstr。

### 失败策略
1. low-level API 不做自动 split，不隐式循环补偿。
2. repeat/stride 越界直接报错。
3. COUNT 模式要求 repeat=0。

### 测试验收
1. 正例: repeat=1/8/255、连续 stride、广播 block stride=0、非连续 repeat stride。
2. 负例: repeat=256、stride 越界、COUNT + repeat!=0。
3. 回归: 现有 TADD/TMUL/TSUB/TMAX/TMIN 行为不变。

### 假设与默认
1. 先落地 A2/A3(MEMORY_BASE)。
2. 默认 repeatStrideMode=false, strideSizeMode=false，stride 单位 block(32B)。
3. 旧高级 API 保持兼容；无循环要求仅作用于新增 low-level API。
