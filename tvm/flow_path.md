# 总体流程

┌─────────────────────────────────────────────────────────┐
│  1. 前端层 (Frontend)                                    │
│     • Tensor Expression (TE)                            │
│     • TVMScript (TIR)                                    │
│     • Relax (高级前端)                                    │
└──────────────────┬──────────────────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────────────────┐
│  2. 中间表示层 (IR Layer)                                │
│     • TensorIR (TIR) - 核心 IR                          │
│     • IRModule - 包含多个函数                            │
└──────────────────┬──────────────────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────────────────┐
│  3. 调度层 (Schedule Layer)                             │
│     • 循环变换 (split, fuse, reorder)                    │
│     • 线程绑定 (bind to threadIdx/blockIdx)              │
│     • 内存优化 (cache_read, cache_write)                │
│     • 向量化 (vectorize, unroll)                        │
└──────────────────┬──────────────────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────────────────┐
│  4. 编译流程 (Compilation Pipeline)                      │
│     • TIR 优化 Pass                                      │
│     • Lower (降低抽象层次)                               │
│     • 代码生成 (LLVM IR / Metal / CUDA)                   │
│     • 链接 (生成 .so/.dylib)                             │
└──────────────────┬──────────────────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────────────────┐
│  5. 运行时执行 (Runtime Execution)                      │
│     • 加载模块                                            │
│     • 内存管理 (分配设备内存)                             │
│     • 执行内核                                            │
│     • 结果获取                                            │
└─────────────────────────────────────────────────────────┘

# TE

```TE Expression → create_prim_func() → TIR → Schedule → Compile → Execute```

类似于numpy，属于高级表达

# TIRScript

```TVMScript → TIR/Relax → Schedule → Compile → Execute```

需要导入头文件

```
from tvm.script import ir as I
from tvm.script import tir as T
from tvm.script import relax as R
```

## TIR PrimFunc

```python
@T.prim_func
```

### 参数

```
T.Buffer(shape, dtype) #读写的内存视图
M: T.int32.            #标量动态参数
p: T.handle            #指针
#可以在函数体内用T.match_buffer(p, (M, N), dtype="float16")绑定它
x = T.int32(2)         #常量
```

### 函数属性

```
T.func_attr({"global_symbol": "name"})
```

* global_symbol：导出符号名（编译后符号名）

* tir.noalias：从那参数buffers不会别名

* target：某些pipeline会塞target信息，一般不手写

### 语句系统

#### 循环（可通过annotations（字典）参数指定一些信息，如展开程度、并行度等）

for i in range(): 与python的语法一致（语法糖）

for i in T.serial(0, 100, step=2): 串行循环

for i in T.parallel(): 并行循环

for i in T.vectorized(): 向量化循环

for i in T.unroll(): 循环展开

for i in T.thread_binding(0, 100, thread="threadIdx.x"): 线程绑定，指定线程标签

for i in T.grid(): 多重循环（语法糖）

#### if/else

#### evaluate

执行一个表达式，若无副作用，则实际执行过程中会被优化掉

### Block系统

#### with T.block("name")

Block表示一个可调度计算单元

#### 轴系统

vi = T.axis.spatial(128, i)：128代表这个轴的范围，为[0, 128)，i表示该轴取到的具体索引值

该轴为空间轴，对应输出张量上的某一维

vk = T.axis.reduce(128, k)：归约轴，代表沿着一维做归约

要有T.init()对写位置做初始化，之后在归约轴上做归约（只在归约循环的第一次迭代中执行）

T.axis.remap("SSR", [i, j, k])：一次声明多个轴，S = spatial, R = reduce

#### T.where(predicate)

只有当predicate为真时，这一次block实例才会执行

### Buffer与内存管理系统

#### T.alloc_buffer(shape, dtype, scope)

scope：存储域（如 "global"、"shared"、"local"）

函数/块 内的临时缓冲区

#### T.match_buffer(handle, shape, dtype)

把T.handle绑定成T.buffer视图

### 函数外部调用

#### 调用外部C/Runtime函数

```python
T.call_extern("int32", "my_extern", arg0, arg1)
```

## Relax

使用SSA格式

SSA：静态单赋值形式，是一种中间表示形式，每个变量只能被赋值一次，使用版本号来区分同一个变量的不同定义

```python
@R.function
```

### 基础语法

#### 常量

R.const(1, "int32")

#### R.Tensor(shape, dtype)

shape可以是静态常量，也可以是一个符号，等到运行时才知道具体数值

#### R.Tuple

R.Tuple(R.Tensor((n, ), "int32"), R.Tensor((n, ), "int32"))

#### R.Object()

表示任意对象

#### dataflow block

```python
with R.dataflow():
    y = R.add(x, )
    z = R.multiply(y, y)
    R.output(z)
return z
```

必须要有R.output，而且只有output的变量才能return，block内是SSA

里面的R.add等都是Relax Op

#### call_tir

Relax -> TIR

```python
out = R.call_tir(
    my_tir_func,
    (x, w),
    out_sinfo=R.Tensor((n, m), "float16")
)
```

#### 控制流

if/else、while

# script -> IRModule

## 无python函数

1. 得到AST

2. 通过FFI构建c++侧的IRBuilder类型的builder对象，维护一个frame栈

3. 遍历AST，根据不同token（tir，relax等）和类型（ClassDef，FunctionDef等）通过各种visit_方法，通过FFI在c++构建对应的frame，并在退出with的时候，构造对应的对象（如IRModule、TIR等）（最外层 frame 的结果保存到 builder->result，内层 frame 的结果交给父 frame，最终由最外层统一体现到 builder 上），并将结果保存到builder供python使用

4. 通过python侧的builder获得IRModule

## 有python函数

1. 得到 AST，并收集“可能作为 pyfunc 的方法

2. 通过 FFI 构建 C++ 的 IRBuilder，维护 frame 栈

3.  遍历 AST，按 token + 类型 visit；pyfunc 只声明为 ExternFunc，不解析函数体

4. 通过 Python 的 builder 拿到 IRModule，再把 Python 可调用对象挂上去

5. 最后返回的是一个经过编译处理的类，能够实例化后直接运行
