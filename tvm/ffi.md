# Object

---

Object (基类)
    ↓
ObjectPtr<T> (智能指针，管理生命周期)
    ↓
ObjectRef (用户接口，类似 shared_ptr)

用于存储元信息（key，index等），所有后续类型都是Object的子类，用于存放数据

需要声明类型信息

```
TVM_FFI_DECLARE_OBJECT_INFO_FINAL(TypeKey, TypeName, Parent_TypeName);
```

ObjectPtr 是 TVM runtime 的“所有权指针”
负责 ref count + 析构 Object

之后ObjectRef 类用于定义接口

同样需要声明

```
TVM_FFI_DEFINE_OBJECT_REF_METHODS_NULLABLE(TypeName, ParentType, ObjectName)
```

# Function

---

## FunctionObj

FunctionObj是Object的子类，是存储函数的地方

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
