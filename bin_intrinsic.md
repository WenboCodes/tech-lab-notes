# BinIntrinsic 设计与参数映射说明

## 1. 执行链路（Single-Issue 边界）
`TBIN_INTRIN` 在 A2/A3 后端的执行路径：

1. `TBIN_INTRIN<OpType>(...)`  
   `include/pto/common/pto_instr.hpp:96`
2. `TBIN_INTRIN_IMPL(...)`  
   `include/pto/npu/a2a3/TBinIntrin.hpp:71`
3. `BinaryIssueOnce(...)`  
   `include/pto/npu/a2a3/TBinOp.hpp:21`
4. `Op::BinInstr(...)` -> `vadd/vsub/vmul/vmax/vmin`  
   `include/pto/npu/a2a3/TAdd.hpp:29`

其中 `BinaryIssueOnce` 是最小粒度边界：一次调用只对应一次 CCE intrinsic 发射，不做循环拆分。

## 2. `BinIntrinsicDesc` 字段与 CCE 参数对照
定义位置：`include/pto/common/bin_intrinsic_desc.hpp:44`

| 字段 | 映射到 CCE 参数 | 单位 | 约束 | 说明 |
|---|---|---|---|---|
| `repeat` | `repeats` | 次 | `0..255` | intrinsic 重复发射次数（u8） |
| `dstBlockStride/src0BlockStride/src1BlockStride` | block stride | 32B block | `0..255` | 同一 repeat 内 block 间距 |
| `dstRepeatStride/src0RepeatStride/src1RepeatStride` | repeat stride | 32B block | `0..255` | 相邻 repeat 间距 |
| `maskMode` | mask mode | 枚举 | `Normal/Count` | 掩码语义选择 |
| `vectorCount` | `SetVectorCount` | 元素数 | Count 模式必须 `>0` | Count 模式有效长度 |
| `tailElements` | `SetContMaskByDType` | 元素数 | `<= 256/sizeof(T)` | Normal 模式尾裁剪 |
| `repeatStrideMode/strideSizeMode` | 扩展模式标志 | 布尔 | 当前必须 `false` | A2/A3 工具链当前不支持 |

## 3. 两种 Mask 语义
校验和执行逻辑在：`include/pto/npu/a2a3/TBinOp.hpp:25`

1. Count 模式
- 要求：`repeat == 0` 且 `vectorCount > 0`
- 动作：`set_mask_count()` + `SetVectorCount(vectorCount)` + 发射 intrinsic

2. Normal 模式
- 要求：`repeat > 0`
- 可选：`tailElements > 0` 时先 `SetContMaskByDType<T>(tailElements)`
- 限制：不允许 `repeat > 1 && tailElements > 0`（单次 issue 无法同时表达“多 repeat + tail”）

## 4. 最小粒度暴露原则
single-issue API 对外暴露的是“1 次 intrinsic 发射”的全部必要参数：

1. 覆盖窗口（满发射）：`repeat * (256 / sizeof(T))` 元素
2. 尾裁剪窗口（仅 Normal）：`tailElements`
3. 地址模式：block stride + repeat stride（dst/src0/src1 三路独立）

超过该粒度的工作（如 `repeat > 255`、复杂 tail 组合）由上层策略负责拆分为多次 `TBIN_INTRIN` 调用。

## 5. 为什么拆出 `bin_intrinsic_desc.hpp`
文件：`include/pto/common/bin_intrinsic_desc.hpp`

1. 轻量依赖，只包含 `<cstdint>`，可直接用于 host/gtest 侧参数校验测试。
2. 避免 `constants.hpp` 的设备限定符依赖（`__ubuf__`、`__gm__`）影响 host 编译。
3. `constants.hpp` 通过 include 复用定义，保持原有代码兼容：
   `include/pto/common/constants.hpp:13`

## 6. 典型配置示例（FP32，64x64）
示例来源：`tests/npu/a2a3/src/st/testcase/tadd/tadd_kernel.cpp:67`

- `elemPerRepeat = 256 / 4 = 64`
- `totalElements = 64 * 64 = 4096`
- `repeat = 4096 / 64 = 64`
- `dst/src0/src1 block stride = 1`
- `dst/src0/src1 repeat stride = 8`（连续布局）

这是“整 repeat、无 tail”的标准 intrinsic 直发场景。

## 7. 参数校验接口（便于单测）
定义位置：`include/pto/common/bin_intrinsic_desc.hpp:63`

- `ValidateBinIntrinsicDesc<T>(desc)`：返回 `BinIntrinsicDescStatus`
- `IsBinIntrinsicDescValid<T>(desc)`：返回 `bool`

已有负例单测示例：
`tests/npu/a2a3/src/st/testcase/tadd/main.cpp:254`
