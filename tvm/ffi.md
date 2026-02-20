# Object

---

Object (基类)
    ↓
ObjectPtr<T> (智能指针，管理生命周期)
    ↓
ObjectRef (用户接口，类似 shared_ptr)

用于存储元信息（key，index等），所有后续类型都是Object的子类，用于存放数据

需要声明类型信息

```c++
TVM_FFI_DECLARE_OBJECT_INFO_FINAL(TypeKey, TypeName, Parent_TypeName);
```

ObjectPtr 是 TVM runtime 的“所有权指针”
负责 ref count + 析构 Object

之后ObjectRef 类用于定义接口

同样需要声明

```c++
TVM_FFI_DEFINE_OBJECT_REF_METHODS_NULLABLE(TypeName, ParentType, ObjectName)
```

# Function

---

## FunctionObj

FunctionObj是Object的子类，是存储函数指针的地方

有cpp_call（异常处理：抛出c++异常） & safe_call（异常处理：返回错误码，必须实现）

## FunctionObj实现类

给FuncObj提供safe call或cpp call的具体实现，并存储函数

## Function类

Functino是ObjectRef的子类，从各种方式实现函数签名的存储

创建方式：

*1.从PackedFunc（需通过实现类自己实现safe call）签名创建*

接受两种函数签名： void(const AnyView*, int32_t, Any*)和void(PackedArgs args, Any*)，接受函数之后实现对应的FunctionObject

*2.从类型化函数转化为packedfunc创建*

*3.从c风格创建（传入safe call）*

*4.从全局注册表获取*

调用方式：

*1.用（）直接调用*

*2.直接调用CallPacked*

也可以使用TypedFunction来进行创建与调用

模版类（接受一个函数及参数类型），里面存储一个Function，可直接转换成Function

导出为DLL符号，通过__tvm_ffi_name调用函数

```
// 导出为 DLL 符号 "__tvm_ffi_AddOne"
TVM_FFI_DLL_EXPORT_TYPED_FUNC(AddOne, AddOne_);
```

用户调用: func(10, 20)
    ↓
Function::operator()()
    ↓
填充参数到 AnyView 数组
    ↓
FunctionObj::CallPacked()
    ↓
检查 cpp_call 是否存在
    ↓
有 cpp_call: 直接调用（快速路径）
    ↓
无 cpp_call: 通过 safe_call（慢速路径，有异常处理）
    ↓
执行实际函数代码
    ↓
返回结果（Any）

# Module

---

Object (基类)
    ↓
ModuleObj (抽象基类，定义接口)
    ↓
LibraryModuleObj (具体实现：动态库模块)
    ↓
Module (ObjectRef，用户接口)

## ModuleObj

函数获取、序列化、模块导入并存储导入模块与导入函数的功能

***模块序列化***

将内存中的模块对象转换为字节流

***反序列化***

从字节流恢复为模块对象

## Module

用户接口

从文件中加载module

## 模块导入机制

* 导入模块到一个ModuleObj的列表里

* 从导入中能查找函数

* ModuleObj内有导入函数缓存，能从中找到对应函数

## 模块加载机制

* 从文件中加载

* 库二进制处理
  
  * 一种反序列化
  
  * 读取嵌入在动态库文件中的二进制数据
  
  * 恢复多个模块
  
  * 并重建模块之间的导入关系（将子模块添加到父模块的imports_中）

***每个模块都有对应的注册的加载器函数（全局函数）和getfunction机制；在创建模块的时候，会把收集到的函数汇总，并存储到模块中，保存时需要序列化模块，使用时在反序列化回来***

# Python_helper

--- 

*用于Python FFI的CC的辅助函数，优化Python到C得调用性能*

## TVMFFIPyCallManager（调用管理器）

```
thread_local：cc关键字，表示该变量在每个线程都有一份副本，互不干扰
```

### FuncCall

将python参数转化为c参数

设置设备流上下文

调用C函数并处理返回值

### ConstructorCall

类似于FuncCall，但用于构造对象

# python侧主要的三个函数

`def register_object(type_key: str | None = None) -> Callable[[_T], _T]:`

返回一个函数（该函数接收一个类，并返回一个类）

首先通过c api根据type key得到注册好的type index

之后根据index得到typeinfo，得到key（python str）；用 C++ 的 type_index 对应的 TVMFFITypeInfo，在 Python 里构造一个 TypeInfo，并把传入的 Python 类 type_cls以及信息填进去；把「type_index / type_key / type_cls ↔ TypeInfo」写进四个全局结构

之后给 cls 挂 property（field.getter/setter）和 方法（method.as_callable）；按需设置 init_

最后类上保存 TypeInfo，方便后续按类查类型信息

` def register_global_func(func_name: str | Callable[..., Any], f: Callable[..., Any] | None = None, override: bool = False) -> Any` 

首先将传入的pyfunc转化成cc中的Function

之后将得到的Function的指针传递给python

在python侧构造Function，并进行chandle赋值

最后再注册进全局函数注册表内

` def _get_global_func(name: str, allow_missing: bool):`

根据全局注册表获得chandle，构建Function并返回

## ObjectDef

- 实现：TypeTable（在 object.cc 里），内部是 type_table_[type_index]，用 type_index（整数） 做下标，每个槽位是一个 Entry（继承自 TVMFFITypeInfo，再加上 fields/methods/metadata 等）。

- 注册：ObjectDef<MyClass>() 构造时先取 type_index_，之后的 def_ro/def_rw/def/def_static 等都会用这个 type_index_ 调 TVMFFITypeRegisterField(type_index_, ...)、TVMFFITypeRegisterMethod(type_index_, ...)、TVMFFITypeRegisterMetadata(type_index_, ...)，把字段、方法、元数据挂到该 type 的 Entry 上。

- 查找：拿到某个对象的 type_index（例如 obj->header_.type_index）后，用 TVMFFIGetTypeInfo(type_index) → TypeTable::Global()->GetTypeEntry(type_index) 得到该类型的 TVMFFITypeInfo*（实际是 Entry），里面就有 fields、methods、metadata。

## GlobalDef

- 实现：GlobalFunctionTable（在 function.cc 里），内部是 Map<String, Any> table_，用函数名（字符串）做 key。

- 注册：GlobalDef().def("name", func) 等会走到 TVMFFIFunctionSetGlobal / TVMFFIFunctionSetGlobalFromMethodInfo，把 (name, Function) 放进这张表。

- 查找：Python/C 调 TVMFFIFunctionGetGlobal(name) 时，用名字在表里查，得到对应的 Function（即 TVMFFIMethodInfo 风格的 Entry）。
