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

给FuncObj提供具体实现，并存储

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
