# 为啥ffi大量用Cython而不是Ctypes

1. 当有指针成员的时候，python的GC会回收无用的指针，导致c访问到野指针

2. python和c都不知道何时malloc或者free（cython可实现RAII，对象不使用时自动销毁资源）

3. 和C 端不一致时，结构体中的对齐要求必须手动指定

4. 不支持c++类

5. GIL控制粗糙（借助GIL控制可以实现多线程执行非python代码）

# ctypes

## features

纯python FFI

不需要编译

面向C API

## 语法

### 加载动态库

```python
from ctypes import *

lib = CDLL("./libfoo.so")
```

### 函数声明

```python
lib.add.argtypes = [c_int, c_int]
lib.add.restype  = c_int

res = lib.add(2, 3)
```

### 指针

```python
p = POINTER(c_int) # 仅声明一个指针
#####################
x = c_int(10)
px = pointer(x)
```

### 结构体

```python
class Point(Structure):
    _fields_ = [
        ("x", c_float),
        ("y", c_float),
    ]
```

### 数组

```python
buf = (c_float * 128)()
```

## 缺点

性能差

生命周期难以管理（python的GC（垃圾回收）回收没用的python对象，但是c仍然使用导致指针悬空）

无c++中类和模版的概念

GIL（全局锁）控制粗糙：只能在调用c函数的那段时间释放GIL

# Cython

## 类型申明

cdef：只在c层存在，python不能直接访问

cpdef：c和python都能访问

```cython
def f(x):
    return x + 1
cdef int add_int(int a, int b):
    return a + b
cpdef int add(int a, int b):
    return a + b
```

模块内部，随便调用

### c变量，指针，数组

```cython
cdef int a
```

### 引用外部函数、宏、常量

```cython
cdef extern from "math.h":
    double sqrt(double x)

def py_sqrt(double x):
    return sqrt(x)

cdef extern from "limits.h":
    int INT_MAX
```

### Cython扩展类

```cython
cdef class BufferOwner:
    cdef char* ptr
    cdef Py_ssize_t n

    def __cinit__(self, Py_ssize_t n):
        self.n = n
        self.ptr = <char*> malloc(n)
        if self.ptr == NULL:
            raise MemoryError()

    def __dealloc__(self):
        if self.ptr != NULL:
            free(self.ptr)

    def address(self):
        return <size_t> self.ptr
```

### GIL控制

```cython
cdef void scale(double* a, Py_ssize_t n) nogil:
    cdef Py_ssize_t i
    for i in range(n):
        a[i] *= 2.0

def py_scale(double[:] a):
    with nogil:
        scale(&a[0], a.shape[0])
```

### setup

```python
from setuptools import setup, Extension
from Cython.Build import cythonize
import numpy as np

extensions = [
    Extension(
        name="mypkg.core",                 # Python import 名
        sources=["mypkg/core.pyx"],        # 对应 .pyx
        include_dirs=[np.get_include()],   # 用到 numpy/memoryview 常加
        language="c++",                    # 如果不用 C++ 就写 "c"
        extra_compile_args=["-O3"],
    ),
    Extension(
        name="mypkg.ops",
        sources=["mypkg/ops.pyx"],
        include_dirs=[np.get_include()],
        language="c++",
        extra_compile_args=["-O3"],
    ),
]

setup(
    name="mypkg",
    version="0.1.0",
    packages=["mypkg"],
    ext_modules=cythonize(
        extensions,
        compiler_directives={
            "language_level": "3",
            "boundscheck": False,
            "wraparound": False,
            "cdivision": True,
            "nonecheck": False,
        },
        annotate=False,  # True 会生成 HTML 报告（调性能很有用）
    ),
)
```
