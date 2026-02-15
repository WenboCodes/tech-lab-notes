问题梳理：
1. Section 1的问题：
    * 题目改为PTO IR Reference
    * 第一段不需要提LLVM版本号

2. Section 2的问题：
    * type system的定义，element types与nv tileir保持一致。参考nv TileIR文档(https://docs.nvidia.com/cuda/tile-ir/latest/sections/types.html)
    * pto.ptr, pto.tile, pto.tile_buf标明待补充

3. Section 3的问题：
    * Address space的定义
    * Pipe的定义
    * Layout包括BLayout，SLayout和PadValue，参照pto-isa仓的介绍

4. Section4的问题：
    * 删除整个section

5. Section5的问题：
    * pto.make_tensor_view参考附录
    * pto.partition_view参考附录
    * pto.alloc_tile对一个tile buffer的生命，每一个alloc_tile对应一个独立的tile buffer。alloc_tile()可以接受一个输入参数，即代表起始地址。默认没有输入参数，代表地址由编译器分别。
    * pto.bind_tile不确定是否保留，请标明`待讨论`
    * pto.subset改为pto.subview，即代表了新生命的tile buffer是输入tile buffer的一部分。
    * pto.tload/tstore的定义修改和补充，见附录
    * pto.ttrans需要一个tmp的输入，这个重点标注一下`待讨论`
    * 标注一下pto.tmov和pto.copy区别是啥
    * 删除掉pto.load_dps和pto.store_dps和pto.mov_dps
    * pto.tmatmul去掉bias输入
    * 去掉所有以_dps结尾的api接口相关定义
    *` 5.16 Control Flow Operations`修改为`5.16 CV相关 operation`
    


6. 增加一个Rationale的section
PTO IR中的Tile与Tensor不同，不是SSA格式的。其pto.tile_buf是一层buffer的语义，而不是value。这么做的目的是为了把tiling或内存分配的工作交给程序员或上层框架，PTO AS的pass仅关注调度和排流水。内存分配在编译器里面是一个NP-hard的问题，而排流水同时也是一个NP-hard的问题。这两个NP-hard的问题在一起会对编译pass的结果产生非常大的困扰。因此，在PTO IR的pipeline设计中，buffer复用的问题由用户解决，通过`pto.alloc_tile`来声明tile.buf的生命周期。

##附录：
pto.make_tensor_view
功能：通过指针建立 GlobalTensor 的构造函数 (Constructor)。定义全局内存 (Global Memory) 中原始数据的“物理大底座”。
● 详细解释：此指令不涉及数据搬运，仅用于声明数据在内存中的排列规则（Strides）。它是所有视图变换的基准，确保后续的切片操作能准确定位物理地址。
● 分型映射：若输入视图带有特定 Stride 模式，编译器在此阶段自动注入 Layout::NZ 等硬件提示，指导后续调用 DMA 分型搬运指令，实现最高效率的加载。
● 映射逻辑： 
    ○ 映射为 GlobalTensor 的 Stride<...> 模板参数。
    ○ 决定了 Tensor 的“视野边界”。
2.2 pto.partition_view
功能：逻辑窗口切分。在大视图上截取特定的计算区域，生成分块视图。
● 详细解释：无论 Shape 是静态还是动态，均通过 partition_view 捕获。它承载了 offsets（决定“从哪开始读”）和 sizes（决定“读多少”），其返回类型为 !pto.partition_tensor_view。
● 映射逻辑： 
    ○ 指针偏移：编译器自动生成 BasePtr + Offset。
    ○ 逻辑 Shape：映射为 GlobalTensor 的 Shape<...> 模板参数。
2.3 pto.tload
功能：物理搬运与维度塌缩 (Dimension Collapse)。
● 详细解释： 
    ○ 严格类型约束：仅接受 partition_tensor_view（逻辑高维）作为输入，输出必须为 tile_buf（物理 2D）。
    ○ 核心约束：partition_view 中所有 size 维度的乘积，必须等于 tile_buf 的 valid_row 与 valid_col 的乘积。
    ○ 语义映射：partition_tensor_view 是高维逻辑视图，而 tile_buf 是二维物理实体。pto.tload 完成了从 N 维到 2 维的线性映射。
3. 映射逻辑示例 (From IR to C++)
场景 A：完全静态 Shape (降维加载)
IR Expression:
// 1. 定义 5D 物理视图 (1, 1, 16, 1024, 1024)
%0 = pto.make_tensor_view %arg0, 
    shape = [1, 1, 16, 1024, 1024], 
    strides = [1048576, 1048576, 1048576, 1024, 1] 
    : !pto.tensor_view<1x1x16x1024x1024xf32>

// 2. 使用 partition_view 切出 5D 子视图，总元素量 = 1*1*16*16*16 = 4096
%1 = pto.partition_view %0, offsets = [0,0,0,0,0], sizes = [1, 1, 16, 16, 16] 
     : !pto.tensor_view<1x1x16x1024x1024xf32> -> !pto.partition_tensor_view<1x1x16x16x16xf32>

// 3. 执行 TLOAD，目标 Tile 256x16, 总容量 = 256*16 = 4096
// 满足约束：1*1*16*16*16 == 256*16
pto.tload ins(%1 : !pto.partition_tensor_view<1x1x16x16x16xf32>) 
          outs(%tile : !pto.tile_buf<256x16xf32>)
Generated PTO C++:
// 编译器自动推导并将 5D 逻辑映射至硬件指令
using ShapeDim = Shape<1, 1, 16, 16, 16>;
using StrideDim = Stride<1, 1, 1048576, 1024, 1>;

GlobalTensor<float, ShapeDim, StrideDim> src(srcPtr);

// 触发硬件 TLOAD，完成高维数据到 256x16 Tile 的填充
TLOAD(src, tile);
场景 B：动态 Shape
IR Expression:
// sizes 使用了运行时变量 %v0, %v1
%1 = pto.partition_view %0, offsets = [%x, %y], sizes = [%v0, %v1] 
     : !pto.tensor_view<?x?xf32> -> !pto.partition_tensor_view<?x?xf32>
Generated PTO C++:
using ShapeDim = Shape<1, 1, 1, -1, -1>;
using StrideDim = Stride<1, 1, 1, -1, 1>;

GlobalTensor<float, ShapeDim, StrideDim> gQ(
    srcPtr + (x * stride_val + y), 
    ShapeDim(v0, v1), 
    StrideDim(stride_val)
);
