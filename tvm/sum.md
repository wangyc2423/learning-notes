**一、TVMScript → Relax/TIR IR：前端解析链**

- **Source / doc AST**
  
  - Source(program) 从函数/类源码拿字符串+位置信息。
  - Source.as_ast() 调 doc.parse(source_str)：
    - doc.parse 先用 Python 内置 ast.parse → Python AST；
    - 再通过注册表映射成 TVM 自己的 doc.AST（doc.Module/doc.FunctionDef 等），做版本隔离。

- **Parser / extra_vars**
  
  - core.parse(program, extra_vars, check_well_formed)：
    - 把函数/类的注解收集到 ann；
    - 用 Source + Parser(source, ann)；
    - Parser.parse(extra_vars)：
      - 把 extra_vars（T/R/tvm/闭包变量等）塞进 var_table；
      - 遍历 doc AST，结合 dispatch_token（"tir"/"relax"/"ir"/"pyfunc"）调用不同的 handler。
  - extra_vars 就是“解析时的全局环境”，让 eval_expr 能真正执行 T.match_buffer 之类的 Python 调用。

- **IRBuilder / frame 栈**
  
  - 外面包一层：with IRBuilder() as builder: parser.parse(...)。
  - 不同 IR 会在解析时开不同 frame：
    - TIR：with T.prim_func(): → PrimFuncFrame
    - TIR block：with T.block(): → BlockFrame
    - Relax：with R.function(): / with R.SeqExpr(): → Relax 对应 frame
    - IRModule：with I.ir_module(): → IRModuleFrame
  - 这些 frame 压在 IRBuilder 的栈里，退出 with 时拼成树：  
    外层 IRModule → 里面 PrimFunc/Function → 里面 Block/Stmt 等。
  - 最后 builder.get() 返回顶层对象（IRModule 或单个 PrimFunc/Function）。

- **装饰器如何选解析器**
  
  - @T.prim_func / @R.function / @I.ir_module 都只是入口，真正的解析依赖：
    
    - 函数/类 decorator 的 dispatch_token 属性（在 entry.py 里用 setattr(..., "dispatch_token", "tir"/"relax"/"ir")）；
    - Parser.visit_FunctionDef 里 get_dispatch_token(node) → "tir" 等；
    - dispatch.get(token="tir", type_name="FunctionDef") → 找到 TIR/Relax/IR 的 handler。
  
  - 在 @I.ir_module 的类里：
    
    - 类方法上的 @T.prim_func 在定义阶段不调用 parse，只给装饰器打上 dispatch_token="tir"；
    - 外层 @I.ir_module 调一次 parse(整个 class)，在解析 ClassDef 时，根据方法 decorator 再分派给 TIR/Relax/pyfunc 解析器。

---

**二、Target 与 tvm.tir.build：多设备与 codegen**

- **Target 数据结构**
  
  - Python：tvm.target.Target("llvm", host="llvm")；
  - C++：TargetNode 里有：
    - kind（"llvm"/"cuda"/"c" 等）；
    - attrs（mcpu/mtriple/keys/libs 等）；
    - host（host Target）；
    - features（由 target_parser 推导出的硬件能力）。
  - 字符串 "llvm -mcpu=... -keys=..." 通过 TargetInternal::FromRawString + TargetKind 注册的解析规则变成结构化 attrs。

- **TIR build 总流程（python/tvm/tir/build.py）**
  
  1. 把 PrimFunc 包成 IRModule。
  2. BindTarget(target_to_bind) 给没有 target 的函数加默认 func.attrs["target"]。
  3. 选 TIR pipeline：get_tir_pipeline(...) / get_default_tir_pipeline(target)，根据 target 做不同 lowering。
  4. split_host_device_mods(mod)：
     - 按 func.attrs["target"].kind 拆成 host_mod（"llvm"/"c"）和 device_mod_dict[target]（"cuda"/"metal" 等）。
  5. 对 host/device mod 跑收尾 pass。
  6. tir_to_runtime(host_mod, device_mod_dict, target_host)：
     - 每个 device_mod 调 codegen_build(device_mod, device_target)：
       - 通过 "target.build."+target.kind.name 找 C++ codegen：如 target.build.llvm、target.build.cuda；
       - 为每个设备生成一个子 runtime.Module。
     - host_mod 调 codegen_build(mhost_all, target_host)。
     - 把各个 device 模块 import_module 到 host 模块，返回一个统一的 tvm.runtime.Module。

- **"llvm" vs "c"**
  
  - "llvm"：直接生成机器码/JIT 模块，可在 TVM 内立即调用；
  - "c"：生成 CSourceModule（C 源码），需要外部编译后才能执行。

---

**三、Relax build → VMExecutable → VirtualMachine**

- **Python 层 relax.build（python/tvm/relax/vm_build.py）**
  
  1. 根据 target 跑 Relax pipeline（类型推断、PlanDevices、canonicalize 等）。
  2. _extract_attrs(mod)：从 mod.attrs 里拿：
     - "external_mods"：外部 runtime.Module 列表；
     - "const_name_to_constant"：常量名 → Constant 张量。
  3. builder = relax.ExecBuilder()：
     - C++ 里 ExecBuilderNode::Create() 会创建内部 exec_ = make_object<VMExecutable>()。
  4. _vmcodegen(builder, mod) → _ffi_api.VMCodeGen(builder, mod)：
     - C++ CodeGenVM::Run 遍历 IRModule 中的所有 relax.Function：
       - 对每个函数用 builder->EmitFunction/EmitCall/EmitRet 生成 VM 字节码；
       - 写入 VMExecutable 的 func_table / instr_offset / instr_data 等；
       - 同时从 IRModule 中移除这些 Relax.Function。
     - 返回剩下只含 TIR/ExternFunc 的 IRModule。
  5. _filter_tir(mod)：抽出 PrimFunc 组成 tir_mod。
  6. _vmlink(builder, target, tir_mod, tir_pipeline, ext_libs, params)：
     - 如前所述，对 TIR 部分调用 tvm.tir.build 得到设备代码模块 lib；
     - 将 lib 和 ext_libs 通过 LinkModules import 进 VMExecutable；
     - 返回一个 runtime.Module，实际节点是 VMExecutable；
     - Python 包一层 relax.VMExecutable(Executable)，里面的 self.mod 就是这个 Module。

- **VMLink C++ 实现（codegen_vm.cc）**
  
  - ObjectPtr<VMExecutable> executable = builder->Get();
    - Get() 对字节码做 Formalize + CheckExecutable，返回内部的 VMExecutable。
  - LinkModules(executable, params, lib, ext_libs)：
    - lidar constants / ext_libs，为需要常量的情况构造 ConstLoaderModule；
    - exec->ImportModule(const_loader_mod) 或 exec->ImportModule(lib) 等，把设备模块/外部模块挂进 VMExecutable 里。
  - return ffi::Module(executable);：
    - 把 ObjectPtr<VMExecutable> 包成通用的 ffi::Module。

---

**四、tvm.runtime.Executable 与 VMExecutable 的关系**

- relax.VMExecutable（Python）只是：
  
  `class VMExecutable(Executable): def __init__(self, mod: tvm.runtime.Module): super().__init__(mod)`

- Executable 只是包装一个 tvm.runtime.Module：
  
  `class Executable: def __init__(self, mod: Module): self.mod = mod # 这里的 mod 底层就是 C++ VMExecutable 节点`

- Executable.jit()：
  
  - 扫描 self.mod 的 import 树，找不能直接运行的子模块（kind == "c"/"static_library" 等）；
  - 如果没有这些“not runnable”模块，就直接 return self.mod；
  - 对 Relax VMExecutable 来说，self.mod 本身就是 runnable 的（kind="relax.VMExecutable"），所以 jit() 不会创建新的 C++ 对象，只是把内部 Module 交出。

---

**五、relax.VirtualMachine：Executable/Module → VM 实例**

python/tvm/runtime/vm.py：

`if not isinstance(rt_mod, tvm.runtime.Module): if isinstance(rt_mod, tvm.runtime.Executable):  rt_mod = rt_mod.jit() else: raise ValueError(...) load_exec = "vm_profiler_load_executable" if profile else "vm_load_executable" self.module = rt_mod[load_exec]()`

- 如果传进来的是 Executable（比如 relax.VMExecutable），就先 jit()：
  
  - 返回的是 rt_mod.mod，即底层的 tvm.runtime.Module，节点类型是 VMExecutable，kind="relax.VMExecutable"。

- rt_mod[load_exec]：
  
  - 用 tvm_ffi.Module.__getitem__ → _ffi_api.ModuleGetFunction(rt_mod, "vm_load_executable")；
  - C++ Module::GetFunction("vm_load_executable") 在 VMExecutable 的 vtable 里找到 VMExecutable::VMLoadExecutable，封成 PackedFunc。

- rt_mod[load_exec]()：
  
  - 调用 VMLoadExecutable()：
    
    `ffi::Module VMExecutable::VMLoadExecutable() const {  ObjectPtr<VirtualMachine> vm = VirtualMachine::Create();  vm->LoadExecutable(GetObjectPtr<VMExecutable>(const_cast<VMExecutable*>(this))); return ffi::Module(vm); }`
  
  - 创建新的 VirtualMachine C++ 对象（节点类型改变为 VirtualMachine），返回新的 runtime.Module；
  
  - Python 把这个 Module 存在 self.module，后续 self.module["invoke_stateful"] 等接口就是在这个 VirtualMachine 节点上。

---

**六、tvm-ffi FFI：从 C++ 到 Python 的桥**

- C++ 反射注册：
  
  `refl::GlobalDef()  .def_method("ffi.ModuleGetFunction",  [](const Module& mod, const String& name, bool query_imports) { return mod->GetFunction(name, query_imports);  });`

- Python _ffi_api.ModuleGetFunction 调用 core._get_global_func("ffi.ModuleGetFunction", ...)：
  
  - _get_global_func 实现在 Cython core.pyx 里，调用 TVMFFIFunctionGetGlobal 找 C++ 全局函数；
  - 把参数/返回值封装成 TVMFFIAny，交给 C++ Lambda 执行。

- 返回值转换由 make_ret / make_ret_object 负责：
  
  - 基本类型（int/float/str 等）走专门分支；
  
  - 所有对象（Module/Tensor/Function/VMExecutable/VirtualMachine 等）走：
    
    `elif type_index >= kTVMFFIStaticObjectBegin:  return make_ret_object(result)`
  
  - make_ret_object：
    
    - 用 type_index 在 TYPE_INDEX_TO_CLS 查 Python 类 cls（比如 tvm_ffi.Module）；
    - 先构造一个基础 Object，把 C 端句柄 result.v_obj 写入 chandle；
    - 再调用 cls.__from_tvm_ffi_object__(cls, obj) 生成最终包装类实例。

- core.Object（Cython）：
  
  `cdef class Object:  cdef void* chandle  def __cinit__(self):  self.chandle = NULL  def __dealloc__(self):  if self.chandle != NULL:  TVMFFIObjectDecRef(self.chandle)`

- tvm_ffi.Module：
  
  `@register_object("ffi.Module") class Module(core.Object): def __getitem__(self, name): return self.get_function(name) def get_function(self, name, query_imports=False): return _ffi_api.ModuleGetFunction(self, name, query_imports) def __call__(self, *args): return self.main(*args)`

这套机制让你在 Python 调用 _ffi_api.VMLink 得到的就是一个 Module 实例，内部 chandle 指向的是 VMExecutable（或后来的 VirtualMachine）C++ 对象。

---

**七、this / kind / 类型不变性的关键点**

- C++ 对象类型是在 make_object<SomeClass>() 时确定的（type_index / type_key 写在对象头部）：
  - make_object<VMExecutable>() → kind="relax.VMExecutable"；
  - 后续无论 Python 怎么用 Executable / Module 包裹，这个 C++ 对象本身的类型不会变。
- Executable.jit() 对 Relax VMExecutable 只返回内部的 Module，不 new 新对象，所以：
  - rt_mod 从 Python 的 Executable 变成 Module；
  - 底层 C++ 对象仍然是原来的 VMExecutable，kind 不变。
- 调 rt_mod["vm_load_executable"]() 时：
  - GetFunction 的 this 是 VMExecutable*；
  - VMLoadExecutable() 里创建了新的 VirtualMachine 对象，返回新的 Module（底层节点类型变成 VirtualMachine），后续 VM 执行都在这个新对象上完成。

---

整体串起来就是：

- 前端：TVMScript → doc.AST → Parser + IRBuilder → IRModule/PrimFunc/Function；
- 中间：Target + tir.build → host/device 拆分 + 各设备 codegen；
- Relax：relax.build → VMCodeGen 生成 VM 字节码（VMExecutable）+ TIR build → VMLink 把二者及常量绑定；
- 运行时：Python Executable 只是壳，VirtualMachine(ex, dev) 通过 jit+vm_load_executable 真正构造 C++ VM 实例；
- FFI：所有 Python ↔ C++ 的函数调用、返回值包装都通过 tvm‑ffi 的 TVMFFIAny + Cython（core.pyx / object.pxi / function.pxi）自动完成，Python 侧模块类只是拿着 chandle 指向 C++ ModuleObj（VMExecutable/VirtualMachine 等）。
